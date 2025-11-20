# Architecture Diagrams

This document provides visual representations of the Farming Game architecture using Mermaid diagrams. These diagrams illustrate component relationships, data flow, system architecture, and deployment structure.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        Browser[Web Browser]
        React[React Frontend]
    end
    
    subgraph "API Gateway"
        REST[REST API<br/>NestJS Controllers]
        WS[WebSocket Gateway<br/>Socket.IO]
    end
    
    subgraph "Application Layer"
        Auth[Auth Module<br/>OAuth/JWT]
        Game[Game Services<br/>20+ Modules]
        Admin[Admin Services]
        Email[Email Service]
    end
    
    subgraph "Data Layer"
        DB[(PostgreSQL<br/>Prisma ORM)]
        Redis[(Redis<br/>Bull Queues)]
        KV[Azure Key Vault<br/>Secrets]
    end
    
    subgraph "External Services"
        Google[Google OAuth]
        Microsoft[Microsoft OAuth]
        AzureEmail[Azure Communication<br/>Services]
    end
    
    Browser --> React
    React --> REST
    React --> WS
    REST --> Auth
    REST --> Game
    REST --> Admin
    WS --> Game
    WS --> Auth
    Game --> DB
    Game --> Redis
    Email --> Redis
    Email --> AzureEmail
    Auth --> Google
    Auth --> Microsoft
    Auth --> KV
    Admin --> DB
    
    style Browser fill:#e1f5ff
    style React fill:#e1f5ff
    style DB fill:#ffe1f5
    style Redis fill:#ffe1f5
    style KV fill:#ffe1f5
```

---

## 2. Monorepo Package Structure

```mermaid
graph LR
    subgraph "farming_game Monorepo"
        Root[Root Package.json<br/>pnpm workspaces]
        
        subgraph "packages/"
            Shared[shared-types<br/>Type Definitions]
            Backend[backend<br/>NestJS API]
            Frontend[frontend<br/>React App]
            ESLint[eslint-plugin-logging-quality<br/>Custom Rules]
        end
    end
    
    Root --> Shared
    Root --> Backend
    Root --> Frontend
    Root --> ESLint
    
    Backend -.depends on.-> Shared
    Frontend -.depends on.-> Shared
    Backend -.uses.-> ESLint
    Frontend -.uses.-> ESLint
    
    style Shared fill:#fff4e1
    style Backend fill:#e1f5ff
    style Frontend fill:#e1f5ff
    style ESLint fill:#f0e1ff
```

---

## 3. Backend Service Module Architecture

```mermaid
graph TB
    subgraph "NestJS App Module"
        App[AppModule]
    end
    
    subgraph "Core Game Modules"
        GameMod[GameModule]
        GameLoopMod[GameLoopModule]
        CardMod[CardEffectModule]
        TileMod[TileEffectModule]
        HarvestMod[HarvestModule]
        BoardMod[BoardModule]
        AssetMod[AssetModule]
        PlayerMod[PlayerModule]
        AIMod[AIModule]
    end
    
    subgraph "Infrastructure Modules"
        AuthMod[AuthModule]
        PrismaMod[PrismaModule]
        LoggerMod[LoggerModule]
        EmailMod[EmailModule]
        HealthMod[HealthModule]
    end
    
    subgraph "Gateway Modules"
        GatewayMod[GatewayV2Module]
    end
    
    subgraph "Specialized Modules"
        MarketMod[MarketplaceModule]
        BankMod[BankruptcyModule]
        BankAucMod[BankruptcyAuctionModule]
        NetWorthMod[NetWorthModule]
        SettingsMod[SettingsModule]
    end
    
    App --> GameMod
    App --> GameLoopMod
    App --> CardMod
    App --> TileMod
    App --> HarvestMod
    App --> BoardMod
    App --> AssetMod
    App --> PlayerMod
    App --> AIMod
    App --> AuthMod
    App --> PrismaMod
    App --> LoggerMod
    App --> EmailMod
    App --> HealthMod
    App --> GatewayMod
    App --> MarketMod
    App --> BankMod
    App --> BankAucMod
    App --> NetWorthMod
    App --> SettingsMod
    
    GatewayMod --> GameMod
    GatewayMod --> PlayerMod
    GatewayMod --> MarketMod
    GatewayMod --> BankAucMod
    
    style App fill:#ff6b6b
    style GameMod fill:#4ecdc4
    style GatewayMod fill:#95e1d3
    style AuthMod fill:#ffe66d
