# Net Worth: How It Works (Consolidated Documentation)

**Version:** 1.0 (Consolidating findings from code analysis and historical docs as of 2025-05-03)

**Purpose:** This document serves as the single source of truth detailing the end-to-end process for calculating, storing, updating, and using the Player Net Worth metric in the Farming Game. It incorporates learnings from previous debugging efforts and architectural changes.

## 0. Historical Context: The Road to Stability

The current net worth system evolved from addressing significant bugs where net worth values were inaccurately calculated (often matching only cash, ignoring assets/debt). A multi-phase fix plan (documented historically in `docs/networthsucks.MD`) was implemented, resulting in the current architecture focused on validation, hooks, and centralized recalculation. Key steps included:
-   Implementing universal transaction wrappers and validators (`createProtectedTransaction`, `validatePlayerUpdateData`) to prevent direct, incorrect `netWorth` updates.
-   Creating the `NetWorthHookService` to centralize recalculation triggers.
-   Adding database-level protections (SQL views/triggers - see Section 8).
-   Enhancing logging and testing significantly.

Understanding this history helps explain the emphasis on protection, hooks, and centralized calculation.

## 1. Definition and Formula

Net Worth is calculated using the following formula:

```
Net Worth = Cash + Total Asset Value + Total Ridge Value - Debt
```

Where:
-   `Cash`: The player's current cash balance.
-   `Total Asset Value`: The sum of the calculated values of all non-ridge assets owned by the player (e.g., Tractors, Harvesters). Calculated by `AssetService.getTotalAssetValue`.
-   `Total Ridge Value`: The sum of the calculated values of all ridge assets owned by the player. Calculated by `AssetService.getTotalRidgeValue`.
-   `Debt`: The player's current debt amount.

## 2. Storage

Net Worth and its key components are stored directly on the `Player` model in the PostgreSQL database, managed via `packages/backend/prisma/schema.prisma`:

```prisma
// packages/backend/prisma/schema.prisma
model Player {
  // ... other fields
  cash              Int      @default(0)
  debt              Int      @default(0)

  // Scoreboard / Calculated values (denormalized for easier access)
  totalAssetValue   Int      @default(0) // Value of owned non-ridge assets
  totalRidgeValue   Int      @default(0) // Value of owned ridges
  netWorth          Int      @default(0) // cash + totalAssetValue + totalRidgeValue - debt

  // ... other fields related to assets like farm_cow_count
  assets            Asset[]  // Relation to non-ridge assets
  ridges            Ridge[]  // Relation to ridge assets

  @@map("players")
}

// Audit Table (Concept from docs/net_worth_system.md, verify exact schema if needed)
// model NetWorthAudit { ... }
```

-   `netWorth`: Stores the calculated net worth.
-   `totalAssetValue`: Stores the calculated value of non-ridge assets.
-   `totalRidgeValue`: Stores the calculated value of ridge assets.
-   `cash` and `debt` are also stored directly on the player model.
-   An audit table (likely `net_worth_audit` or similar) logs historical changes (see Section 9).

Storing these calculated values is a form of denormalization to simplify retrieval.

## 3. Central Calculation: `GameService.recalculateNetWorthForAllPlayers`

The primary mechanism for ensuring `netWorth` is accurate is the `GameService.recalculateNetWorthForAllPlayers` method located in `packages/backend/src/services/game.service.ts`.

**Steps:**

