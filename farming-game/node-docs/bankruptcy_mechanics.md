# Bankruptcy Mechanics: Digital Implementation vs. Board Game Rules

This document explains how bankruptcy and bankruptcy auctions are defined, triggered, and processed in the digital Farming Game. It references the original board game rules and details the current backend and frontend codepaths, highlighting both implemented and intended behaviors.

---

## 1. Official Board Game Bankruptcy Rules (Reference)

> If a player gets in a situation that requires more cash than he has on hand, he may borrow (providing there is room left on the credit line) from the bank in $5,000 increments, paying a 20% refinance penalty fee for not properly managing his finances. Another option is to borrow from his neighbors at a free market rate and terms, i.e., whatever the player can work out. If the player has already borrowed to his maximum bank credit of $50,000 and has bills that require more cash than he has on hand, he is bankrupt. He may then try to sell movable assets (cattle or equipment) to neighbors at reduced prices. Creative players may work out a temporary rental of crop land or crop futures for percentages to neighboring farmers for needed cash. If sufficient money is not raised immediately, there is a BANKRUPTCY AUCTION to raise money to pay the bankrupt player's bills.

> **BANKRUPTCY AUCTION:** All non-bankrupt players roll the die, high roller becoming the auctioneer. The auctioneer chooses what property is to be sold at the auction to satisfy the debt. Bidding among players starts at 50% of the asset's value, highest bidder wins. The auction continues until all outstanding bills and Bank Notes are paid in full and the bankrupt farmer has at least $5,000 in cash of operating money. If the player has any productive assets left, he may continue farming.

---

## 2. Bankruptcy Triggers in the Digital Game

A player is considered bankrupt when their cash balance becomes negative and they are unable to resolve the deficit through forced loans (see [loan_mechanics.md](loan_mechanics.md)).

