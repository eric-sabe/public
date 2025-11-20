# Game Effects: Definition, Triggering, and Processing in Farming Game

This document provides a comprehensive overview of how game effects (as represented by `GameEffectType`) are defined, triggered, and processed in the Farming Game application. It is intended as a technical reference for developers seeking to understand the architecture and flow of effect handling, from board tiles and cards to backend processing and frontend updates.

## 1. Effect Definition: Where and How Are Effects Specified?

### 1.1. GameEffectType Enum
- **Location:** `packages/shared-types/src/game-types.ts`
- **Purpose:** Enumerates all possible effect types that can occur in the game, such as `GainCash`, `PayCash`, `DrawCard`, `GoToTile`, `ExpensePerAsset`, etc.
- **Usage:** Used by both backend and frontend to type-check and interpret effect logic.

### 1.2. Board Tiles
- **Location:** Board tile definitions are found in `shared-types/src/board.ts` and are instantiated in the backend as part of game setup.
- **Effect Assignment:** Each `BoardTile` may have an `effectType` (from `GameEffectType`) and associated `effectDetails` (arbitrary data, often validated by Zod schemas).
- **Example:** Landing on a tile with `effectType: GainCash` and `effectDetails: { amount: 2000 }`.

### 1.3. Cards (Farmer's Fate, Option To Buy, Operating Expense, etc.)
- **Location:** Card templates are defined in the backend (`packages/backend/src/config/cards.catalog.ts`) and stored in the database (`CardDefinition` model).
- **Effect Assignment:** Each card has an `effectType` and `effectDetails`.
- **Example:** A Farmer's Fate card with `effectType: PayCashIfAsset`, `effectDetails: { assetType: 'Tractor', amount: 1000 }`.

### 1.4. Zod Schemas for Effect Details
- **Location:** `shared-types/src/game-types.ts`
- **Purpose:** Provide runtime validation for complex `effectDetails` objects, ensuring type safety and correct structure.

## 2. Effect Triggering: When and How Are Effects Activated?

### 2.1. Board Tile Effects
- **Trigger:** When a player lands on a board tile with an `effectType`, the effect is triggered immediately.
- **Codepath:**
  - `GameLoopService.handlePlayerTurn` (backend): Detects tile landing and calls `TileEffectService.applyTileEffect`.
  - `TileEffectService.applyTileEffect`: Reads the tile's `effectType` and `effectDetails`, then dispatches to the appropriate handler method.