1.  **Fetch Player Data:** Retrieves player data, including associated assets and ridges.
2.  **Calculate Asset/Ridge Values:**
    *   Calls `AssetService.getTotalAssetValue(player.assets)` to get the value of non-ridge assets.
    *   Calls `AssetService.getTotalRidgeValue(player.ridges)` to get the value of owned ridges.
    *   *(Note: The asset value calculation sums the value of each asset, determined by multiplying the asset's `quantity` (e.g., acres) by the base value per unit (`BASE_ASSET_VALUES`) for its `assetType`).*
3.  **Update Total Values:** Updates the `totalAssetValue` and `totalRidgeValue` fields for the player in the database within the current transaction (`tx` if provided).
4.  **Calculate Net Worth (DB Level):** Executes a raw SQL query to update the `netWorth` field *directly in the database*. This critical step ensures atomicity and uses the most current values present in the database row for the calculation:
    ```sql
    UPDATE "players"
    SET "netWorth" = "cash" + "totalAssetValue" + "totalRidgeValue" - "debt"
    WHERE "id" = $1
    ```
5.  **Verification (Optional but Recommended):** After the raw update, the code often refetches the player data to verify that the `netWorth` stored in the database matches the expected value based on the components (`cash`, `totalAssetValue`, `totalRidgeValue`, `debt`). Logs errors if discrepancies are found.

**Important Note:** A private method `GameService.calculateNetWorth` exists but seems primarily for ad-hoc checks (e.g., during `exerciseOption`) and does *not* perform the database update itself. The authoritative update path is `recalculateNetWorthForAllPlayers`.

### 3.1 Initial Net Worth Calculation

The player's initial net worth is established through the following sequence:

1.  **`GameService.createGame`:** When a new game is created, this method:
    *   Creates the `Player` record with starting `cash` and `debt`.
    *   Immediately creates the initial `PlayerAsset` records (e.g., 10 acres of `Hay`, 10 acres of `Grain`).
2.  **First Recalculation:** The *first* call to `GameService.recalculateNetWorthForAllPlayers` for that player (typically triggered by the `NetWorthHookService` when the player joins the game via the `GameGateway`) calculates the initial `totalAssetValue`, `totalRidgeValue`, and finally the `netWorth` based on the cash, debt, and *newly created* starting assets.

This ensures the net worth reflects all starting components from the beginning.

## 4. Hooks: Triggering Recalculation (`NetWorthHookService`)

The `NetWorthHookService` (`packages/backend/src/services/net-worth-hook.service.ts`) centralizes the triggering of recalculations. It provides specific hook methods called by other services whenever a component of net worth changes:

-   `afterCashOperation(playerId, tx)`: Called after changes to player `cash`.
-   `afterAssetChange(playerId, tx)`: Called after changes to player non-ridge `assets` (affecting `totalAssetValue`).
-   `afterDebtChange(playerId, tx)`: Called after changes to player `debt`.
-   `afterRidgeChange(playerId, tx)`: Called after changes to player `ridges` (affecting `totalRidgeValue`).

These hooks typically wrap the call to `GameService.recalculateNetWorthForAllPlayers` for the specified player within the provided transaction (`tx`).

## 5. Integration Points: When are Hooks Called?

The hook methods (and thus `recalculateNetWorthForAllPlayers`) are invoked by various services:

-   **`CardEffectService`:** After applying card effects modifying cash, assets, debt, or ridges.
-   **`TileEffectService`:** After applying tile effects modifying cash, assets, debt, or ridges.
-   **`HarvestService`:** After harvest operations impacting cash or asset values.
-   **`GameService`:** After specific actions like `exerciseOption`.
-   **`GameLoopService`:** After player and AI turns are processed, often calling `recalculateNetWorthForAllPlayers` directly for all players as a final safeguard.
-   **Potentially on Game Load (`GameGateway`):** To ensure clients receive fresh data.

**(See Section 10 for known complexities regarding hook execution order in nested calls)**

## 6. Protection Against Direct Modification (`player-update.validator.ts`)

Directly setting the `netWorth` field is explicitly forbidden to maintain the integrity of the calculation.

-   **`player-update.validator.ts`:**
    *   The `validatePlayerUpdateData` function intercepts any player update data.
    *   If it detects an attempt to set the `netWorth` field directly, it **removes** the field from the update data and logs a **critical error**.
    *   The `createProtectedTransaction` function wraps database updates involving player data. After the update, it includes validation checks to ensure the `netWorth` stored in the database correctly corresponds to its constituent parts (`cash`, `totalAssetValue`, `totalRidgeValue`, `debt`). It logs errors if mismatches occur.

## 7. Frontend Usage

The frontend primarily consumes the `netWorth` value provided by the backend's game state updates.

-   **`Scoreboard.tsx`:** Displays the `player.netWorth` value for each player.
-   **`PlayerHand.tsx`:** Uses `player.netWorth` within the `checkOTBAffordability` function to calculate the player's maximum borrowing capacity (`maxDebt = player.netWorth * 0.5`), which determines if they can afford OTB cards that require borrowing.

## 8. Database-Level Protection & Maintenance

As part of the stabilization effort, database-level mechanisms were introduced or planned:
-   **SQL Functions/Views:** (e.g., `get_player_net_worth`, `get_players_with_net_worth_discrepancies` - check `docs/db` or schema for exact names/implementation). These provide a canonical way to calculate net worth directly in SQL.
-   **Triggers (Potentially):** Database triggers might be in place to automatically enforce correctness or log changes (verify schema/migration files).
-   **Maintenance Job:** `NetWorthHookService.runNetWorthMaintenanceJob()` exists to periodically scan for and potentially correct discrepancies between calculated and stored net worth values.
-   **Validation Function:** `NetWorthHookService.validatePlayerNetWorth` can be used to check if a player's stored net worth matches the calculated value based on its components.

## 9. Auditing

A dedicated audit table (e.g., `net_worth_audit` - verify exact name in schema) logs changes to net worth, providing a history for debugging. `NetWorthHookService.createNetWorthAuditRecord` is used to add entries.

## 10. Known Complexities and Future Considerations

-   **Nested Hook Registration:** As detailed in `docs/networthsuckspart2.md`, a potential complexity exists when services call each other (e.g., `CardEffectService` calling `TileEffectService.handlePayment`). This can lead to scenarios where net worth hooks might be registered multiple times or in an unexpected order within a single transaction. While workarounds might exist (like targeted manual hook registration for specific effects like `PayInterest`), a more robust solution involving a shared `HookRegistrationContext` was proposed but may not be fully implemented. This interaction remains a point of potential fragility requiring careful handling in transaction logic.
-   **Database Triggers/Functions Status:** While mentioned in planning documents, the exact implementation status and effectiveness of database-level triggers and functions for enforcement should be verified if debugging deep discrepancies.
-   **Drift Correction:** The concept of a fully automated "drift correction" system mentioned in planning documents might not be fully realized beyond the maintenance job.

## 11. Best Practices & Debugging

-   **Transactions:** Always wrap operations modifying net worth components within a Prisma transaction (`prisma.$transaction(async (tx) => { ... })`) and pass the `tx` client to hook methods.
-   **Use Hooks:** Rely on the `NetWorthHookService` methods after modifying cash, assets, debt, or ridges. Do *not* call `recalculateNetWorthForAllPlayers` directly unless necessary (like end-of-turn consolidation).
-   **Avoid Direct `netWorth` Setting:** Never attempt to set `netWorth` directly in update data; rely on the recalculation triggered by hooks.
-   **Debugging:**
    -   Check backend logs for "[NET_WORTH_CRITICAL]", "[NET_WORTH_CORRECTION]", and specific hook service debug messages.
    -   Verify player `cash`, `debt`, `totalAssetValue`, `totalRidgeValue` in the database using Prisma Studio or a DB client.
    -   Inspect the `net_worth_audit` table history (if implemented and populated).
    -   Use `NetWorthHookService.validatePlayerNetWorth(playerId)` for ad-hoc checks during debugging sessions.
    -   Consider the nested hook complexity (Section 10) if issues arise during complex actions involving multiple service interactions.

## 12. Win Conditions & Net Worth Thresholds

-   The configuration in `packages/backend/src/config/game.config.ts` defines both `winningNetWorth` (currently `$250,000`) and `minTurnsForNetWorthWin` (currently `40`).
-   `GameLoopService.checkGameEndConditions` reads these values and requires that at least one active player has completed `minTurnsForNetWorthWin` turns **and** has a `netWorth` ≥ `winningNetWorth` before marking the game `Completed`.
-   This safeguard prevents high-variance net worth spikes early in a session from short-circuiting long-running games (e.g., stress tests beyond 40 turns).
-   When diagnosing premature game completion, confirm the player turn counts (`players.turnsTaken`) alongside net worth to see if the configuration thresholds were satisfied.