```

---

## 4. WebSocket Gateway V2 Architecture

```mermaid
graph TB
    subgraph "Gateway V2 Module"
        Gateway[GameGatewayV2<br/>Main Gateway]
        
        subgraph "Handlers"
            ConnHandler[ConnectionHandler<br/>Join/Leave/Auth]
            TurnHandler[TurnHandler<br/>Turn Management]
            ActionHandler[ActionHandler<br/>Player Actions]
            PresenceHandler[PresenceHandler<br/>AFK Detection]
            AdminHandler[AdminHandler<br/>Admin Commands]
            MarketHandler[MarketplaceHandler<br/>Trading]
            BankHandler[BankruptcyAuctionHandler<br/>Auctions]
        end
        
        subgraph "Services"
            Broadcast[BroadcastService<br/>Room Broadcasting]
            AutoAction[AutoActionService<br/>AI Auto-Actions]
            Transform[TransformService<br/>Data Transformation]
            SharedState[SharedStateService<br/>State Coordination]
        end
    end
    
    subgraph "Business Services"
        GameService[GameService]
        GameLoopService[GameLoopService]
        CardEffectService[CardEffectService]
        MarketplaceService[MarketplaceService]
        BankruptcyAuctionService[BankruptcyAuctionService]
    end
    
    Gateway --> ConnHandler
    Gateway --> TurnHandler
    Gateway --> ActionHandler
    Gateway --> PresenceHandler
    Gateway --> AdminHandler
    Gateway --> MarketHandler
    Gateway --> BankHandler
    
    ConnHandler --> Broadcast
    TurnHandler --> Broadcast
    ActionHandler --> Broadcast
    PresenceHandler --> AutoAction
    
    TurnHandler --> GameLoopService
    ActionHandler --> GameService
    ActionHandler --> CardEffectService
    MarketHandler --> MarketplaceService
    BankHandler --> BankruptcyAuctionService
    
    Broadcast --> Transform
    AutoAction --> SharedState
    
    style Gateway fill:#ff6b6b
    style ConnHandler fill:#4ecdc4
    style TurnHandler fill:#4ecdc4
    style ActionHandler fill:#4ecdc4
    style Broadcast fill:#95e1d3
    style AutoAction fill:#95e1d3
```

---

## 5. Request/Response Flow: Player Action

```mermaid
sequenceDiagram
    participant Client as React Client
    participant Socket as Socket.IO Client
    participant Gateway as GameGatewayV2
    participant Handler as ActionHandler
    participant Service as GameService
    participant DB as PostgreSQL
    participant Broadcast as BroadcastService
    
    Client->>Socket: User clicks "Roll Dice"
    Socket->>Gateway: emit('client:rollDice', payload)
    Gateway->>Handler: handleRollDice(payload)
    Handler->>Service: validateTurn(playerId)
    Service->>DB: Query player state
    DB-->>Service: Player data
    Service-->>Handler: Validation result
    
    alt Valid Turn
        Handler->>Service: executeRollDice(playerId)
        Service->>DB: Update game state
        DB-->>Service: Updated state
        Service-->>Handler: Game state + dice result
        Handler->>Broadcast: broadcastToRoom('server:gameStateUpdate')
        Broadcast->>Socket: Emit to all clients in room
        Socket-->>Client: Update UI with new state
    else Invalid Turn
        Handler->>Socket: Emit error to client
        Socket-->>Client: Show error message
    end
```

---

## 6. Authentication Flow

```mermaid
sequenceDiagram
    participant User as User Browser
    participant Frontend as React Frontend
    participant Backend as NestJS Backend
    participant OAuth as OAuth Provider<br/>(Google/Microsoft)
    participant DB as PostgreSQL
    participant KV as Azure Key Vault
    
    User->>Frontend: Click "Login with Google"
    Frontend->>Backend: GET /auth/google
    Backend->>OAuth: Redirect to OAuth
    OAuth->>User: Login prompt
    User->>OAuth: Enter credentials
    OAuth->>Backend: Callback with code
    Backend->>OAuth: Exchange code for tokens
    OAuth-->>Backend: Access token + profile
    Backend->>DB: Find or create user
    DB-->>Backend: User data
    Backend->>KV: Get JWT secret
    KV-->>Backend: Secret key
    Backend->>Backend: Generate JWT token
    Backend->>Frontend: Set HTTP-only cookie with JWT
    Frontend->>User: Redirect to game lobby