### 2.2. Card Effects
- **Trigger:**
  - **Drawn Card:** Some cards (e.g., Farmer's Fate) are applied immediately upon draw; others (e.g., Option To Buy) are added to the player's hand for later use.
  - **Played Card:** When a player chooses to play a card from their hand (via UI), the effect is triggered.
- **Codepath:**
  - `GameLoopService.handleDrawCard` or `CardEffectService.applyCardEffect` (for immediate effects).
  - `GameGateway.handleApplyCardEffect` (for player-initiated plays): Receives the socket event, validates, and calls `GameService.applyCardEffect`.
  - `GameService.applyCardEffect`: Delegates to `CardEffectService.applyCardEffect`.

### 2.3. Special/Chained Effects
- Some effects (e.g., `DrawCard`, `GoToTile`) may trigger additional effects or actions, leading to chained processing within the same turn.

## 3. Effect Processing: Backend Flow and Main Services

### 3.1. TileEffectService
- **Location:** `packages/backend/src/services/tile-effect.service.ts`
- **Role:** Central handler for all board tile effects.
- **Pattern:** Contains a main dispatcher method (e.g., `applyTileEffect`) that switches on `effectType` and calls private handler methods for each effect.
- **Responsibilities:**
  - Adjust player state (cash, position, assets, etc.)
  - Handle movement (`GoToTile`), cash changes, harvest bonuses, etc.
  - Log actions and generate game log entries
  - Return updated state to the game loop

### 3.2. CardEffectService
- **Location:** `packages/backend/src/services/card-effect.service.ts`
- **Role:** Central handler for all card effects, including Farmer's Fate, OTB, OE, and special cards.
- **Pattern:** Contains a main dispatcher method (e.g., `applyCardEffect`) that switches on `effectType` and calls private handler methods for each effect.
- **Responsibilities:**
  - Apply immediate card effects (e.g., gain/pay cash, skip turn, disaster)
  - Handle option cards (add to hand, process when exercised)
  - Manage persistent/temporary effects (e.g., harvest multipliers)
  - Log actions and generate game log entries
  - Return updated state to the game loop

### 3.3. GameLoopService
- **Location:** `packages/backend/src/services/game-loop.service.ts`
- **Role:** Orchestrates the main turn flow, including movement, effect triggering, card draws, and harvests.
- **Responsibilities:**
  - Detects when effects should be triggered (tile landing, card draw, card play)
  - Calls `TileEffectService` and `CardEffectService` as needed
  - Handles chained effects and ensures correct order of operations
  - Integrates with `HarvestService` for harvest-related effects

### 3.4. GameGateway
- **Location:** `packages/backend/src/gateways/gateway-v2/game-gateway-v2.gateway.ts`
- **Role:** Handles all real-time socket events from the frontend, including player actions that may trigger effects.
- **Responsibilities:**
  - Receives events like `client:rollDice`, `client:applyCardEffect`, `client:exerciseOption`
  - Validates and routes actions to the appropriate service
  - Emits updated game state and logs to all clients

### 3.5. HarvestService
- **Location:** `packages/backend/src/services/harvest.service.ts`
- **Role:** Handles the calculation of harvest income and the application of Operating Expense (OE) card effects during harvest.
- **Responsibilities:**
  - Draws and applies OE cards (Expense, ExpensePerAsset, PayInterest, etc.)
  - Applies temporary harvest modifiers and bonuses
  - Returns income, expenses, and log messages to the game loop

## 4. Effect Application Flow: End-to-End Example

1. **Player lands on a tile with an effect:**
   - `GameLoopService.handlePlayerTurn` detects the tile's `effectType` and calls `TileEffectService.applyTileEffect`.
   - `TileEffectService` processes the effect, updates player/game state, and logs the action.
   - If the effect is `DrawCard`, the player draws a card and may trigger further effects (see below).

2. **Player draws a card with an immediate effect:**
   - `CardEffectService.applyCardEffect` is called (either directly or via the game loop).
   - The effect is processed, and the game state is updated accordingly.
   - If the card is an option (e.g., OTB), it is added to the player's hand for later use.

3. **Player plays a card from hand (via UI):**
   - Frontend emits `client:applyCardEffect` or `client:exerciseOption` via socket.
   - `GameGateway` receives the event, validates, and calls `GameService.applyCardEffect` or `GameService.exerciseOptionToBuyAsset`.
   - `CardEffectService` processes the effect, updates state, and logs the action.

4. **State Update and Notification:**
   - After any effect is processed, the backend emits `server:gameStateUpdate` and `server:gameLog` to all clients.
   - The frontend updates its state and UI accordingly.

## 5. BoardTile Effects vs. Card Effects: Relationship and Differences

- **BoardTile Effects:** Triggered automatically upon landing; processed by `TileEffectService`.
- **Card Effects:** May be immediate (applied on draw) or deferred (added to hand, played later); processed by `CardEffectService`.
- **Chained Effects:** Some effects (e.g., `DrawCard`, `GoToTile`) can trigger additional effects, handled recursively by the game loop and effect services.

## 6. Relevant Files and Codepaths

- `shared-types/src/game-types.ts`: Effect type definitions, enums, and Zod schemas
- `backend/src/config/cards.catalog.ts`: Card templates and effect assignments
- `backend/src/services/tile-effect.service.ts`: Board tile effect processing
- `backend/src/services/card-effect.service.ts`: Card effect processing
- `backend/src/services/game-loop.service.ts`: Turn orchestration and effect triggering
- `backend/src/services/harvest.service.ts`: Harvest and OE card effect processing
- `backend/src/gateways/gateway-v2/game-gateway-v2.gateway.ts`: Socket event handling

## 7. Socket Events and Frontend Integration

- **Triggering Effects:**
  - Player actions (roll dice, play card, exercise option) are sent as socket events from the frontend.
  - The backend processes these actions, applies effects, and emits updated state.
- **Receiving Updates:**
  - The frontend listens for `server:gameStateUpdate`, `server:gameLog`, and other events to update the UI.
  - Immediate effects (e.g., cash changes, card draws) are reflected in the UI as soon as the backend processes them.

## 8. Summary

Game effects in Farming Game are defined declaratively (in board tiles and cards), triggered by player actions or game events, and processed by dedicated backend services. The architecture ensures a clear separation of concerns, robust type safety, and real-time synchronization between backend logic and frontend UI. 