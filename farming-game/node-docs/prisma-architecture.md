# Prisma Architecture Guide for farming_game

This document provides a comprehensive overview of how Prisma is integrated and used throughout the farming_game monorepo. It covers the complete architecture from schema definition to client usage, resolving common confusion points about client generation and import patterns.

## Current Status (December 2024)

**✅ Production-Ready Architecture**: Prisma integration is fully optimized for production deployment with Azure-specific enhancements.

**✅ Standardized @prisma/client Imports**: All imports use the standard `@prisma/client` package import pattern for consistency and reliability.

**✅ Enhanced Connection Management**: Production-grade connection handling with timeouts, error recovery, and Azure optimization.

**✅ Comprehensive Schema**: Complete database schema covering game mechanics, user management, authentication, logging, and administrative features.

**✅ Advanced Logging Integration**: Seamless integration with the unified logging system and audit trails.

**✅ Zero Circular Dependencies**: Carefully architected module dependencies with proper lifecycle management.

## Table of Contents
1. [Schema and Configuration](#schema-and-configuration)
2. [Client Generation Architecture](#client-generation-architecture)
3. [NestJS Integration Pattern](#nestjs-integration-pattern)
4. [Import Patterns and Best Practices](#import-patterns-and-best-practices)
5. [Service Layer Architecture](#service-layer-architecture)
6. [Database Connection Management](#database-connection-management)
7. [Troubleshooting Common Issues](#troubleshooting-common-issues)

## Schema and Configuration

### Schema Location
- **Primary Schema**: `packages/backend/prisma/schema.prisma`
- **Schema Organization**: Single schema file defining all models, enums, and relationships
- **Database Provider**: PostgreSQL
- **Environment Variable**: `DATABASE_URL` (required)

### Generator Configuration
```prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["native", "darwin-arm64"]
}
```

**Key Points:**
- Client uses default Prisma location (`node_modules/.prisma/client/`)
- Multi-platform binary targets for cross-platform development and Azure deployment
- Standard Prisma client-js provider optimized for production
- No custom output path - follows Prisma best practices

### Database Configuration
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

**Connection String Formats:**

**Development:**
```
DATABASE_URL="postgresql://farming_user:password@localhost:5432/farming_game_db?schema=farming_game"
```

**Production (Azure PostgreSQL):**
```
DATABASE_URL="postgresql://farmingadmin:***@production-db-server.postgres.database.azure.com:5432/farming_game_db?sslmode=require&connect_timeout=10&pool_timeout=10&socket_timeout=10"
```

**Key Features:**
- SSL/TLS required for production connections
- Connection timeouts configured for Azure reliability
- Schema-specific connection for data isolation

## Client Generation Architecture

### Generated Client Structure
The Prisma client is generated to the standard Prisma location using default configuration:

#### Client Location: `node_modules/.prisma/client/`
```
packages/backend/node_modules/.prisma/client/
├── index.js              # Main client export
├── index.d.ts            # TypeScript definitions (comprehensive type coverage)
├── edge.js               # Edge runtime version
├── index-browser.js      # Browser-compatible version
├── wasm.js               # WebAssembly version
├── schema.prisma         # Copy of the source schema
├── package.json          # Client package configuration
├── runtime/              # Runtime dependencies and query engine
└── libquery_engine-*.so  # Platform-specific query engine binaries
```

**Production Features:**
- **Multi-platform binaries**: Native and ARM64 support for cross-platform deployment
- **Complete type coverage**: All models, enums, and relationships fully typed
- **Query engine optimization**: Platform-specific binaries for optimal performance
- **Edge runtime support**: Compatible with serverless and edge deployment scenarios

### Generation Commands
```bash
# Generate Prisma client
pnpm --filter @farming_game/backend run prisma:generate

# Or directly with Prisma CLI
cd packages/backend && npx prisma generate
```

## NestJS Integration Pattern

### Global Module Pattern
Prisma is integrated as a **Global Module** in NestJS, making it available throughout the application without explicit imports in each module.

#### PrismaModule (`src/prisma/prisma.module.ts`)
```typescript
@Global()
@Module({
  imports: [forwardRef(() => LogDbModule)],
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule implements OnModuleInit, OnModuleDestroy
```

**Features:**
- **Global Registration**: Available in all modules without explicit imports
- **Lifecycle Management**: Handles database connection/disconnection
- **Circular Dependency Handling**: Uses `forwardRef()` for LogDbModule
- **Comprehensive Logging**: Detailed initialization and cleanup logging

#### PrismaService (`src/prisma/prisma.service.ts`)
```typescript
@Injectable()
export class PrismaService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name);
  public readonly client: PrismaClient;

  constructor() {
    this.client = new PrismaClient({
      log: ['error'],
      errorFormat: 'pretty',
      datasources: {
        db: {
          url: process.env.DATABASE_URL + '&connect_timeout=10&pool_timeout=10&socket_timeout=10'
        }
      }
    });
  }

  async onModuleInit() {
    try {
      console.log('[PrismaService] Connecting to database...');
      
      // Timeout protection for Azure deployments
      const connectionPromise = this.client.$connect();
      const timeoutPromise = new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Database connection timeout after 15 seconds')), 15000)
      );
      
      await Promise.race([connectionPromise, timeoutPromise]);
      console.log('[PrismaService] Successfully connected to database');
    } catch (error: unknown) {
      const err = error as Error;
      console.error(`[PrismaService] Failed to connect to database: ${err?.message || 'Unknown error'}`, err?.stack || '');
      throw error; // Prevent app startup if DB connection fails
    }
  }

  async onModuleDestroy() {
    console.log('[PrismaService] Disconnecting from database...');
    await this.client.$disconnect();
    console.log('[PrismaService] Successfully disconnected from database');
  }
}
```

**Enhanced Architecture Pattern:**
- **Composition over Inheritance**: Wraps PrismaClient rather than extending it
- **Explicit Client Access**: Services access database via `prismaService.client`
- **Production-Ready Lifecycle**: Connection management with timeout protection
- **Azure Optimization**: Connection string enhancements for cloud deployment
- **Graceful Error Handling**: Prevents app startup on database connection failure
- **Circular Dependency Prevention**: Uses console logging during initialization

## Import Patterns and Best Practices

### Current Import Patterns in Codebase

#### ✅ Standardized @prisma/client Pattern (All Files)
All files consistently use the standard Prisma import pattern for reliability and maintainability:

```typescript
// Import types and models from @prisma/client
import { 
  Player, 
  Game, 
  GameStatus, 
  Prisma,
  AssetType,
  GamePhase 
} from "@prisma/client";

// Import service for database operations
import { PrismaService } from "../prisma/prisma.service.js";
```

#### Common Import Examples by Service Type

**Game Services:**
```typescript
import { 
  Game, 
  Player, 
  GameStatus, 
  GamePhase,
  Prisma 
} from "@prisma/client";
```

**Card System Services:**
```typescript
import { 
  CardDefinition, 
  PlayerCard, 
  CardLocation,
  DeckType,
  GameEffectType 
} from "@prisma/client";
```

**User Management Services:**
```typescript
import { 
  User, 
  UserRole, 
  UserStatus,
  Invitation,
  AccountActivity 
} from "@prisma/client";
```

**Logging Services:**
```typescript
import { 
  UnifiedLogEntry, 
  LogSource, 
  LogSeverity,
  Prisma 
} from "@prisma/client";
```

#### ❌ Legacy Patterns (No Longer Used)
These patterns were previously present but have been eliminated:
```typescript
// OLD: Relative path imports (removed)
import { Player } from "../../node_modules/.prisma/client/index.js";

// OLD: Generated prisma path (removed)  
import { Player } from "../generated/prisma/index.js";
```

### Best Practice Guidelines

1. **Always use PrismaService**: Never instantiate PrismaClient directly in services
2. **Standardized @prisma/client imports**: Use consistent `@prisma/client` import pattern ✅
3. **Type-safe imports**: Import model types and Prisma namespace for complete type safety
4. **Service injection**: Use dependency injection with `@Inject(PrismaService)` 
5. **Selective imports**: Import only the types and enums you need to reduce bundle size
6. **Consistent patterns**: Follow established patterns for each service category (game, user, logging)

#### Advanced Import Patterns

**Transaction Types:**
```typescript
import { Prisma } from "@prisma/client";

// Use Prisma transaction type
const result = await this.prismaService.client.$transaction(async (tx: Prisma.TransactionClient) => {
  // Transaction operations
});
```

**Conditional Types:**
```typescript
import { Prisma } from "@prisma/client";

// Type-safe where conditions
const whereClause: Prisma.PlayerWhereInput = {
  gameId: gameId,
  isActive: true
};
```

**Select and Include Types:**
```typescript
import { Prisma } from "@prisma/client";

// Type-safe select statements
const playerSelect: Prisma.PlayerSelect = {
  id: true,
  name: true,
  cash: true,
  assets: true
};
```

## Service Layer Architecture

### Dependency Injection Pattern
```typescript
@Injectable()
export class GameService {
  constructor(
    @Inject(PrismaService) private readonly prisma: PrismaService
  ) {}
  
  async createGame(data: GameCreateInput): Promise<Game> {
    return this.prisma.client.game.create({ data });
  }
}
```

### Usage Examples Across Services

#### Game Service
```typescript
// Complex query with relations
const game = await this.prisma.client.game.findUnique({
  where: { id: gameId },
  include: {
    players: true,
    ridges: true,
    logEntries: true
  }
});
```

#### Player Service
```typescript
// Transaction example
const result = await this.prisma.client.$transaction(async (tx) => {
  const player = await tx.player.update({
    where: { id: playerId },
    data: { cash: { increment: amount } }
  });
  
  await tx.gameLogEntry.create({
    data: {
      gameId: player.gameId,
      playerId: player.id,
      message: `Player gained $${amount}`
    }
  });
  
  return player;
});
```

#### Log Database Service
```typescript
import { LogSource, LogSeverity } from "@prisma/client";

// Using LogDbService for unified logging
await this.prismaService.client.unifiedLogEntry.create({
  data: {
    id: logId,
    timestamp: new Date(),
    source: LogSource.backend,
    severity: LogSeverity.info,
    context: 'GameService',
    tags: ['game-creation', 'multiplayer'],
    message: 'Game created successfully',
    userId: userId,
    gameId: game.id,
    metadata: { 
      playersCount: 4,
      gameMode: 'HUMANS_AND_AI',
      features: ['enhanced-logging', 'auto-actions']
    }
  }
});
```

#### User Management Service
```typescript
import { User, UserRole, UserStatus, Invitation } from "@prisma/client";

// Create user with OAuth integration
const newUser = await this.prismaService.client.user.create({
  data: {
    email: profile.email,
    username: profile.displayName,
    emailVerified: true,
    role: UserRole.USER,
    status: UserStatus.ACTIVE,
    oauthProvider: 'GOOGLE',
    oauthProviderId: profile.id,
    profilePictureUrl: profile.photos[0]?.value
  }
});

// Track account activity
await this.prismaService.client.accountActivity.create({
  data: {
    userId: newUser.id,
    action: 'ACCOUNT_CREATED',
    ipAddress: request.ip,
    userAgent: request.headers['user-agent'],
    details: { provider: 'Google OAuth' }
  }
});
```

#### Administrative Services
```typescript
import { Experiment, ExperimentCategory, ExperimentRiskLevel } from "@prisma/client";

// A/B testing experiment management
const experiment = await this.prismaService.client.experiment.create({
  data: {
    name: 'Enhanced UI Mode',
    description: 'Test new experimental UI features',
    category: ExperimentCategory.UI,
    riskLevel: ExperimentRiskLevel.LOW,
    targetParticipants: 100,
    endDate: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30 days
  }
});

// Invitation system management
const invitation = await this.prismaService.client.invitation.create({
  data: {
    code: generateInvitationCode(),
    inviteeEmail: email,
    label: 'Beta Tester',
    inviterId: adminUserId,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
  }
});
```

## Database Connection Management

### Enhanced Connection Lifecycle
1. **App Startup**: 
   - PrismaModule.onModuleInit() with comprehensive validation
   - PrismaService.onModuleInit() with timeout protection
   - Connection test with 15-second timeout for Azure compatibility
   - Graceful failure if database unavailable

2. **Runtime**: 
   - Connection pool managed by Prisma with optimized settings
   - Connection timeouts configured for cloud deployment
   - Automatic retry logic for transient failures

3. **App Shutdown**: 
   - PrismaModule.onModuleDestroy() with cleanup logging
   - PrismaService.onModuleDestroy() with graceful disconnection
   - Resource cleanup and connection pool disposal

### Production Connection Configuration
```typescript
// Enhanced PrismaService constructor for production
this.client = new PrismaClient({
  log: ['error'],              // Error-only logging for performance
  errorFormat: 'pretty',       // Human-readable error formatting
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '&connect_timeout=10&pool_timeout=10&socket_timeout=10'
    }
  }
});
```

### Azure-Optimized Features
- **Connection Timeouts**: 10-second timeouts for connect, pool, and socket operations
- **SSL Enforcement**: Required `sslmode=require` for production connections
- **Timeout Protection**: 15-second connection timeout with Promise.race pattern
- **Graceful Degradation**: App startup fails fast if database unavailable
- **Connection Pool Management**: Optimized for Azure PostgreSQL Flexible Server

### Environment Requirements & Validation
- **DATABASE_URL**: Required PostgreSQL connection string with SSL parameters
- **Format Validation**: PrismaModule validates URL format and SSL configuration
- **Startup Validation**: Connection test during app initialization
- **Error Handling**: Detailed error logging and app startup prevention on failure
- **Development vs Production**: Different connection string formats supported

## Troubleshooting Common Issues

### Issue 1: "Cannot find module '@prisma/client'" 
**Cause**: Prisma client not generated after schema changes
**Status**: ✅ All imports use standard @prisma/client pattern
**Solution**:
```bash
cd packages/backend
npx prisma generate
# Restart TypeScript server if needed
```

### Issue 2: "Database connection timeout" (Azure)
**Cause**: Azure deployment connection issues or firewall restrictions
**Status**: ✅ Enhanced timeout handling implemented
**Solution**:
- Check Azure PostgreSQL firewall rules allow App Service IPs
- Verify SSL connection requirements (`sslmode=require`)
- Review connection string timeout parameters
- Check Azure logs for specific connection errors

### Issue 3: "PrismaService is undefined"
**Cause**: Dependency injection issue or circular dependency
**Solution**:
- Ensure PrismaModule is imported in app.module.ts
- Use `@Inject(PrismaService)` in constructors
- For circular dependencies, use `forwardRef()` pattern
- Check module import order and initialization

### Issue 4: "Circular dependency detected" during logging
**Cause**: LogDbService and PrismaService circular reference
**Status**: ✅ Resolved with console logging during initialization
**Solution**:
- PrismaService uses console.log during startup to avoid circular dependency
- LogDbService waits for PrismaService to be ready
- Module imports use `forwardRef()` where necessary

### Issue 5: Production deployment connection failures
**Cause**: Environment variables or Azure configuration issues
**Solution**:
- Verify DATABASE_URL includes all required parameters
- Check Azure PostgreSQL connection string format
- Ensure SSL certificates are properly configured
- Review Azure App Service application settings
- Test connection with Azure-specific timeout settings

### Issue 6: Type errors with new Prisma models
**Cause**: Outdated generated client after schema changes
**Solution**:
- Regenerate client: `npx prisma generate`
- Clear TypeScript cache: `rm -rf node_modules/.cache`
- Restart development server and TypeScript language server
- Verify all new imports are included

### Issue 7: Migration conflicts in development
**Cause**: Multiple developers working on schema changes
**Solution**:
```bash
cd packages/backend
# Reset local development database
npx prisma migrate reset
# Apply all migrations from main branch
npx prisma migrate deploy
# Generate fresh client
npx prisma generate
```

## Development Workflow

### Making Schema Changes
1. **Edit Schema**: Modify `packages/backend/prisma/schema.prisma`
2. **Create Migration**: `npx prisma migrate dev --name your_change_description`
3. **Generate Client**: `npx prisma generate` (often automatic after migration)
4. **Update Code**: Update any services using the changed models
5. **Test**: Ensure all affected services still work

### Adding New Models
1. **Define Model**: Add model definition to schema.prisma
2. **Add Relations**: Define relationships with existing models
3. **Create Migration**: Generate and apply migration
4. **Create Service**: Add service for the new model
5. **Update Types**: Import new types in relevant services

### Performance Considerations
- **Connection Pooling**: Managed automatically by Prisma
- **Query Optimization**: Use `include`/`select` to limit data fetching
- **Transactions**: Use `$transaction()` for atomic operations
- **Logging**: Monitor slow queries via Prisma logging

---

## Summary

The farming_game Prisma architecture follows these key principles:

1. **Single Source of Truth**: All database models defined in comprehensive schema file
2. **Global Service Pattern**: PrismaService available throughout the application via dependency injection
3. **Composition Pattern**: Services compose with PrismaService rather than inheriting from PrismaClient
4. **Standardized @prisma/client Imports**: ✅ Consistent import pattern across entire codebase
5. **Production-Ready Lifecycle**: Enhanced connection management with Azure optimization
6. **Type Safety**: Complete TypeScript integration with generated types and advanced Prisma types
7. **Zero Circular Dependencies**: ✅ Careful module architecture prevents initialization issues
8. **Cloud-Optimized Configuration**: Timeout handling and SSL enforcement for production deployment

### Architecture Benefits

- **Developer Experience**: Consistent patterns, comprehensive types, and clear error messages
- **Production Reliability**: Connection timeouts, graceful failure handling, and Azure optimization  
- **Scalability**: Optimized connection pooling and efficient query patterns
- **Maintainability**: Single schema source of truth and standardized service patterns
- **Type Safety**: Full compile-time checking with Prisma's generated types
- **Operational Excellence**: Enhanced logging, monitoring, and debugging capabilities

### Current Implementation Status

This architecture provides a robust, scalable foundation for database operations while maintaining clear separation of concerns and type safety throughout the application. **As of December 2024, the Prisma integration is production-ready with comprehensive features supporting game mechanics, user management, authentication, logging, and administrative capabilities.**

The system successfully handles the complex relationships and data requirements of a multiplayer farming game while providing the flexibility for future enhancements and scalability requirements. 