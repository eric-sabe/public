# Marketplace Mechanics

## Overview

The marketplace system allows players to buy and sell assets, OTB cards, and ridge leases between each other during active games. This creates dynamic trading opportunities and strategic depth to the farming game.

## Core Concepts

### Marketplace Listings

Players can create listings for:
- **Assets**: Grain, Hay, Cows, Fruit, Tractors, Harvesters
- **OTB Cards**: Option-to-Buy cards from their hand
- **Ridge Leases**: Leased farmland plots

Each listing includes:
- Item type and quantity
- Asking price set by seller
- Automatic expiration after a set time
- Suggested price based on item value

### Trading Rules

#### Asset Trading
- Players can list any quantity of assets they own
- Assets must be available (not already committed)
- Cannot sell the last productive asset (prevents elimination)

#### OTB Card Trading
- Only unexercised OTB cards can be listed
- Cards must be in the player's hand
- Marketplace-purchased OTB cards have phase restrictions

#### Ridge Lease Trading
- Only leased ridges can be listed
- Cannot sell the last productive ridge (prevents elimination)

### Phase Restrictions

Marketplace-purchased OTB cards are restricted to specific game phases:
- **Restriction**: Can only be exercised during the next Spring Planting phase
- **Purpose**: Prevents immediate exercise of purchased options
- **Display**: Frontend shows clear phase restriction warnings

### Price Discovery

The system provides suggested pricing based on:
- Asset base values from game configuration
- OTB card cost from card definitions
- Ridge lease value calculations

### Transaction Flow

1. **Listing Creation**: Player selects item, sets price, confirms listing
2. **Marketplace Display**: Listing appears for all players to browse
3. **Purchase**: Buyer purchases at asking price
4. **Instant Settlement**: Money and assets transfer immediately
5. **Listing Removal**: Sold listings are removed from marketplace

## Technical Implementation

### Database Schema

```prisma
model MarketplaceListing {
  id            String   @id @default(cuid())
  gameId        String
  sellerId      String
  listingType   ListingType
  assetType     AssetType?
  quantity      Int?
  askingPrice   Int
  suggestedPrice Int?
  playerCardId  String?  @unique
  ridgeId       String?  @unique
  status        ListingStatus @default(ACTIVE)
  createdAt     DateTime @default(now())
  expiresAt     DateTime?

  seller        Player    @relation(fields: [sellerId], references: [id])
  playerCard    PlayerCard? @relation(fields: [playerCardId], references: [id])
  ridge         Ridge?    @relation(fields: [ridgeId], references: [id])
  game          Game      @relation(fields: [gameId], references: [id])
}

enum ListingType {
  ASSET
  OTB_CARD
  RIDGE_LEASE
}

enum ListingStatus {
  ACTIVE
  SOLD
  CANCELLED
  EXPIRED
}
```

### Key Services

#### MarketplaceService
- `createListing()`: Validates and creates marketplace listings
- `cancelListing()`: Allows sellers to remove active listings
- `purchaseListing()`: Processes transactions between buyers and sellers
- `getActiveListings()`: Retrieves current marketplace inventory

#### Validation Logic
- Asset availability and ownership
- Sufficient quantity for sales
- Prevention of elimination through asset sales
- Phase restrictions for purchased OTB cards

### Socket Events

#### Client-to-Server Events
- `CLIENT_EVENT_LIST_ASSET`: Create new marketplace listing
- `CLIENT_EVENT_CANCEL_LISTING`: Remove active listing
- `CLIENT_EVENT_PURCHASE_FROM_MARKETPLACE`: Buy listed item
- `CLIENT_EVENT_REQUEST_MARKETPLACE_DATA`: Get current listings

#### Server-to-Client Events
- `SERVER_EVENT_MARKETPLACE_UPDATE`: Listing changes (create/cancel/sell)
- `SERVER_EVENT_LISTING_CREATED`: New listing notification
- `SERVER_EVENT_LISTING_SOLD`: Purchase completion
- `SERVER_EVENT_LISTING_CANCELLED`: Listing removal

## Game Balance Considerations

### Economic Impact
- Creates additional cash flow opportunities
- Allows specialization (sell excess, buy needed assets)
- Can accelerate or delay player elimination
- Adds strategic trading decisions

### Prevention of Abuse
- Phase restrictions prevent immediate OTB exercise
- Elimination protection prevents self-destructive sales
- Price validation ensures reasonable marketplace values
- Instant settlement prevents transaction disputes

### Player Experience
- Real-time marketplace updates
- Clear pricing guidance
- Intuitive buy/sell interface
- Immediate transaction feedback

## Future Enhancements

### Potential Features
- Auction-style selling (Dutch auctions)
- Price negotiation system
- Marketplace statistics and trends
- Bundle sales (multiple items)
- Time-limited special offers
- Player reputation system

### Technical Improvements
- Marketplace analytics dashboard
- Advanced pricing algorithms
- Mobile-responsive marketplace UI
- Marketplace chat system
- Automated price suggestions based on market data
