# WebSocket Event Contracts

This document outlines the agreed-upon event names (constants exported from `packages/shared-types/src/socket-events.ts`) and data payloads for real-time communication between the backend (Node.js/Socket.IO) and the frontend (React/Socket.IO Client).

**Note:** For most client-to-server events, the `playerId` is derived from the authenticated socket connection on the backend and does not need to be explicitly sent in the payload, unless otherwise specified (like in `client:joinGame`).

**Authentication Note:** For all socket events, the backend derives the authenticated player's identity from the JWT token provided during the socket handshake. The `playerId` is only required in the payload for initial join or where explicitly noted. All other events use the authenticated context.

## Client-to-Server Events

Events emitted by the client to the server, using constants like `CLIENT_EVENT_JOIN_GAME`.

*   **`client:joinGame` (`CLIENT_EVENT_JOIN_GAME`)**: Client requests to join a specific game room.
    *   **Payload:** `{ gameId: string, playerId: string }` (`ClientJoinGamePayload`)
    *   **Server Action:** Adds socket to the game room (`gameId`), authenticates the user, broadcasts `playerJoined`. Sends `gameStateUpdate` and potentially `turnActionsAvailable` to the joining player.

*   **`client:requestActions` (`CLIENT_EVENT_REQUEST_ACTIONS`)**: Client requests the available actions for the current turn.
    *   **Payload:** `{ gameId: string }` (`ClientRequestActionsPayload`)
    *   **Server Action:** Validates it's the player's turn and sends `turnActionsAvailable` to the requesting player.

*   **`client:rollDice` (`CLIENT_EVENT_ROLL_DICE`)**: Player initiates their turn action (roll dice).
    *   **Payload:** `{ gameId: string }` (`ClientRollDicePayload`)
    *   **Server Action:** Validates it's the player's turn, generates roll, calls `GameLoopService.handlePlayerTurn`, emits `gameLog`, `gameStateUpdate`. May also emit `turnActionsAvailable` after movement.

*   **`client:payLoan` (`CLIENT_EVENT_PAY_LOAN`)**: Player chooses to make a payment on their loan.
    *   **Payload:** `{ gameId: string, amount: number }` (`ClientPayLoanPayload`)
    *   **Server Action:** Validates player, amount, affordability. Updates player state. Emits `gameLog`, `gameStateUpdate`, `turnActionsAvailable`.

*   **`client:exerciseOption` (`CLIENT_EVENT_EXERCISE_OPTION`)**: Player chooses to exercise an Option To Buy/Lease card.
    *   **Payload:** `{ gameId: string, playerCardId: string, cashPayment?: number, assetId?: string, ridgeId?: string }` (`ClientExerciseOptionPayload`)
    *   **cashPayment:** Optional. Amount player wants to pay in cash (≥20% of cost, ≤cost, ≤player cash). Defaults to 20% if omitted.
    *   **Server Action:** Validates player, card, affordability. Calls `GameService.exerciseOptionToBuyAsset`/`exerciseOptionToLeaseRidge`. Emits `gameLog`, `gameStateUpdate`, `turnActionsAvailable`.

*   **`client:declineOption` (`CLIENT_EVENT_DECLINE_OPTION`)**: Player chooses to decline an option to buy/lease.
    *   **Payload:** `{ gameId: string, playerCardId: string }` (`ClientDeclineOptionPayload`)
    *   **Server Action:** Validates player, card. Discards card. Emits `gameLog`, `gameStateUpdate`, `turnActionsAvailable`.

