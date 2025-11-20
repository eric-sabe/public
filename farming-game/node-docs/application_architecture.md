# Farming Game Architecture Documentation

This document provides a comprehensive overview of the Farming Game application architecture, including detailed information about each package, key files, data flow, and how different components interact.

## Table of Contents

1. [Project Structure](#project-structure)
2. [Shared Types Package](#shared-types-package)
3. [Backend Architecture](#backend-architecture)
4. [Frontend Architecture](#frontend-architecture)
5. [Data Flow & Communication](#data-flow--communication)
6. [Key Entities & Models](#key-entities--models)
7. [Socket Events & Communication](#socket-events--communication)
8. [Authentication Flow](#authentication-flow)
9. [Code Paths for Common Operations](#code-paths-for-common-operations)
10. [Production Deployment](#production-deployment)

## Project Structure

The application follows a monorepo structure using pnpm workspaces:

```
farming_game/
├── .cursor/
│   └── rules/
├── .github/
│   └── workflows/
├── docs/
│   ├── about/
│   │   ├── db/
│   │   ├── github-issues/
│   │   ├── log_audit_test_areas/
│   │   └── ... (architecture, data, logging, etc.)
│   ├── api/
│   ├── ignore-archive/
│   ├── memory-bank/
│   ├── prd-active/
│   ├── prd-implemented/
│   └── ...
├── packages/
│   ├── backend/
│   │   ├── src/
│   │   │   ├── auth/
│   │   │   ├── controllers/
│   │   │   ├── gateways/gateway-v2/
│   │   │   ├── logger/
│   │   │   ├── mcp/
│   │   │   ├── modules/
│   │   │   ├── prisma/
│   │   │   ├── services/
│   │   │   └── utils/
│   │   ├── log-retention/
│   │   ├── prisma/
│   │   ├── test/
│   │   └── tools/
│   ├── frontend/
│   │   ├── src/
│   │   │   ├── assets/
│   │   │   ├── components/
│   │   │   │   ├── settings/
│   │   │   │   └── ui/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   ├── services/
│   │   │   ├── store/
│   │   │   ├── styles/
│   │   │   ├── types/
│   │   │   └── utils/
│   │   └── public/
│   ├── shared-types/
│   │   ├── src/
│   │   ├── cjs/
│   │   └── esm/
│   └── eslint-plugin-logging-quality/
├── scripts/
│   └── log_audit/
├── deployments/
├── .gitignore
├── package.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
```

**Key Directories:**
- `docs/`: Comprehensive documentation including architecture, PRDs, database specs, and GitHub issues
- `packages/backend/`: NestJS backend with modular architecture, OAuth auth, and advanced logging
- `packages/frontend/`: React frontend with settings management, OAuth callbacks, and experimental features
- `packages/shared-types/`: TypeScript types compiled to both CommonJS and ESM formats
- `packages/eslint-plugin-logging-quality/`: Custom ESLint plugin for logging quality enforcement
- `scripts/log_audit/`: Advanced log audit tooling with TypeScript AST analysis, console usage detection, and catch block analysis
- `deployments/`: Azure deployment history and configuration

## Shared Types Package

**Path**: `packages/shared-types/`

### Key Files

- `src/game-types.ts`: Core game entity types (GameState, PlayerPublic, Asset, etc.) with Zod validation schemas
- `src/socket-events.ts`: Complete socket event definitions with payloads and TypeScript interfaces
- `src/auth-types.ts`: Authentication and user management types
- `src/logging.ts`: Logging system types and enums
- `src/ai-types.ts`: AI player and decision-making types
- `src/board.ts`: Game board and tile types
- `src/deck.ts`: Card and deck management types
- `src/ridge.ts`: Ridge/leasehold property types
- `src/game-phase.ts`: Game phase definitions
- `src/log-tags.ts`: Standardized logging tags and contexts
- `src/index.ts`: Centralized exports avoiding conflicts

### Build Architecture

The shared types package builds to both CommonJS and ESM formats:
- `cjs/`: CommonJS build for backend compatibility
- `esm/`: ES Module build for modern frontend usage
- Both formats include TypeScript declarations and source maps

### Important Types

#### Game State Types (`game-types.ts`)
- `GameState`: Complete game state sent to clients
- `PlayerPublic`: Public player representation (excludes sensitive data)
- `PlayerState`: Full player state for server use
- `Asset`: Game assets (Hay, Grain, Cows, Equipment) - now uses enum values
- `Ridge`: Leasehold properties with cow counts
- `BoardTile`: Game board tiles with effects
- `CardDefinition`: Card template definitions
- `CardInstance`: Player-owned card instances
- `TemporaryHarvestModifierEntry`: Dynamic harvest modifiers with Zod validation
- `GameEffectType`: Comprehensive effect types for cards and tiles

#### Authentication Types (`auth-types.ts`)
- `User`: User account with OAuth and role management
- `UserRole`: USER, ADMIN, SERVICE_ACCOUNT, DEVELOPER
- `UserStatus`: ACTIVE, BANNED, SUSPENDED
- `Invitation`: Invitation system for user onboarding
- `AccountActivity`: Comprehensive activity tracking
- `SystemStatistics`: Admin dashboard metrics

#### Socket Event Types (`socket-events.ts`)
- **Client Events**: `client:joinGame`, `client:rollDice`, `client:leaveGame`, etc.
- **Server Events**: `server:gameStateUpdate`, `server:gameLog`, `server:autoAction`, etc.
- **Session Management**: Enhanced connection handling and auto-actions
- **Admin Events**: Real-time administrative capabilities

## Backend Architecture

**Path**: `packages/backend/`

### Key Files & Directories

- `src/main.ts`: Application entry point with custom Socket.IO adapter and logging setup
- `src/app.module.ts`: Root NestJS module with comprehensive dependency injection
- `src/auth/`: Complete OAuth authentication system (Google & Microsoft)
- `src/controllers/`: REST API controllers including `GameController`, `AdminController`, `SettingsController`, `InvitationsController`, `ExperimentsController`, `LogsController`, and `HealthController`
- `src/gateways/gateway-v2/`: Advanced WebSocket gateway with specialized handlers
- `src/services/`: Modular business logic services organized by domain
- `src/logger/`: Enhanced Winston logging with structured formatting
- `src/mcp/`: Management Control Panel API for administrative access
- `src/prisma/`: Database service and Prisma client configuration
- `log-retention/`: Advanced log retention system with archiving

### Modular Service Architecture

The backend uses a highly modular architecture with dedicated service modules:

#### Core Game Modules
- `GameModule`: Core game logic, state management, and player coordination
- `GameLoopModule`: Turn management, player actions, and game progression
- `DeckModule`: Card deck management, shuffling, and distribution
- `TileEffectModule`: Board tile effects and player movement logic
- `HarvestModule`: Harvest calculations and agricultural mechanics
- `BoardModule`: Game board state and tile management
- `AssetModule`: Player asset management and transactions
- `AIModule`: AI player decision-making and automation
- `CardEffectModule`: Comprehensive card effect implementation (2845+ lines)
- `BankruptcyModule`: Player bankruptcy processing and asset liquidation
- `PlayerModule`: Player state management and lifecycle
- `NetWorthModule`: Net worth calculations and hook system
- `PlayerUpdateModule`: Player state validation and updates

#### Infrastructure Modules
- `AuthModule`: OAuth authentication with Google and Microsoft
- `EmailModule`: Azure Communication Services with Redis job queue
- `LoggerModule`: Winston-based structured logging system
- `PrismaModule`: Global database service with connection management
- `LogDbModule`: Unified logging database operations
- `LogRetentionModule`: Advanced log retention with archiving and field size management
- `DatabaseSecurityModule`: Database security monitoring and RLS
- `HealthModule`: Health check endpoints and system monitoring

#### Administrative Modules
- `SettingsModule`: User preferences and system configuration
- `EmailModule`: Notification system with template engine
- `ExperimentsModule`: A/B testing framework and feature flags
- `InvitationsModule`: Invitation system for user onboarding
- `AdminModule`: Administrative operations and user management
- `McpModule`: Management Control Panel API for administrative access

### Gateway V2 Architecture

**Path**: `src/gateways/gateway-v2/`

The WebSocket communication uses a sophisticated handler-based architecture:

#### Specialized Handlers
- `ConnectionHandler`: Socket connection, authentication, and room management
- `TurnHandler`: Turn progression, action validation, and game flow
- `ActionHandler`: Player action processing and state updates
- `PresenceHandler`: Player presence, AFK detection, and auto-actions
- `AdminHandler`: Real-time administrative commands and monitoring

#### Gateway Services
- `BroadcastService`: Targeted and room-based message broadcasting
- `AutoActionService`: Automated actions for disconnected/AFK players with AI-driven decision making
- `SharedStateService`: Cross-handler state coordination
- `TransformService`: Data transformation and serialization

#### Handler Structure
```
handlers/
├── action.handler.ts          # Player action processing
├── admin.handler.ts           # Administrative operations
├── connection.handler.ts      # Socket lifecycle management
├── presence.handler.ts        # AFK detection and auto-actions
├── turn.handler.ts            # Turn management and validation
└── index.ts                   # Handler exports

services/
├── auto-action.service.ts     # AI-driven auto actions
├── broadcast.service.ts       # Message broadcasting
├── shared-state.service.ts    # Cross-handler coordination
├── transform.service.ts       # Data transformation
└── index.ts                   # Service exports

state/
├── shared-state.service.ts    # State coordination
└── index.ts                   # State exports

types/
├── card-types.ts              # Card-related type definitions
├── payload-types.ts           # Event payload interfaces
├── player-types.ts            # Player state types
├── ridge-types.ts             # Ridge/leasehold types
└── index.ts                   # Type exports
```

### Database Schema & Prisma Architecture

**Current Schema Entities:**

#### Core Game Entities
- `Game`: Game sessions with comprehensive state, metadata, and lifecycle management
- `Player`: Player instances with full game state, connection status, and activity tracking
- `PlayerAsset`: Asset ownership with cost, income tracking, and equipment management
- `Ridge`: Leasehold properties with cow capacity and rental income
- `CardDefinition`: Card templates with effect specifications and dynamic pricing
- `PlayerCard`: Player-owned cards with location, duration, and effect state
- `GameDeck`: Deck management for different card types (Farmer's Fate, Opportunity to Buy, etc.)
- `BoardTile`: Configurable board tiles with dynamic effects and harvest modifiers
- `GameLogEntry`: Structured game event logging with player attribution and timestamps

#### User Management System
- `User`: Complete user accounts with OAuth integration (Google/Microsoft)
- `Invitation`: Email-based invitation system for user onboarding with expiration
- `AccountActivity`: Comprehensive activity tracking and auditing with IP logging
- `Experiment`: A/B testing framework for feature flags and user segmentation
- `ExperimentParticipation`: User participation tracking in experiments

#### Logging Infrastructure
- `UnifiedLogEntry`: Centralized logging across all system components with structured metadata

### Authentication Architecture

**OAuth-Only System:**
- **Providers**: Google OAuth 2.0 and Microsoft OAuth 2.0
- **JWT Strategy**: 72-hour token expiration with user validation
- **Invitation Flow**: Email-verified invitation system for new users
- **Role-Based Access**: USER, ADMIN, SERVICE_ACCOUNT, DEVELOPER roles
- **Security**: Comprehensive audit logging and IP restrictions

### Email Infrastructure

**Azure Communication Services Integration:**
- **Service**: Azure Communication Services for transactional emails
- **Queue System**: Redis-based job queue using Bull with retry logic and failure handling
- **Templates**: Responsive HTML email templates with branding via `EmailTemplateService`
- **Processing**: Background job processing with `EmailProcessor` for reliable delivery
- **Features**: Account activity notifications, admin alerts, system updates, invitation emails
- **Domain**: Verified custom domain with SPF/DKIM authentication
- **Configuration**: Graceful degradation when email service is unavailable

### Log Retention System

**Advanced Log Management (`log-retention/`):**
- **Retention Policies**: Configurable by severity (error, warn, info, debug) and service context
- **Archiving**: JSONL.gz compression with configurable storage (filesystem/S3)
- **Field Size Management**: Automatic truncation of oversized log entries (max 10KB messages, 32KB metadata)
- **Service-Specific Rules**: Context-based retention overrides (e.g., GameService errors kept 6 months)
- **Batch Processing**: Configurable batch sizes (default 1000) for efficient cleanup
- **Phase 3 Features**: Enhanced archiving with compression levels and warning thresholds

### Key Services Detail

#### Game Logic Services
- `GameService`: Game creation, joining, state management, and turn coordination
- `GameLoopService`: Complete turn cycle management with action validation. It executes configuration-driven end-condition checks, including the `winningNetWorth` and `minTurnsForNetWorthWin` thresholds defined in `packages/backend/src/config/game.config.ts`, before marking a game `Completed`.
- `CardEffectService`: Comprehensive card effect implementation (2845+ lines)
- `TileEffectService`: Board tile effect processing with dynamic harvest modifiers
- `DeckService`: Card deck shuffling, drawing, and management
- `BankruptcyService`: Player bankruptcy processing and asset liquidation
- `HarvestService`: Agricultural calculations and yield processing with modifiers
- `BoardService`: Game board state and tile management
- `AssetService`: Player asset management and transactions
- `PlayerService`: Player lifecycle and state management
- `AIService`: AI decision-making with multiple personality profiles (Cautious, Expansionist, Gambler, Balanced)
- `RidgeService`: Leasehold property management and rental calculations
- `NetWorthService`: Net worth calculations with hook system validation

#### Infrastructure Services
- `AuthService`: OAuth handling, JWT generation, and user validation
- `EmailService`: Azure Communication Services client wrapper
- `EmailQueueService`: Redis job queue management with Bull and retry logic
- `AdminService`: User management, system statistics, and administrative operations
- `InvitationsService`: Invitation creation, validation, and lifecycle management
- `ExperimentsService`: A/B testing framework and feature flag management
- `SettingsService`: User preferences and system configuration management
- `LogDbService`: Unified logging database operations
- `DatabaseSecurityService`: Database security monitoring and RLS management

### Custom Socket.IO Adapter

**JWT Authentication Middleware (`CustomIoAdapter`):**
- Validates JWT tokens on every WebSocket connection via cookie extraction
- Extracts user context and attaches to socket data for authenticated operations
- Handles connection rejection for invalid/expired tokens with specific error messages
- Manages user-socket mapping for targeted broadcasting and room management
- Implements CORS configuration for multiple environments (local dev, Azure, custom domain)
- Provides connection lifecycle logging with structured metadata

## Frontend Architecture

**Path**: `packages/frontend/`

### Key Files & Directories

- `src/main.tsx`: Application entry point with React 18 setup and Vite configuration
- `src/App.tsx`: Root component with routing and authentication flow
- `src/pages/`: OAuth callback handling, settings management, privacy policy, and game pages
- `src/components/`: Comprehensive UI components with styled-components and responsive design
- `src/components/settings/`: Complete settings management system with admin capabilities
- `src/components/ui/`: Reusable UI components (Button, Card, Modal, etc.) with theme integration
- `src/services/`: Backend communication (authService, socketService) and authentication services
- `src/store/`: Zustand state management with experimental features and game state
- `src/utils/`: Enhanced logging, API client, harvest calculations, and utility functions
- `src/hooks/`: Custom React hooks for socket connections and auth refresh

### State Management

**Zustand Store Architecture (`src/store/gameStore.ts`):**

#### Core Game State
- `gameState: GameState | null`: Complete game state from backend
- `isConnected: boolean`: WebSocket connection status
- `logs: GameLogEntry[]`: Structured game log entries with player attribution
- `availableActions: string[]`: Current player's available actions
- `myPlayerId: string | null`: Client's player identifier
- `error: string | null`: Error state management

#### Experimental Features
- `uiMode: 'classic' | 'experimental'`: UI mode toggle with localStorage persistence (defaults to 'experimental' for new users)
- `experimentalFeatures`: Feature flags for A/B testing (barnyard defaults to true for new users)
- `immediateBuyDecisionData`: Real-time card decision prompts

#### Store Actions
- **Connection Management**: `setConnectionStatus`, `clearGameState`
- **Game State**: `setGameState`, `addLog`, `setAvailableActions`
- **Player Management**: `setMyPlayerId`, `getMyPlayerState`
- **UI Features**: `setUIMode`, `toggleExperimentalFeature`

#### Advanced Selectors
- `selectMyPlayer`: Current player's complete state
- `selectCurrentTurnPlayer`: Active turn player
- `selectIsMyTurn`: Turn validation logic
- `selectPersonalStats`: Aggregated player statistics

### Authentication Flow

**OAuth-Only System:**
1. **Invitation Entry**: Users enter invitation codes on dedicated page
2. **Provider Selection**: Choice between Google and Microsoft OAuth
3. **OAuth Redirect**: Backend handles provider authentication
4. **Callback Processing**: Frontend receives JWT token via URL parameters
5. **Token Storage**: JWT stored in localStorage for subsequent requests
6. **Protected Routes**: Route guards validate authentication status

### Settings Management System

**Comprehensive Settings Pages (`src/components/settings/`):**

#### User Settings
- `MyAccountSettings`: Profile management and account information
- `PrivacySettings`: Privacy preferences and data controls
- `GameStatsSettings`: Game statistics and performance metrics
- `ExperimentsSettings`: Feature flag participation management
- `InvitationsSettings`: Invitation management for users

#### Administrative Settings
- `AdminUserManagement`: User administration with role and status management
- `AdminInvitationManagement`: System-wide invitation oversight
- `AdminSystemStats`: System health and usage statistics

### Enhanced Logging System

**Frontend Logging (`src/utils/enhanced-logger.ts`):**
- Structured logging with context and metadata
- Session tracking and user attribution
- Console output with severity-based formatting
- Integration with backend logging for full-stack traceability

### UI Architecture

**Component Organization:**
- **Pages**: Top-level route components (LoginPage, SettingsPage, PrivacyPolicyPage, etc.)
- **Core Components**: `GameView`, `GameBoard`, `ActionPanel`, `Scoreboard`, `GameLog`
- **UI Components**: Reusable components (`Button`, `Card`, `Modal`, `Input`, `ToastManager`)
- **Settings Components**: Modular settings system with admin/user management capabilities
- **Experimental Components**: `ExperimentalGameLayout`, `UIModeSelector` for A/B testing
- **Utility Components**: Modals, toasts, form helpers, and activity timers

**Styling Strategy:**
- **Styled Components**: CSS-in-JS with theme provider and TypeScript integration
- **Tailwind CSS**: Utility-first CSS framework for rapid component development
- **Global Styles**: Consistent spacing, colors, and typography via theme system
- **Responsive Design**: Mobile-first approach with breakpoints and flexible layouts
- **Theme System**: Centralized design tokens, color palette, and component variants

## Data Flow & Communication

### Client-Server Communication Flow

#### Authentication & Authorization
1. **OAuth Initiation**: Frontend redirects to provider OAuth endpoints
2. **Backend Validation**: Server validates OAuth tokens and creates/updates users
3. **JWT Generation**: Backend issues JWT tokens for subsequent API calls
4. **Socket Authentication**: WebSocket connections authenticated via JWT middleware

#### Game Lifecycle Management
1. **Game Creation**: REST API creates games with owner privileges
2. **Game Joining**: WebSocket events handle player joining with room management
3. **Real-time Updates**: Bidirectional WebSocket communication for game state
4. **Persistent Storage**: All game state changes persisted to PostgreSQL

#### Enhanced Communication Patterns

##### WebSocket Event Flow
1. **Client Action**: User interaction triggers frontend action
2. **Event Emission**: Frontend emits typed socket event with payload
3. **Handler Processing**: Backend gateway routes to specialized handlers
4. **Service Coordination**: Handlers coordinate with appropriate business services
5. **State Updates**: Database updates and game state modifications
6. **Broadcasting**: Updates broadcast to relevant players via rooms
7. **Frontend Updates**: Client receives updates and re-renders UI

##### REST API Integration
- **Administrative Operations**: User management, system settings, invitation handling
- **Data Retrieval**: Game lists, user profiles, system statistics
- **File Operations**: Log downloads, export functionality
- **Health Checks**: System monitoring and status verification

### Enhanced Session Management

#### Connection Handling
- **Graceful Disconnections**: Planned departure vs. unexpected disconnects
- **Reconnection Logic**: Automatic reconnection with state restoration
- **AFK Detection**: Automatic action handling for inactive players
- **Grace Periods**: Configurable timeouts for reconnection attempts

#### Auto-Action System
- **Trigger Conditions**: Inactivity timeouts and disconnection events
- **Action Selection**: AI-driven decision making for disconnected players
- **Notification System**: Real-time updates about automated actions
- **Manual Override**: Players can resume control upon reconnection

## Key Entities & Models

### Enhanced Game State Model

**GameState Structure:**
```typescript
interface GameState {
  id: string;
  status: GameStatus;
  phase: GamePhase;
  currentTurnIndex: number;
  activePlayerId: string;
  players: PlayerPublic[];
  ridges: Ridge[];
  assets: Asset[];
  board: BoardTile[];
  turnNumber: number;
  currentYear: number;
  logs: GameLogEntry[];
  // ... additional state properties
}
```

### Player Management System

**PlayerPublic vs PlayerState:**
- **PlayerPublic**: Client-safe representation without sensitive data
- **PlayerState**: Complete server-side state with internal calculations
- **Player**: Database entity with full persistence and relationships

### Advanced Card System

**Card Architecture:**
- **CardDefinition**: Template definitions with effect specifications
- **PlayerCard**: Owned instances with location and duration tracking
- **Effect System**: Complex effect chains with immediate and ongoing impacts
- **Dynamic Pricing**: Context-sensitive card costs and availability

### Asset Management

**Asset Types & Calculations:**
- **Basic Assets**: Hay, Grain, Fruit with income generation
- **Livestock**: Cows with feeding requirements and milk production
- **Equipment**: Tractors and Harvesters with operational benefits
- **Real Estate**: Ridge leaseholds with cow capacity and rental income

## Socket Events & Communication

### Comprehensive Event Architecture

#### Client-to-Server Events
```typescript
// Game Lifecycle
CLIENT_EVENT_JOIN_GAME
CLIENT_EVENT_LEAVE_GAME
CLIENT_EVENT_REQUEST_ACTIONS

// Player Actions
CLIENT_EVENT_ROLL_DICE
CLIENT_EVENT_PAY_LOAN
CLIENT_EVENT_EXERCISE_OPTION
CLIENT_EVENT_APPLY_CARD_EFFECT
CLIENT_EVENT_END_TURN

// Trading & Auctions
CLIENT_EVENT_PROPOSE_SALE
CLIENT_EVENT_RESPOND_TO_SALE
CLIENT_EVENT_PLACE_BID

// Emergency Actions
CLIENT_EVENT_DECLARE_BANKRUPTCY
```

#### Server-to-Client Events
```typescript
// State Management
SERVER_EVENT_GAME_STATE_UPDATE
SERVER_EVENT_GAME_LOG
SERVER_EVENT_TURN_ACTIONS_AVAILABLE
SERVER_EVENT_TURN_START

// Player Management
SERVER_EVENT_PLAYER_JOINED
SERVER_EVENT_PLAYER_LEFT
SERVER_EVENT_AUTO_ACTION

// Game Events
SERVER_EVENT_AUCTION_STARTED
SERVER_EVENT_AUCTION_UPDATE
SERVER_EVENT_AUCTION_ENDED
SERVER_EVENT_DICE_ROLLED

// Trading & Sales
SERVER_EVENT_SALE_OFFER_RECEIVED
SERVER_EVENT_SALE_RESULT

// System Events
SERVER_EVENT_ERROR
SERVER_EVENT_GAME_OVER
SERVER_EVENT_TRANSACTION_COMPLETE
SERVER_EVENT_BANKRUPTCY_DECLARED
SERVER_EVENT_BANKRUPTCY_PROCESSED
SERVER_EVENT_GAME_LIST_UPDATE
```

### Enhanced Game Logging Architecture

**Structured Log Entries:**
- **Player Attribution**: Logs linked to specific players with automatic name resolution
- **Action Context**: Detailed metadata about game state changes
- **System Events**: Administrative and automated actions clearly marked
- **Timestamp Precision**: Millisecond-accurate event ordering
- **Metadata Enrichment**: Additional context for debugging and analysis

### Real-time Administrative Features

**Admin Socket Events:**
- Real-time user management and system monitoring
- Live game observation and intervention capabilities
- System health monitoring and alert broadcasting
- Configuration changes with immediate effect propagation

## Authentication Flow

### OAuth-Only Architecture

#### New User Registration Flow
1. **Invitation Code Entry**: User enters invitation code on frontend
2. **Code Validation**: Backend validates invitation exists and is unexpired
3. **OAuth Provider Selection**: User chooses Google or Microsoft
4. **Provider Authentication**: Redirect to OAuth provider with invitation in state
5. **Email Verification**: Backend verifies OAuth email matches invitation email
6. **Account Creation**: New user created with OAuth profile data
7. **Invitation Marking**: Invitation marked as used and linked to new user
8. **JWT Generation**: Login token issued for immediate access

#### Existing User Login Flow
1. **OAuth Initiation**: User clicks provider login button
2. **Provider Authentication**: Standard OAuth flow with provider
3. **User Lookup**: Backend finds existing user by email or provider ID
4. **Profile Update**: OAuth tokens and profile data refreshed
5. **Activity Logging**: Login event recorded with metadata
6. **JWT Generation**: Fresh token issued for session

#### Security Features
- **Email Verification**: OAuth providers handle email verification
- **Invitation Validation**: Email-specific invitation codes prevent unauthorized access
- **Activity Tracking**: Comprehensive audit logging for all authentication events
- **Token Security**: 72-hour JWT expiration with secure signing
- **Role-Based Access**: Granular permissions based on user roles

## Code Paths for Common Operations

### Enhanced Game Operations

#### Creating and Joining Games
1. **Game Creation**: 
   - `GameController.createGame()` → `GameService.createGame()`
   - Owner privileges assigned, game state initialized
   - Database persistence with transaction safety
   
2. **Player Joining**:
   - Socket event `client:joinGame` → `ConnectionHandler.handleJoinGame()`
   - Authentication validation and room assignment
   - Player state initialization and broadcasting

#### Turn Management System
1. **Turn Initiation**:
   - `TurnHandler.handleTurnStart()` → `GameLoopService.startPlayerTurn()`
   - Available actions calculation and broadcasting
   - Timeout management for inactive players

2. **Action Processing**:
   - `ActionHandler.handlePlayerAction()` → Service-specific processors
   - Validation, state updates, and effect resolution
   - Broadcasting of results and state changes

#### Advanced Card Effects

##### ImmediateOptionalBuyAsset Effect Flow

Special effect type for cards that require immediate buy/pass decision upon being drawn (e.g., "Uncle Bert's Legacy").

**Human Players:**
1. Card drawn → `CardEffectService._applyCardEffectLogic()`
2. Backend emits `server:promptImmediateBuyDecision` with 30-second timeout
3. Frontend displays `ImmediateBuyDecisionDialog` with countdown timer
4. Player has 30 seconds to decide (warning at < 10 seconds)
5. Player decision → `client:resolveImmediateBuyDecision`
6. `CardEffectService.resolveImmediateBuyAsset()` processes decision
7. State updates, asset purchases, and card discard
8. Results broadcast to all players in game
9. If timeout expires: Auto-pass and discard card

**AI Players:**
- Decision made inline via `CardEffectService.makeAIImmediateBuyDecision()`
- Enhanced AI logic considers:
  - Current asset portfolio (existing holdings)
  - Game phase (early/mid/late game strategy)
  - Cost per unit analysis (value assessment)
  - AI profile-based decision thresholds
- Immediate processing in same transaction
- No socket interaction required

**Safety Mechanisms:**
- 30-second timeout prevents stuck cards
- Player disconnect handling auto-passes pending decisions
- Transaction-based processing ensures atomicity
- Comprehensive error handling and logging

**Frontend Components:**
- `ImmediateBuyDecisionDialog`: Modal with countdown, styled with theme
- Real-time countdown with warning states
- Disabled buttons after timeout
- Proper accessibility and keyboard navigation

### Administrative Operations

#### User Management
1. **Role Changes**: `AdminController.updateUserRole()` → `AdminService.updateUserRole()`
2. **Account Actions**: Ban/unban with audit logging and notification emails
3. **System Monitoring**: Real-time statistics and health checks

#### Invitation Management
1. **Invitation Creation**: `InvitationsController.createInvitation()` → Email validation
2. **Code Generation**: Secure random code with expiration date
3. **Email Delivery**: Async job queue processing with templates
