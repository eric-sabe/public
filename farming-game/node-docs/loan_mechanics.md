# Loan Mechanics: Board Game Rules vs. Backend Implementation

This document details the intended loan mechanics per The Farming Game® rules, how loan/forced loan effects are currently implemented in the backend, and what needs to change to align the digital version with the official rules. It references the effect processing architecture described in [game-effects.md](game-effects.md).

## 1. Official Board Game Rule: Forced Loans

According to the original game instructions:

> "If a player gets in a situation that requires more cash than he has on hand, he may borrow (providing there is room left on the credit line) from the bank in **$5,000 increments**, paying a **20% refinance penalty fee** for not properly managing his finances."

**Example:**
- Need $3,000: Borrow $5,000. Pay $1,000 (20% of $5k) penalty. Apply $4,000 to the expense. Net $1,000 cash gain.
- Need $7,000: Borrow $10,000. Pay $2,000 (20% of $10k) penalty. Apply $8,000 to the expense. Net $1,000 cash gain.

## 2. How Loan Effects Are Triggered in the Game

Loan/forced loan effects are typically triggered by:
- **Board Tile Effects:** Landing on a tile with an `effectType` such as `PayCash`, `Expense`, or `ExpensePerAsset` (see [game-effects.md](game-effects.md), section 2.1).
- **Card Effects:** Drawing or playing a card with an effect that requires a payment (e.g., `PayCashIfAsset`, `OperatingExpense` cards).

When a player cannot pay the required amount, the backend must process a forced loan according to the rules above.

## 3. Current Backend Implementation

- **Relevant Codepath:**
  - `TileEffectService.handlePayment` (in `packages/backend/src/services/tile-effect.service.ts`)
  - Invoked by tile or card effects that require a payment (see [game-effects.md](game-effects.md), section 3.1 and 3.2).
- **Current Logic:**
  1. Calculates the **exact shortfall**: `requiredLoan = amountDue - player.cash`.
  2. Checks if `player.debt + requiredLoan` exceeds `MAX_DEBT`.
  3. If within the limit:
     - Increments `player.debt` by the **exact requiredLoan**.
     - Reduces `player.cash` to $0.
     - **No rounding up** to $5,000 increments.
     - **No 20% penalty** is applied.

## 4. Discrepancy Summary

The backend currently deviates from the official rules in two key ways:
- **Loan Increment:** Loans are taken for the exact shortfall, not in $5,000 increments.
- **Refinance Penalty:** No 20% penalty is applied to the borrowed amount.

This makes financial difficulty less punitive than intended by the board game.

## 5. Required Backend Changes (TileEffectService.handlePayment)

To align with the official rules, update the forced loan logic as follows:

1. **Calculate Loan Amount:**
   - `shortfall = amountDue - player.cash`
   - `loanAmount = Math.ceil(shortfall / 5000) * 5000`
   - If `player.debt + loanAmount` exceeds `MAX_DEBT`, proceed to bankruptcy logic.
2. **Calculate Penalty:**
   - `penalty = loanAmount * 0.20`
3. **Apply Loan and Payment:**
   - Player receives `loanAmount - penalty` in cash.
   - Pays the required amount (`amountDue`).
   - Final cash: `player.cash = player.cash + loanAmount - penalty - amountDue`
   - Debt increases by `loanAmount`.
4. **Update Player State:**
   - Use a single Prisma update: `debt: { increment: loanAmount }`, `cash: { increment: loanAmount - penalty - shortfall }`
5. **Logging:**
   - Log the loan amount, penalty, and resulting cash/debt changes for transparency.

**See [game-effects.md](game-effects.md) for the overall effect processing flow and how payment effects are triggered and handled.**

## 6. References
- [game-effects.md](game-effects.md): Effect processing architecture and codepaths
- `packages/backend/src/services/tile-effect.service.ts`: Payment and loan logic
- `GameEffectType.PayCash`, `GameEffectType.Expense`, etc.: Effect triggers