**Triggers:**
- Payment effects (tile or card) that reduce cash below zero and cannot be covered by a loan.
- Explicit player action (e.g., "Declare Bankruptcy" in the frontend's AuctionPanel).

---

## 3. Backend Bankruptcy Processing Flow

**Primary Codepath:**  
- `packages/backend/src/services/bankruptcy.service.ts` (`BankruptcyService`)

### 3.1. Bankruptcy Check

- `BankruptcyService.checkBankruptcy(playerId, tx)`
  - Checks if the player's cash is negative.
  - Attempts to resolve via a forced loan (see [loan_mechanics.md](loan_mechanics.md)).
  - If the loan is not possible or insufficient, initiates the bankruptcy auction.

### 3.2. Interactive Bankruptcy Auction

The bankruptcy auction system has been upgraded to provide interactive, real-time auctions where all players can participate.

**Primary Services:**
- `BankruptcyAuctionService`: Manages auction creation, lot sequencing, and bidding logic
- `BankruptcyAuctionHandler`: Handles real-time socket events and timer management

**Auction Process:**
1. **Auction Creation**: When bankruptcy occurs, `BankruptcyAuctionService.createAuction()` groups the bankrupt player's assets into lots
2. **Lot Sequencing**: Assets are organized into sequential lots (cows in blocks of 10, fruit in blocks of 5, individual equipment, ridge leases)
3. **Interactive Bidding**: Each lot is auctioned with real-time bidding from all active players
4. **Timer Management**: 30-second bidding windows with automatic extension for last-minute bids
5. **Bank Buyback**: If no player bids, the bank purchases at minimum price
6. **Asset Transfer**: Winning bidders receive assets, proceeds go to bankrupt player

**Lot Creation Rules:**
- **Cows**: Auctioned in blocks of 10 (max 5 blocks)
- **Fruit**: Auctioned in blocks of 5 (max 10 blocks)
- **Equipment**: Individual items (tractors, harvesters)
- **Ridges**: Individual lease transfers
- **Priority**: High-value assets auctioned first

**Bidding Mechanics:**
- Minimum bid: Set per lot (typically 50-80% of asset value)
- Bid increments: $1 minimum increase
- Time limit: 30 seconds per lot, extends with new bids
- All active players can participate
- Bank acts as final buyer if no bids

**Auction Completion:**
- Continues until all lots are sold OR bankrupt player has sufficient funds
- If player reaches $5,000 cash minimum, they survive and continue playing
- If insufficient funds raised, player is eliminated from the game

### 3.3. Logging and State Updates

- All steps, bids, and results are logged for transparency.
- Player and asset states are updated in the database within a transaction.

---

## 4. Frontend Auction Experience

**Primary Components:**
- `BankruptcyAuctionModal`: Main auction interface for all players
- `AuctionLotCard`: Individual lot display with bidding controls and countdown timer

**Interactive Features:**
- **Real-time Updates**: Live bidding, timer countdown, and auction progress
- **Participatory Bidding**: All active players can place bids on current lots
- **Automatic Modal Display**: Auction modal appears automatically when bankruptcy auctions start
- **Bid Validation**: Prevents invalid bids (insufficient funds, below minimum)
- **Pass Option**: Players can pass on lots they're not interested in
- **Completion Notifications**: Clear feedback when auctions end (survival or elimination)

**User Experience:**
- Modal overlay ensures auction takes priority during gameplay
- Clear display of current lot, time remaining, and bidding history
- Intuitive bid input with suggested amounts
- Toast notifications for bid outcomes and auction results

---

## 5. Deviations from Board Game Rules

**Enhanced Interactive Experience:**
- **Real-time Participation**: All players can actively bid instead of automated bidding
- **Timer-based Auctions**: 30-second bidding windows with extensions for active bidding
- **Bank Buyback**: Automated bank purchase when no player bids (maintains game flow)
- **Lot Organization**: Assets grouped into logical lots for efficient auctioning

**Rule Adherence:**
- Minimum bid typically 50-80% of asset value (configurable per lot)
- Auction continues until debt is paid or all assets are sold
- $5,000 minimum cash threshold for bankrupt players to continue
- Winner-takes-all bidding (highest bid wins each lot)

**Technical Improvements:**
- Real-time synchronization across all players
- Automatic timer management and bid validation
- Comprehensive logging and audit trails
- Transaction safety with database rollbacks

---

## 6. Key Files and Codepaths

- **Backend Services:**
  - `packages/backend/src/services/bankruptcy.service.ts` (bankruptcy detection and loan attempts)
  - `packages/backend/src/services/bankruptcy-auction.service.ts` (auction creation and bidding logic)
  - `packages/backend/src/gateways/gateway-v2/handlers/bankruptcy-auction.handler.ts` (real-time auction coordination)
  - `TileEffectService` and `CardEffectService` (trigger bankruptcy checks)

- **Frontend Components:**
  - `packages/frontend/src/components/BankruptcyAuctionModal.tsx` (main auction interface)
  - `packages/frontend/src/components/AuctionLotCard.tsx` (individual lot display and bidding)
  - `packages/frontend/src/services/socketService.ts` (auction socket event handling)

- **Database Models:**
  - `BankruptcyAuction`, `AuctionLot`, `Bid` (Prisma models for auction state)

- **Shared Types:**
  - `packages/shared-types/src/marketplace-types.ts` (auction data structures)
  - `packages/shared-types/src/socket-events.ts` (auction/bankruptcy events)

---

## 7. Future Improvements

- **Auction Variations**: Support for different auction types (Dutch auctions, sealed bids)
- **Player Negotiation**: Direct player-to-player loans and asset trading during bankruptcy
- **Creative Arrangements**: Support for crop futures, land rentals, and complex deals
- **Auction Analytics**: Statistics and insights for auction performance
- **Mobile Optimization**: Enhanced mobile experience for auction participation

---

## 8. References

- [marketplace_mechanics.md](marketplace_mechanics.md): Related trading system mechanics
- [loan_mechanics.md](loan_mechanics.md): Forced loan logic and bankruptcy prevention
- [game-effects.md](game-effects.md): Effect processing and payment triggers
- `packages/backend/src/services/bankruptcy-auction.service.ts`: Interactive auction logic
- `packages/frontend/src/components/BankruptcyAuctionModal.tsx`: Modern auction UI
- `docs/ignore-archive/original_game_content/original-board-game-rules.md`: Official board game rules 