```

---

## 7. Database Schema Relationships

```mermaid
erDiagram
    User ||--o{ Player : "has"
    User ||--o{ Invitation : "creates"
    User ||--o{ AccountActivity : "generates"
    
    Game ||--o{ Player : "contains"
    Game ||--o{ Ridge : "has"
    Game ||--o{ BoardTile : "has"
    Game ||--o{ GameDeck : "has"
    Game ||--o{ GameLogEntry : "generates"
    Game ||--o{ MarketplaceListing : "has"
    Game ||--o{ BankruptcyAuction : "has"
    
    Player ||--o{ PlayerAsset : "owns"
    Player ||--o{ PlayerCard : "holds"
    Player ||--o{ Bid : "places"
    Player ||--o{ AuctionLot : "bids_on"
    
    CardDefinition ||--o{ PlayerCard : "instantiates"
    
    Ridge ||--o| Player : "leased_by"
    
    MarketplaceListing ||--o| PlayerCard : "references"
    MarketplaceListing ||--o| Ridge : "references"
    
    BankruptcyAuction ||--o{ AuctionLot : "contains"
    AuctionLot ||--o{ Bid : "receives"
    
    Game {
        string id PK
        GameStatus status
        int currentTurnIndex
        GamePhase phase
        string activePlayerId FK
    }
    
    Player {
        string id PK
        string gameId FK
        string userId FK
        string name
        int cash
        int debt
        int netWorth
        boolean isAI
    }
    
    User {
        string id PK
        string email
        UserRole role
        UserStatus status
    }
    
    CardDefinition {
        string id PK
        string title
        GameEffectType effectType
        json effectDetails
    }
    
    PlayerCard {
        string id PK
        string playerId FK
        string cardDefinitionId FK
        CardLocation location
    }
```

---

## 8. Frontend State Management Flow

```mermaid
graph LR
    subgraph "React Components"
        GameView[GameView]
        ActionPanel[ActionPanel]
        Scoreboard[Scoreboard]
        GameLog[GameLog]
    end
    
    subgraph "Zustand Store"
        Store[gameStore]
        State[Game State]
        Actions[Store Actions]
    end
    
    subgraph "Socket Service"
        Socket[socketService]
        Events[Event Handlers]
    end
    
    subgraph "Backend"
        Gateway[WebSocket Gateway]
    end
    
    GameView --> Store
    ActionPanel --> Store
    Scoreboard --> Store
    GameLog --> Store
    
    Store --> State
    Store --> Actions
    
    Actions --> Socket
    Socket --> Events
    Events --> Gateway
    
    Gateway --> Events
    Events --> Store
    Store --> State
    State --> GameView
    State --> ActionPanel
    State --> Scoreboard
    State --> GameLog
    
    style Store fill:#ff6b6b
    style Socket fill:#4ecdc4
    style Gateway fill:#95e1d3
```

---

## 9. Build & Deployment Pipeline

```mermaid
graph LR
    subgraph "Development"
        Dev[Local Development<br/>pnpm run dev]
    end
    
    subgraph "Build Process"
        Build1[Build shared-types<br/>pnpm --filter shared-types build]
        Build2[Build backend<br/>pnpm --filter backend build]
        Build3[Build frontend<br/>pnpm --filter frontend build]
    end
    
    subgraph "Testing"
        Test[Run Tests<br/>pnpm test]
        TypeCheck[Type Check<br/>pnpm typecheck]
    end
    
    subgraph "CI/CD"
        GitHub[GitHub Actions]
        Docker[Docker Build]
        Azure[Azure App Service]
    end
    
    Dev --> Build1
    Build1 --> Build2
    Build2 --> Build3
    Build3 --> Test
    Test --> TypeCheck
    TypeCheck --> GitHub
    GitHub --> Docker
    Docker --> Azure
    
    style Build1 fill:#fff4e1
    style Build2 fill:#e1f5ff
    style Build3 fill:#e1f5ff
    style GitHub fill:#95e1d3
    style Azure fill:#ff6b6b
```

---

## 10. Logging & Monitoring Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        Backend[Backend Services]
        Frontend[Frontend Components]
    end
    
    subgraph "Logging Layer"
        Winston[Winston Logger]
        UnifiedLog[UnifiedLogEntry]
        LogDB[LogDbService]
    end
    
    subgraph "Storage Layer"
        PostgreSQL[(PostgreSQL<br/>UnifiedLogEntry Table)]
        Files[Log Files<br/>JSONL.gz]
        Archive[Archive Storage<br/>S3/Filesystem]
    end
    
    subgraph "Retention System"
        Retention[LogRetentionModule]
        ArchiveService[Archive Service]
    end
    
    Backend --> Winston
    Frontend --> Winston
    Winston --> UnifiedLog
    UnifiedLog --> LogDB
    LogDB --> PostgreSQL
    
    Retention --> PostgreSQL
    Retention --> ArchiveService
    ArchiveService --> Files
    ArchiveService --> Archive
    
    style Winston fill:#4ecdc4
    style UnifiedLog fill:#95e1d3
    style Retention fill:#ffe66d
    style PostgreSQL fill:#ff6b6b
```

---

## 11. Marketplace & Bankruptcy Auction Flow

```mermaid
sequenceDiagram
    participant Player1 as Player 1<br/>(Seller)
    participant Player2 as Player 2<br/>(Buyer)
    participant Market as MarketplaceService
    participant Auction as BankruptcyAuctionService
    participant DB as PostgreSQL
    participant Broadcast as BroadcastService
    
    Note over Player1,Market: Marketplace Flow
    Player1->>Market: List asset for sale
    Market->>DB: Create MarketplaceListing
    Market->>Broadcast: Emit 'marketplaceUpdate'
    Broadcast->>Player2: Show listing
    
    Player2->>Market: Purchase listing
    Market->>DB: Update listing status
    Market->>DB: Transfer assets
    Market->>Broadcast: Emit 'gameStateUpdate'
    
    Note over Player1,Auction: Bankruptcy Auction Flow
    Player1->>Auction: Declare bankruptcy
    Auction->>DB: Create BankruptcyAuction
    Auction->>DB: Create AuctionLots
    Auction->>Broadcast: Emit 'auctionStarted'
    
    loop For each lot
        Auction->>Broadcast: Emit 'lotActive'
        Player2->>Auction: Place bid
        Auction->>DB: Create Bid record
        Auction->>Broadcast: Emit 'auctionUpdate'
    end
    
    Auction->>DB: Finalize auction
    Auction->>Broadcast: Emit 'auctionEnded'
```

---

## 12. Email Queue Processing Flow

```mermaid
graph LR
    subgraph "Application"
        Service[EmailService]
        Queue[EmailQueueService]
    end
    
    subgraph "Redis Queue"
        BullQueue[Bull Queue<br/>Redis]
        Jobs[Email Jobs]
    end
    
    subgraph "Processing Layer"
        Processor[EmailProcessor]
        Retry[Retry Logic]
    end
    
    subgraph "External"
        AzureEmail[Azure Communication<br/>Services]
    end
    
    Service --> Queue
    Queue --> BullQueue
    BullQueue --> Jobs
    Jobs --> Processor
    Processor --> AzureEmail
    
    AzureEmail -.failure.-> Retry
    Retry --> Processor
    
    style BullQueue fill:#ff6b6b
    style Processor fill:#4ecdc4
    style AzureEmail fill:#95e1d3
```

---

## Summary

These diagrams illustrate:

1. **System Architecture:** High-level overview of all components and their relationships
2. **Package Structure:** Monorepo organization and dependencies
3. **Service Modules:** Backend modular architecture with 20+ specialized modules
4. **WebSocket Gateway:** Real-time communication architecture with specialized handlers
5. **Request Flow:** Sequence of operations for player actions
6. **Authentication:** OAuth flow with JWT cookie management
7. **Database Schema:** Entity relationships and data model
8. **Frontend State:** Zustand store and component interactions
9. **Build Pipeline:** Development to production deployment process
10. **Logging System:** Comprehensive logging architecture with retention
11. **Marketplace/Auction:** Trading and bankruptcy auction flows
12. **Email Processing:** Queue-based email delivery system

These visual representations complement the textual documentation and provide a comprehensive understanding of the Farming Game architecture.