*   **`client:applyCardEffect` (`CLIENT_EVENT_APPLY_CARD_EFFECT`)**: Player chooses to play a card from their hand (e.g., Farmer's Fate).
    *   **Payload:** `{ gameId: string, playerCardId: string, cardInstanceId: number }` (`ClientApplyCardEffectPayload`) *(Note: `playerCardId` seems redundant if `cardInstanceId` is present? Check usage)*
    *   **Server Action:** Validates player, card. Calls `GameService.applyCardEffect`. Emits `gameLog`, `gameStateUpdate`, `turnActionsAvailable`.

*   **`client:endTurn` (`CLIENT_EVENT_END_TURN`)**: Player explicitly ends their turn.
    *   **Payload:** `{ gameId: string }` (`ClientEndTurnPayload`)
    *   **Server Action:** Validates it's the player's turn, advances turn via `GameService.advanceTurn`. Emits `gameLog`, `gameStateUpdate`, `turnStart`.

*   **`client:placeBid` (`CLIENT_EVENT_PLACE_BID`)**: Player places a bid in an auction.
    *   **Payload:** `{ gameId: string, amount: number }` (`ClientPlaceBidPayload`)
    *   **Server Action:** Validates bid, updates auction state. Emits `auctionUpdate`, `gameStateUpdate`.

*   **`client:declareBankruptcy` (`CLIENT_EVENT_DECLARE_BANKRUPTCY`)**: Player declares bankruptcy.
    *   **Payload:** `{ gameId: string }` (`ClientDeclareBankruptcyPayload`)
    *   **Server Action:** Processes bankruptcy. Emits `bankruptcyProcessed`, `gameStateUpdate`, `gameLog`.

*   **`client:proposeSaleOffer` (`CLIENT_EVENT_PROPOSE_SALE`)**: Player proposes to sell assets to another player.
    *   **Payload:** `{ gameId: string, buyerPlayerId: string, assetType: keyof typeof AssetType, quantity: number, price: number }` (`ClientProposeSalePayload`)
    *   **Server Action:** Creates offer, validates assets. Emits `saleOfferReceived` to the buyer.

*   **`client:respondToSaleOffer` (`CLIENT_EVENT_RESPOND_TO_SALE`)**: Player responds to a sale offer.
    *   **Payload:** `{ gameId: string, offerId: string, accepted: boolean }` (`ClientRespondToSalePayload`)
    *   **Server Action:** Processes sale if accepted. Emits `saleResult` to involved players, `gameStateUpdate` to all.

*   **`client:resolveImmediateBuyDecision` (`CLIENT_EVENT_RESOLVE_IMMEDIATE_BUY_DECISION`)**: Player submits their decision for an immediate buy/pass scenario (e.g., "Uncle Bert's Legacy").
    *   **Payload:** `{ gameId: string, cardId: string, decision: "buy" | "pass" }` (`ClientResolveImmediateBuyDecisionPayload`)
    *   **Server Action:** Processes the buy/pass decision via `CardEffectService.resolveImmediateBuyAsset`. Emits `gameLog` and `gameStateUpdate` to all players in the room.
    *   **Validation:**
        - Player must be able to afford the full cost (not just down payment)
        - Card must still be in player's hand
        - Card must be `ImmediateOptionalBuyAsset` type
        - Decision must be received before timeout expires
    *   **Timeout Handling:** If timeout expires before client response, server auto-passes and discards the card.

*   **`client:leaveGame` (`CLIENT_EVENT_LEAVE_GAME`)**: Client requests to leave the current game.
    *   **Payload:** `{ gameId: string, playerId: string, isIntentional: boolean }` (`ClientLeaveGamePayload`)
    *   **Server Action:** Updates player's connection status, removes from game room, notifies others via `server:playerLeft`. May trigger auto-action logic if it was the player's turn.

*   **`client:endGame` (`CLIENT_EVENT_END_GAME`)**: Host client requests to end the game for all players.
    *   **Payload:** `{ gameId: string, hostId: string }` (`ClientEndGamePayload`)
    *   **Server Action:** Verifies host, marks game as ended, notifies all via `server:gameEndedByHost`, disconnects sockets.

*   **`client:updateActivity` (`CLIENT_EVENT_UPDATE_ACTIVITY`)**: Client informs the server of player activity (e.g., mouse movement, button click).
    *   **Payload:** `{ gameId: string, playerId: string }` (`ClientUpdateActivityPayload`)
    *   **Server Action:** Updates `lastActivityTime`, resets AFK flags if applicable.

### Marketplace Events (Client-to-Server)

*   **`client:listAsset` (`CLIENT_EVENT_LIST_ASSET`)**: Player lists an asset for sale in the marketplace.
    *   **Payload:** `{ gameId: string, listingType: 'ASSET' | 'OTB_CARD' | 'RIDGE_LEASE', assetType?: AssetType, quantity?: number, playerCardId?: string, ridgeId?: string, askingPrice: number }` (`ClientListAssetPayload`)
    *   **Server Action:** Validates asset ownership and eligibility. Creates marketplace listing. Emits `listingCreated`, `marketplaceUpdate`.

*   **`client:cancelListing` (`CLIENT_EVENT_CANCEL_LISTING`)**: Player cancels their marketplace listing.
    *   **Payload:** `{ gameId: string, listingId: string }` (`ClientCancelListingPayload`)
    *   **Server Action:** Validates listing ownership. Cancels listing. Emits `listingCancelled`, `marketplaceUpdate`.

*   **`client:purchaseFromMarketplace` (`CLIENT_EVENT_PURCHASE_FROM_MARKETPLACE`)**: Player purchases an item from the marketplace.
    *   **Payload:** `{ gameId: string, listingId: string }` (`ClientPurchaseFromMarketplacePayload`)
    *   **Server Action:** Validates affordability and listing availability. Processes transaction. Emits `listingSold`, `gameStateUpdate`, `marketplaceUpdate`.

*   **`client:requestMarketplaceData` (`CLIENT_EVENT_REQUEST_MARKETPLACE_DATA`)**: Client requests current marketplace data.
    *   **Payload:** `{ gameId: string }` (`ClientRequestMarketplaceDataPayload`)
    *   **Server Action:** Sends current marketplace listings. Emits `marketplaceUpdate`.

### Bankruptcy Auction Events (Client-to-Server)

*   **`client:placeAuctionBid` (`CLIENT_EVENT_PLACE_AUCTION_BID`)**: Player places a bid on the current auction lot.
    *   **Payload:** `{ gameId: string, auctionId: string, lotId: string, amount: number }` (`ClientPlaceAuctionBidPayload`)
    *   **Server Action:** Validates bid amount and timing. Records bid. Emits `auctionBidPlaced`, extends timer if needed.

*   **`client:passOnLot` (`CLIENT_EVENT_PASS_ON_LOT`)**: Player explicitly passes on bidding for the current auction lot.
    *   **Payload:** `{ gameId: string, auctionId: string, lotId: string }` (`ClientPassOnLotPayload`)
    *   **Server Action:** Records pass. May end lot early if all players pass. Emits `auctionBidPlaced`.

## Server-to-Client Events

Events emitted by the server to one or more clients, using constants like `SERVER_EVENT_GAME_STATE_UPDATE`.

*   **`server:gameStateUpdate` (`SERVER_EVENT_GAME_STATE_UPDATE`)**: Server sends the complete, updated game state.
    *   **Payload:** `{ gameState: GameState }` (`ServerGameStateUpdatePayload`)
    *   **Recipients:** All players in the game room

*   **`server:gameLog` (`SERVER_EVENT_GAME_LOG`)**: Server sends enhanced log messages with structured player information.
    *   **Payload:** `{ logs: GameLogEntry[], timestamp: string }` (`ServerGameLogPayload`)
    *   **Enhanced in Phase 4:** Now sends an array of `GameLogEntry` objects instead of simple strings
    *   **GameLogEntry Structure:** `{ gameId: string, playerId?: string, playerName?: string, message: string, timestamp: string }`
    *   **Benefits:** Clean message separation, structured player data, improved frontend rendering
    *   **Recipients:** All players in the game room

*   **`server:turnActionsAvailable` (`SERVER_EVENT_TURN_ACTIONS_AVAILABLE`)**: Server informs the current player of possible actions.
    *   **Payload:** `{ availableActions: AvailableAction[] }` (`ServerTurnActionsAvailablePayload`)
    *   **Type `AvailableAction`:** `typeof CLIENT_EVENT_ROLL_DICE | typeof CLIENT_EVENT_PAY_LOAN | typeof CLIENT_EVENT_EXERCISE_OPTION | typeof CLIENT_EVENT_APPLY_CARD_EFFECT | typeof CLIENT_EVENT_END_TURN`
    *   **Recipients:** Only the active player

*   **`server:turnStart` (`SERVER_EVENT_TURN_START`)**: Server announces the start of a new turn.
    *   **Payload:** `{ activePlayerId: string, turnNumber: number }` (`ServerTurnStartPayload`)
    *   **Recipients:** All players in the game room

*   **`server:playerJoined` (`SERVER_EVENT_PLAYER_JOINED`)**: Server notifies clients that a new player has connected.
    *   **Payload:** `{ player: PlayerPublic }` (`ServerPlayerJoinedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:playerLeft` (`SERVER_EVENT_PLAYER_LEFT`)**: Server notifies clients that a player has disconnected or intentionally left.
    *   **Payload:** `{ playerId: string, isIntentional: boolean, timestamp: string }` (`ServerPlayerLeftPayload`)
    *   **Recipients:** All players in the game room

*   **`server:error` (`SERVER_EVENT_ERROR`)**: Server sends an error message if a requested action was invalid.
    *   **Payload:** `{ message: string, action?: string }` (`ServerErrorPayload`)
    *   **Recipients:** Only the player who sent the invalid request

*   **`server:gameOver` (`SERVER_EVENT_GAME_OVER`)**: Server announces the winner of the game.
    *   **Payload:** `{ winnerPlayerId: string, reason: string }` (`ServerGameOverPayload`)
    *   **Recipients:** All players in the game room

*   **`server:auctionUpdate` (`SERVER_EVENT_AUCTION_UPDATE`)**: Server provides an update on an ongoing auction.
    *   **Payload:** `{ auction: Auction | null }` (`ServerAuctionUpdatePayload`) (*Note: `Auction` type defined in `game-types.ts`*)
    *   **Recipients:** All players in the game room

*   **`server:saleOfferReceived` (`SERVER_EVENT_SALE_OFFER_RECEIVED`)**: Server sends a sale offer to the potential buyer.
    *   **Payload:** `{ offer: SaleOffer }` (`ServerSaleOfferReceivedPayload`) (*Note: `SaleOffer` type needs definition*)
    *   **Recipients:** Only the buyer player

*   **`server:saleResult` (`SERVER_EVENT_SALE_RESULT`)**: Server informs involved players about the outcome of a sale.
    *   **Payload:** `{ offerId: string, accepted: boolean, sellerId: string, buyerId: string, message: string }` (`ServerSaleResultPayload`)
    *   **Recipients:** Only the seller and buyer

*   **`server:bankruptcyProcessed` (`SERVER_EVENT_BANKRUPTCY_PROCESSED`)**: Server announces the outcome of a bankruptcy declaration.
    *   **Payload:** `{ playerId: string, eliminated: boolean, message: string }` (`ServerBankruptcyProcessedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:promptImmediateBuyDecision` (`SERVER_EVENT_PROMPT_IMMEDIATE_BUY_DECISION`)**: Server prompts a specific player to make an immediate buy/pass decision for a card effect.
    *   **Payload:** `{ gameId: string, cardId: string, cardTitle: string, assetType: AssetType, quantity: number, cost: number, downPayment: number, currentCash: number, timeoutSeconds?: number }` (`ServerPromptImmediateBuyDecisionPayload`)
    *   **Recipients:** Only the specific player whose decision is required
    *   **Timeout:** Default 30 seconds. If no response received, server auto-passes and discards the card.
    *   **Frontend Action:** Display modal dialog with countdown timer and warning at < 10 seconds remaining.

*   **`server:gameEndedByHost` (`SERVER_EVENT_GAME_ENDED_BY_HOST`)**: Server notifies all players that the host has ended the game.
    *   **Payload:** `{ gameId: string, hostId: string, hostName: string, timestamp: string }` (`ServerGameEndedByHostPayload`)
    *   **Recipients:** All players in the game room. Client should navigate user away from the game screen.

*   **`server:autoAction` (`SERVER_EVENT_AUTO_ACTION`)**: Server notifies clients that an automatic action was taken for a player due to inactivity or disconnection.
    *   **Payload:** `{ playerId: string, playerName: string, reason: "disconnected" | "inactivity", action_details: string }` (`ServerAutoActionPayload`)
    *   **Recipients:** All players in the game room. `action_details` describes the action taken (e.g., "declined decision: Uncle Bert's Legacy").

*   **`server:gameListUpdate` (`SERVER_EVENT_GAME_LIST_UPDATE`)**: Server broadcasts updated list of available games to all connected clients in the lobby.
    *   **Payload:** `{ games: GameState[] }` (`ServerGameListUpdatePayload`)
    *   **Recipients:** All connected clients in the lobby

*   **`server:auctionStarted` (`SERVER_EVENT_AUCTION_STARTED`)**: Server announces the start of an auction to all players in a room.
    *   **Payload:** `{ gameId: string, assetId: string, assetType: string, startingBid: number, currentBid: number, currentBidderId: string | null, timeRemaining: number }` (`ServerAuctionStartedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:auctionEnded` (`SERVER_EVENT_AUCTION_ENDED`)**: Server announces the end of an auction.
    *   **Payload:** `{ gameId: string, assetId: string, winningBid: number, winningBidderId: string, winningBidderName: string }` (`ServerAuctionEndedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:bankruptcyDeclared` (`SERVER_EVENT_BANKRUPTCY_DECLARED`)**: Server announces that a player has declared bankruptcy.
    *   **Payload:** `{ gameId: string, playerId: string, playerName: string, debtAmount: number }` (`ServerBankruptcyDeclaredPayload`)
    *   **Recipients:** All players in the game room

*   **`server:transactionComplete` (`SERVER_EVENT_TRANSACTION_COMPLETE`)**: Server announces a transaction has been completed.
    *   **Payload:** `{ gameId: string, playerId: string, transactionId: string, transactionType: string, amount: number, newBalance: number }` (`ServerTransactionCompletePayload`)
    *   **Recipients:** All players in the game room

*   **`server:playerAssetUpdate` (`SERVER_EVENT_PLAYER_ASSET_UPDATE`)**: Server announces a player's assets have been updated.
    *   **Payload:** `{ gameId: string, playerId: string, assetType: AssetType, quantity: number }` (`PlayerAssetUpdatePayload`)
    *   **Recipients:** All players in the game room

*   **`server:diceRolled` (`SERVER_EVENT_DICE_ROLLED`)**: Server announces the result of a dice roll.
    *   **Payload:** `{ gameId: string, playerId: string, diceValue: number, newPosition: number }` (`ServerDiceRolledPayload`)
    *   **Recipients:** All players in the game room

### Marketplace Events (Server-to-Client)

*   **`server:marketplaceUpdate` (`SERVER_EVENT_MARKETPLACE_UPDATE`)**: Server sends the complete marketplace data to requesting client.
    *   **Payload:** `{ listings: MarketplaceListing[] }` (`ServerMarketplaceUpdatePayload`)
    *   **Recipients:** Requesting client (usually the player who triggered marketplace view)

*   **`server:listingCreated` (`SERVER_EVENT_LISTING_CREATED`)**: Server announces a new listing has been created.
    *   **Payload:** `{ listing: MarketplaceListing }` (`ServerListingCreatedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:listingSold` (`SERVER_EVENT_LISTING_SOLD`)**: Server announces a listing has been sold.
    *   **Payload:** `{ listingId: string, buyerId: string, buyerName: string, sellerId: string, sellerName: string, price: number }` (`ServerListingSoldPayload`)
    *   **Recipients:** All players in the game room

*   **`server:listingCancelled` (`SERVER_EVENT_LISTING_CANCELLED`)**: Server announces a listing has been cancelled.
    *   **Payload:** `{ listingId: string, sellerId: string, sellerName: string }` (`ServerListingCancelledPayload`)
    *   **Recipients:** All players in the game room

### Bankruptcy Auction Events (Server-to-Client)

*   **`server:bankruptcyAuctionStarted` (`SERVER_EVENT_BANKRUPTCY_AUCTION_STARTED`)**: Server announces the start of a bankruptcy auction.
    *   **Payload:** `{ auctionId: string, bankruptPlayerId: string, bankruptPlayerName: string, totalLots: number, debtAmount: number }` (`ServerBankruptcyAuctionStartedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:auctionLotStarted` (`SERVER_EVENT_AUCTION_LOT_STARTED`)**: Server announces the start of a new auction lot.
    *   **Payload:** `{ auctionId: string, lot: AuctionLotData, lotNumber: number, totalLots: number, timeRemaining: number }` (`ServerAuctionLotStartedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:auctionBidPlaced` (`SERVER_EVENT_AUCTION_BID_PLACED`)**: Server announces a bid has been placed on the current auction lot.
    *   **Payload:** `{ auctionId: string, lotId: string, bidderId: string, bidderName: string, amount: number, timeRemaining: number }` (`ServerAuctionBidPlacedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:auctionLotEnded` (`SERVER_EVENT_AUCTION_LOT_ENDED`)**: Server announces an auction lot has ended.
    *   **Payload:** `{ auctionId: string, lotId: string, winnerId?: string, winnerName?: string, winningBid?: number, bankPurchased: boolean, nextLotIn?: number }` (`ServerAuctionLotEndedPayload`)
    *   **Recipients:** All players in the game room

*   **`server:bankruptcyAuctionCompleted` (`SERVER_EVENT_BANKRUPTCY_AUCTION_COMPLETED`)**: Server announces the bankruptcy auction has completed.
    *   **Payload:** `{ auctionId: string, bankruptPlayerId: string, bankruptPlayerName: string, playerEliminated: boolean, remainingDebt?: number }` (`ServerBankruptcyAuctionCompletedPayload`)
    *   **Recipients:** All players in the game room

**See also:**
- [data_structures.md](data_structures.md) for detailed payload interfaces and type definitions.
- [application_architecture.md](application_architecture.md) for system-level data flow and event handling architecture.