# Comprehensive Logging Strategy for farming_game

This document outlines the comprehensive logging strategy for the `farming_game` project, covering both backend and frontend applications. The goal is to establish a standardized, controllable, and effective logging system that aids in development, debugging, and monitoring. Consistent logging practices are crucial for maintaining code quality, understanding application behavior, and diagnosing issues efficiently.

## Recent Updates (2025-05-23)
- **Log Retention & Cleanup Phase 3**: Completed implementation of advanced features including context-based retention periods, S3 storage support, field size management, and enhanced reporting
- **Unified Database Logging**: Implemented a new unified PostgreSQL-based logging system that stores both backend and frontend logs in a single database table for easier querying and analysis
- **Enhanced Frontend Logger**: Added a robust frontend logging system with batching, remote transmission, and configurable settings
- **Data Scrubbing**: Added automatic scrubbing of sensitive information from logs
- **Rate Limiting**: Implemented a 1000 logs/minute rate limit to prevent database flooding
- **Error Resilience**: Improved error handling and fallback mechanisms for more reliable logging

## 1. Introduction

This document outlines the comprehensive logging strategy for the `farming_game` project, covering both backend and frontend applications. The goal is to establish a standardized, controllable, and effective logging system that aids in development, debugging, and monitoring. Consistent logging practices are crucial for maintaining code quality, understanding application behavior, and diagnosing issues efficiently.

## 2. Backend Logging Strategy (NestJS with Winston)

The backend logging system aims to replace inconsistent and overly verbose logging with a robust, configurable, and standardized approach using NestJS\'s built-in logger interface, backed by Winston. This allows developers to easily control log verbosity based on severity level, message source (context), and specific message categories (tags) via environment variables, improving debugging efficiency and reducing log noise in production.

### 2.1. Goal

To implement a robust, configurable, and standardized logging system in `packages/backend` using NestJS\'s built-in logger interface with Winston. This system will allow fine-grained control over log verbosity via environment variables, targeting severity level, message source (context), and specific message categories (tags).

### 2.2. How Logging Works (Architecture Overview)

The enhanced logging system follows this flow:

1.  **Developer Code:** Services, controllers, gateways, etc., inject the standard NestJS `Logger` (`@nestjs/common`).
    *   Example: `private readonly logger = new Logger(MyService.name);`
2.  **Logging Calls:** Developers use standard logger methods (`this.logger.log()`, `.debug()`, `.warn()`, `.error()`). The `MyService.name` provided during instantiation is automatically passed as the `context`.
3.  **NestJS Core:** NestJS intercepts these calls.
4.  **Global Logger:** Because we configure our custom `WinstonLogger` globally in `main.ts` (`app = await NestFactory.create(AppModule, { logger: new WinstonLogger() })`), NestJS forwards the log message, level, and context to our `WinstonLogger` instance.
5.  **WinstonLogger (`winston.logger.ts`):**
    *   Receives the log details.
    *   Maps NestJS log levels (log, debug, warn, error, verbose, fatal) to Winston levels (info, debug, warn, error).
    *   Extracts tags from log messages in the format `[TAG_NAME]` at the beginning of the message.
6.  **Winston Core (`winston` library):**
    *   Applies a **custom filter**:
        *   Checks if the message level meets the minimum threshold defined by the `LOG_LEVEL` environment variable.
        *   **OR**, if the level is below `LOG_LEVEL`, checks if the message\'s `context` (e.g., \'MyService\') is included in the `DEBUG_LOG_CONTEXTS` environment variable.
        *   **OR**, if the level is below `LOG_LEVEL`, checks if the message contains a tag that\'s included in the `DEBUG_LOG_TAGS` environment variable.
        *   **AND**, checks if the message\'s `context` is not included in the `SUPPRESSED_LOG_CONTEXTS` environment variable.
        *   **AND**, checks if a message's tag is not included in the `SUPPRESSED_LOG_TAGS` environment variable.
        *   If none of these conditions are met (or a suppression condition IS met), the log message is discarded.
        *   If the message passes the filter, it\'s formatted (timestamp, level, context, tag, message).
7.  **Winston Transports:**
    *   **Console Transport:** Outputs the formatted log to the console.
    *   **DailyRotateFile Transport:** Writes the formatted log to a file in `packages/backend/src/debuglogs/`, rotating daily or by size.

### 2.3. Developer Usage Guide

#### Injecting the Logger

In any NestJS provider (Service, Controller, Gateway, etc.), import `Logger` from `@nestjs/common` and inject it, preferably providing the class name as the context:

```typescript
import { Injectable, Logger } from \'@nestjs/common\';

@Injectable()
export class MyService {
  // Pass the class name as the context for better log filtering
  private readonly logger = new Logger(MyService.name);

  doSomething() {
    this.logger.log(\'Starting operation...\');
    // ... logic ...
  }
}
```

#### Using Log Levels

Use the appropriate log level for your message:

*   `this.logger.error(message, stack?, context?)`: For serious errors that prevent normal operation. Include stack trace if available.
*   `this.logger.warn(message, context?)`: For potential issues or situations that might require attention but don\'t stop execution.
*   `this.logger.log(message, context?)`: **(Maps to INFO)** For general operational messages, milestones, or significant events.
*   `this.logger.debug(message, context?)`: For detailed information useful only during development or specific debugging sessions.
*   `this.logger.verbose(message, context?)`: **(Maps to DEBUG)** Even more detailed information, typically disabled unless needed.
*   `this.logger.fatal(message, context?)`: **(Maps to ERROR)** For critical errors forcing application shutdown.

**Key:** Choose levels wisely. Avoid excessive `debug` or `log` messages for routine operations unless necessary for specific tracing.

#### Using Message Tags

You can add tags to your log messages for more granular filtering:

```typescript
// Adding a tag at the start of a log message
this.logger.debug(\`[NET_WORTH_DEBUG] Calculated net worth: \${calculatedNetWorth}\`);
this.logger.error(\`[NET_WORTH_CRITICAL] Net worth calculation mismatch!\`);
```

Tags should follow a structured naming convention:

- Use uppercase letters and underscores.
- Group related logs under the same prefix (e.g., `NET_WORTH_*`, `DB_*`, `AUTH_*`).
- Use suffixes to indicate severity or purpose (e.g., `_DEBUG`, `_CRITICAL`, `_TRACE`).

Common tag patterns:

- `[FEATURE_DEBUG]`: Detailed debugging information.
- `[FEATURE_CRITICAL]`: Critical errors or issues.
- `[FEATURE_TRACE]`: Execution flow tracing.
- `[FEATURE_WATCH]`: Conditions to monitor.

### 2.4. Configuration (Environment Variables)

Logging verbosity is controlled via five environment variables (typically set in a `.env` file):

*   `LOG_LEVEL`:
    *   **Purpose:** Sets the default minimum severity level required for a log message to be processed.
    *   **Values:** `error`, `warn`, `info`, `debug`.
    *   **Default:** `info`.
    *   **Example:** `LOG_LEVEL=warn` means only `warn` and `error` messages will be logged by default.

*   `DEBUG_LOG_CONTEXTS`:
    *   **Purpose:** Allows specific `debug` (or `verbose`) level messages to bypass the `LOG_LEVEL` restriction if their source context matches.
    *   **Values:** A comma-separated list of context names or wildcard patterns (e.g., `GameService`, `Asset*`).
    *   **Default:** Empty string (`''`).
    *   **Example:** `DEBUG_LOG_CONTEXTS=GameService,AssetService`.

*   `DEBUG_LOG_TAGS`:
    *   **Purpose:** Allows specific `debug` (or `verbose`) level messages to bypass the `LOG_LEVEL` restriction if their message contains a matching tag.
    *   **Values:** A comma-separated list of tag names or wildcard patterns (e.g., `NET_WORTH_CRITICAL`, `DB_*`).
    *   **Default:** Empty string (`''`).
    *   **Example:** `DEBUG_LOG_TAGS=NET_WORTH_CRITICAL`.

*   `SUPPRESSED_LOG_CONTEXTS`:
    *   **Purpose:** Completely suppresses all log messages from specified contexts, regardless of their log level.
    *   **Values:** A comma-separated list of context names or wildcard patterns.
    *   **Default:** Empty string (`''`).
    *   **Example:** `SUPPRESSED_LOG_CONTEXTS=PlayerUpdateValidator`.

*   `SUPPRESSED_LOG_TAGS`:
    *   **Purpose:** Completely suppresses all log messages containing the specified tags, regardless of their level or context.
    *   **Values:** A comma-separated list of tag names or wildcard patterns.
    *   **Default:** Empty string (`''`).
    *   **Example:** `SUPPRESSED_LOG_TAGS=NET_WORTH_DEBUG`.

**Wildcards:** `*` matches multiple characters, `?` matches a single character.

#### Common Scenarios:

*   **Production:** `LOG_LEVEL=warn`, `DEBUG_LOG_CONTEXTS=`, `DEBUG_LOG_TAGS=`, `SUPPRESSED_LOG_CONTEXTS=PlayerUpdateValidator`, `SUPPRESSED_LOG_TAGS=`
*   **Staging/Testing:** `LOG_LEVEL=info`
*   **Debugging Specific Service(s):** `LOG_LEVEL=info`, `DEBUG_LOG_CONTEXTS=MyProblematicService,AnotherService`
*   **Debugging Specific Functionality:** `LOG_LEVEL=info`, `DEBUG_LOG_TAGS=NET_WORTH_CRITICAL`
*   **Full Development Debug:** `LOG_LEVEL=debug`
*   **Focused Debugging:** `LOG_LEVEL=info`, `DEBUG_LOG_CONTEXTS=GameService,GameGateway`, `DEBUG_LOG_TAGS=NET_WORTH_CRITICAL`, `SUPPRESSED_LOG_CONTEXTS=PlayerUpdateValidator`, `SUPPRESSED_LOG_TAGS=NET_WORTH_TRACE`

### 2.5. Message Tag Examples

```typescript
// Game Service Net Worth Calculations
this.logger.debug(\`[NET_WORTH_DEBUG] Calculated net worth: \${player.cash} + \${assetValue} + \${ridgeValue} - \${player.debt} = \${calculatedNetWorth}\`);
this.logger.error(\`[NET_WORTH_CRITICAL] Net worth calculation mismatch! \${calculatedNetWorth} vs \${doubleCheckNetWorth}\`);

// Database Transaction Tracking
this.logger.debug(\`[DB_TRACE] Starting transaction for player update: \${playerId}\`);
this.logger.warn(\`[DB_WATCH] Long-running transaction detected: \${duration}ms for operation: \${operation}\`);
```

Console output example:
`[Nest] 58522 - 04-May-2025 12:38:07 EDT     debug [GameService] [NET_WORTH_DEBUG] Calculated net worth: ...`

Filtering with grep:
`grep "NET_WORTH_CRITICAL" packages/backend/src/debuglogs/*.log`

### 2.6. Viewing and Analyzing Logs

#### Log Locations

1.  **Console Output:** Real-time filtered logs during development (`pnpm run dev`).
2.  **Log Files:** `packages/backend/src/debuglogs/`
    *   **Naming:** `Backend-DD-MMM-YYYY_HH_mm_ss.log`.
    *   **Rotation:** Automatic compression and retention (default: 3 days, max 20MB/file).

#### Log Format

`[Nest] <PID> - <Timestamp>     <LEVEL> [<Context>] [<Tag>] <Message> [Stack Trace (if error)]`

*   **`<PID>`:** Process ID.
*   **`<Timestamp>`:** Date and time (e.g., `03-May-2025 20:05:00 PDT`).
*   **`<LEVEL>`:** Severity (e.g., `INFO`, `DEBUG`). Color-coded in console.
*   **`<Context>`:** Source class name (e.g., `GameService`).
*   **`<Tag>`:** Optional category (e.g., `NET_WORTH_DEBUG`). Cyan in console.
*   **`<Message>`:** Log content.
*   **`[Stack Trace]`:** For `error` level logs.

#### Analysis Techniques

*   **Real-time Monitoring:** Watch console output.
*   **Viewing Log Files:** Use text editors, `tail -f`, or `grep`.
*   **Adjusting Verbosity:** Modify `.env` variables (`LOG_LEVEL`, `DEBUG_LOG_CONTEXTS`, `DEBUG_LOG_TAGS`, `SUPPRESSED_LOG_CONTEXTS`, `SUPPRESSED_LOG_TAGS`) and restart the server.

### 2.7. Filtering Implementation Details

The logging system applies filters in the following order of precedence:

1.  **Suppressed Contexts**: If a log\'s context matches any pattern in `SUPPRESSED_LOG_CONTEXTS`, it is dropped.
2.  **Suppressed Tags**: If a log\'s tag matches any pattern in `SUPPRESSED_LOG_TAGS`, it is dropped.
3.  **Log Level Filtering**: Logs must meet the configured `LOG_LEVEL` threshold.
4.  **Debug Contexts & Tags Overrides**: Even if a log doesn\'t meet the `LOG_LEVEL` threshold, it can pass if its context is in `DEBUG_LOG_CONTEXTS` or its tag is in `DEBUG_LOG_TAGS` (unless suppressed by rules 1 or 2).

### 2.8. Log Analysis and Configuration Tools

#### Log Scanning Script

A Python script (`scripts/scan_logs.py`) analyzes log files to generate configuration templates:

```bash
python3 scripts/scan_logs.py --output env --log-dir packages/backend/src/debuglogs > .env.logging
```

This script helps identify high-volume contexts/tags and provides sample configurations.

#### Using the Generated Config

Copy the generated `.env.logging` content to your main `.env` file (e.g., `packages/backend/.env`) and uncomment desired settings. Restart the server to apply.

## 3. Frontend Console Logging Strategy

The frontend logging strategy aims for controllability, standardization, and debuggability, allowing developers to adjust log verbosity and focus directly in the browser.

### 3.1. Goals

-   **Controllability:** Adjust log verbosity and focus via browser dev console.
-   **Standardization:** Consistent log levels, context, and informal tagging.
-   **Debuggability:** Enable detailed logs for specific components/services.
-   **Performance:** Reduce default log noise.
-   **Clarity:** Easy-to-read and filter logs.

### 3.2. Core Concepts

Inspired by backend logging:

-   **Log Levels:** `ERROR`, `WARN`, `INFO`, `DEBUG`.
-   **Context:** String identifying the log source (e.g., component/service name).
-   **Tags (Informal):** Keywords within the message string (e.g., `[WEBSOCKET]`, `[STATE_UPDATE]`). These are for visual scanning and browser devtools filtering, not programmatic filtering by the logger itself.
-   **Runtime Configuration:** Settings managed via `localStorage` and `window` functions for live changes.

### 3.3. Logger Implementation (`packages/frontend/src/utils/logger.ts`)

A central logger utility (`logger.ts`) provides core logging functionality.

#### Log Levels

-   `ERROR`: Critical errors impairing functionality.
-   `WARN`: Potential issues, minor errors not breaking the application.
-   `INFO`: Significant lifecycle events, user actions, major state transitions.
-   `DEBUG`: Detailed information for development/debugging.

Internal severity mapping: `error: 4, warn: 3, info: 2, debug: 1`.

#### Main Logger Function

A core `logger.log(level, context, message, ...payload)` function will:
1.  Retrieve current configuration from `localStorage`.
2.  Check if the `context` is in `suppressedContexts`. If so, abort.
3.  Compare the message\'s `level` with the configured global `logLevel`.
4.  Check if the message is `debug` level AND its `context` is in `debugContexts`.
5.  If the message should be logged (based on global level or specific debug context enablement), format it and output to the browser\'s console (e.g., `console.error`, `console.log`).
6.  Formatting includes: Timestamp (ISO), Level (uppercase), Context, Message, and additional payload.

Example: `[2023-10-27T10:30:00.123Z] [DEBUG] [SocketService] [WEBSOCKET_RECV] Received gameStateUpdate { ...payload }`

#### Convenience Methods

-   `logger.error(context, message, ...payload)`
-   `logger.warn(context, message, ...payload)`
-   `logger.info(context, message, ...payload)`
-   `logger.debug(context, message, ...payload)`

#### Tagging Convention (Informal)

-   Tags are part of the message string for visual aid and devtools filtering.
-   Format: `[TAG_NAME_UPPERCASE]`
-   Examples: `[API_CALL]`, `[RENDER]`, `[STATE_MUTATION]`, `[USER_ACTION]`.

### 3.4. Configuration and Control (via `localStorage` and `window`)

Settings persist in `localStorage`. Helper functions on `window` allow manipulation via the developer console.

#### `localStorage` Keys:

-   `logging_level`: (String) Minimum log level. Values: `"error"`, `"warn"`, `"info"`, `"debug"`. Default: `"info"`.
-   `logging_debug_contexts`: (String) Comma-separated contexts for which `DEBUG` messages are always shown. Default: `""`. Example: `"SocketService,ActionPanelComponent"`.
-   `logging_suppressed_contexts`: (String) Comma-separated contexts whose logs are never shown. Default: `""`. Example: `"NoisyComponent"`.

#### `window` Helper Functions:

-   `window.logConfig()`: Displays current configuration.
-   `window.setLogLevel(level: 'error'|'warn'|'info'|'debug')`: Sets global log level.
-   `window.enableDebugFor(...contexts: string[])`: Adds contexts to `logging_debug_contexts`.
-   `window.disableDebugFor(...contexts: string[])`: Removes contexts from `logging_debug_contexts`.
-   `window.suppressLogsFrom(...contexts: string[])`: Adds contexts to `logging_suppressed_contexts`.
-   `window.unsuppressLogsFrom(...contexts: string[])`: Removes contexts from `logging_suppressed_contexts`.
-   `window.resetLogConfig()`: Resets to defaults.

### 3.5. Standardization: Usage Guidelines

#### When to Log What:

-   **`logger.error(context, message, errorObject, ...data)`:** Uncaught exceptions, critical API/socket failures, UI/game flow breakers.
    -   Example: `logger.error('AuthService', '[API_CALL] Login failed', errorResponse);`
-   **`logger.warn(context, message, ...data)`:** Unexpected but non-breaking backend responses, non-critical API/socket issues, potential state inconsistencies.
    -   Example: `logger.warn('GameSetup', '[VALIDATION] Invalid player name', { receivedName });`
-   **`logger.info(context, message, ...data)`:** Major component lifecycles, significant user actions, socket status, high-level game flow.
    -   Example: `logger.info('ActionPanel', '[USER_ACTION] Player clicked End Turn');`
-   **`logger.debug(context, message, ...data)`:** Detailed socket payloads, fine-grained state changes, prop/state values for rendering diagnostics, complex function entry/exit.
    -   Example: `logger.debug('SocketService', '[WEBSOCKET_RECV] Game state update', { newState });`
    -   **Default State:** Most `debug` logs are hidden if `logLevel` is `info`; enable via `enableDebugFor('MyComponent')`.

#### Choosing Context Strings:

-   Be consistent.
-   Components: Component name (e.g., `'ActionPanel'`).
-   Services: Service name (e.g., `'SocketService'`).
-   Hooks: Hook name (e.g., `'usePlayerActivity'`).
-   Utilities: Descriptive name (e.g., `'gameStateReducer'`).

#### Payload:

-   Pass objects as separate arguments for interactive inspection in the browser console.
    -   Good: `logger.debug('MyComponent', 'State updated', { user, items });`
    -   Less Good: `logger.debug('MyComponent', \`State updated: user=\${JSON.stringify(user)}\`);`

### 3.6. Future Considerations (Optional Enhancements)

-   In-app log viewer/controller.
-   Log persistence (download/remote sending).
-   Advanced programmatic tag filtering.
-   Zustand middleware for automated store logging.

## 4. Unified Database Log Storage

The farming_game project now implements a unified database logging system that consolidates both backend and frontend logs into a single PostgreSQL table (`UnifiedLogEntry`). This enables powerful cross-system debugging and analysis capabilities that were not possible with the previous file-based approach.

### 4.1. Schema and Storage

The central component is the `UnifiedLogEntry` table with the following schema:

```prisma
model UnifiedLogEntry {
  id         String   @id @default(cuid())
  timestamp  DateTime @default(now())
  source     LogSource
  severity   LogSeverity
  context    String
  tags       String[] // Multiple tags supported!
  message    String
  userId     String?
  sessionId  String?
  gameId     String?
  metadata   Json?

  @@index([timestamp])
  @@index([source])
  @@index([context])
  @@index([gameId])
  @@index([userId])
  @@index([tags], type: GIN) // GIN index for array search
}

enum LogSource {
  backend
  frontend
}

enum LogSeverity {
  error
  warn
  info
  debug
}
```

The schema supports:
- **Multiple Tags**: Each log entry can have multiple tags for flexible categorization.
- **Flexible Metadata**: Structured data specific to each source (stack traces, browser info, etc.)
- **Efficient Querying**: Indexed fields for fast filtering and searching.

#### Schema Design Rationale

- **tags String[]**: Using a string array rather than a single concatenated string allows for more efficient filtering through PostgreSQL's GIN index, enabling queries like "show me all logs with tags that include 'NET_WORTH_DEBUG' OR 'DB_TRACE'".
  
- **metadata Json?**: The JSONB column provides flexibility for storing unstructured or semi-structured data specific to each log source:
  - For backend: stack traces, request details, object dumps
  - For frontend: browser info, component state, action payloads
  
- **Multiple Indexes**: While adding multiple indexes has a small write performance cost, it dramatically improves query performance for the most common access patterns used in troubleshooting.

#### Implementation Location

The schema is implemented in `prisma/schema.prisma` with appropriate migrations in the `prisma/migrations` directory.

### 4.2. Backend Integration

Backend logs are automatically written to both the database and fallback files:

- **DB Writing**: The `LogDbService` handles writing logs to the PostgreSQL database with appropriate error handling.
  
- **Rate Limiting**: A limit of 1000 logs/minute prevents database flooding in high-volume scenarios.
  - When the rate limit is exceeded, logs are automatically routed to file-only storage
  - A warning is logged when rate limiting is activated
  - The rate limit is configurable via environment variables
  
- **Fallback Mechanism**: If database writing fails, logs are still written to files.
  - Automatic retry logic with configurable backoff
  - Error events are tracked to avoid recursive logging errors
  - Health check mechanism to detect when database connectivity is restored
  
- **Data Scrubbing**: Sensitive information is automatically removed from logs before storage:
  - JWT tokens and authorization headers
  - API keys and access tokens
  - Passwords in URLs and request bodies
  - Credit card numbers and payment information
  - Social security numbers and personal identifiers
  - Email addresses in specific contexts
  - Database connection strings and credentials
  
#### Data Scrubber Implementation

The data scrubber is implemented in `packages/backend/src/logger/data-scrubber.ts`:

```typescript
// Example pattern from data scrubber
const DEFAULT_SCRUBBER_PATTERNS = [
  // JWT Bearer tokens
  { pattern: /(Bearer\s+)[A-Za-z0-9-_=]+\.[A-Za-z0-9-_=]+\.?[A-Za-z0-9-_.+/=]*/, replacement: '$1[REDACTED_JWT]' },
  // API keys
  { pattern: /([Aa][Pp][Ii][_-]?[Kk][Ee][Yy][=:]\s*["']?)[A-Za-z0-9]{16,}(["']?)/, replacement: '$1[REDACTED_API_KEY]$2' },
  // Database connection strings
  { pattern: /(mongodb(?:\+srv)?:\/\/(?:[^:]+:[^@]*)@)([^@]+)/, replacement: '$1[REDACTED_DB_CREDENTIALS]' },
  // More patterns...
];
```

The data scrubber can be enabled/disabled via the `ENABLE_LOG_SCRUBBING` environment variable (default: enabled) and patterns can be extended by adding to the pattern array.

#### Request-Scoped Logging

The backend logging implementation includes request-scoped logging capabilities:

- Each incoming HTTP request receives a unique request ID
- All logs generated during a request automatically include this ID in metadata
- Request metadata (path, method, client info) is attached to request-scoped logs
- Enable via the `REQUEST_SCOPED_LOGGING` environment variable (default: enabled)

### 4.3. Frontend Integration

The enhanced frontend logger (`packages/frontend/src/utils/enhanced-logger.ts`) provides advanced logging capabilities that integrate with the unified database logging system:

#### Core Features

- **Batched Sending**: Logs are queued and sent in batches to reduce network requests.
  - Default batch size: 5 logs
  - Default flush interval: 5 seconds
  - Automatic flush on critical errors
  - Manual flush capability for immediate sending
  
- **Session Tracking**: Each browser session gets a unique ID for correlation.
  - UUID generated on application startup
  - Persisted in localStorage to track the same session across page reloads
  - Available in all log entries for cross-request correlation
  
- **Browser Metadata**: Includes contextual information automatically with each log:
  - Current URL and route
  - Browser user agent
  - Device/screen information
  - Application version
  - Network connection status
  
- **Opt-In Control**: Users can opt in/out of remote logging via settings.
  - Configurable via Settings UI component
  - Defaults to disabled in production, enabled in development
  - Can be changed programmatically or via localStorage
  - User preference persisted across sessions
  
- **Client-side Throttling**: Prevents overwhelming the backend API.
  - Rate limiting with configurable thresholds
  - Back-off strategy for failed transmissions
  - Queue size limits with overflow handling
  - Priority system (errors get sent first)
  
- **Circular Reference Handling**: Special handling for cyclic objects in payloads.
  - Automatic detection and safe serialization
  - Preservation of object structure
  - Size limits for large objects
  - Configurable depth limits

#### Usage Example

```typescript
import { enhancedLogger } from '../utils/enhanced-logger';

// Basic usage
enhancedLogger.info('UserDashboard', 'User data loaded');

// With tags
enhancedLogger.info('UserDashboard', '[USER_ACTION] Settings updated', { 
  settings: updatedSettings 
});

// With metadata
enhancedLogger.error('AuthProvider', 'Login failed', {
  userId: attemptedUserId,
  error: error.message,
  metadata: {
    attempts: loginAttempts,
    ipAddress: clientIp,
    browserFingerprint: fingerprint
  }
});

// Force immediate sending
enhancedLogger.error('PaymentProcessor', '[CRITICAL] Payment failed', paymentData);
enhancedLogger.flushLogs(); // Send immediately without waiting for batch
```

#### Implementation Details

The enhanced frontend logger is built as a wrapper around the base frontend logger, adding remote transmission capabilities while maintaining all local console logging features:

1. **Request Batching**:
   ```typescript
   private logQueue: FrontendLogEntry[] = [];
   private flushInterval: number = 5000; // 5 seconds default
   private maxBatchSize: number = 5; // Default batch size
   
   private scheduleFlush() {
     if (this.flushTimeout) return;
     this.flushTimeout = window.setTimeout(() => this.flushLogs(), this.flushInterval);
   }
   ```

2. **Queue Management**:
   ```typescript
   public queueLog(level: LogLevel, context: string, message: string, ...payload: any[]): void {
     // Generate log entry with metadata
     const logEntry = this.createLogEntry(level, context, message, payload);
     
     // Add to queue
     this.logQueue.push(logEntry);
     
     // Auto-flush if we reach batch size or it's an error
     if (this.logQueue.length >= this.maxBatchSize || level === 'error') {
       this.flushLogs();
     } else {
       this.scheduleFlush();
     }
   }
   ```

3. **API Endpoint**:
   The `/api/logs/batch` endpoint in `LogsController` accepts batched logs and processes them through the unified logging system:
   
   ```typescript
   @Controller('logs')  // Global prefix 'api' is set in main.ts
   export class LogsController {
     @Post('batch')
     async createLogBatch(@Body() logBatch: FrontendLogBatchDto): Promise<BatchLogResult> {
       // Process and store logs via LogDbService
       // Includes validation and rate limiting
     }
   }
   ```

4. **Configuration**:
   ```typescript
   // Enable/disable remote logging
   enhancedLogger.setRemoteLoggingEnabled(true);
   
   // Configure batch settings
   enhancedLogger.setMaxBatchSize(10);
   enhancedLogger.setFlushInterval(10000); // 10 seconds
   ```

### 4.4. Configuration

Frontend remote logging can be configured via:

- `localStorage.setItem('logging_remote_enabled', 'true')` - Enable remote logging
- `enhancedLogger.setRemoteLoggingEnabled(true)` - Programmatically enable remote logging
- Configuration UI in the Settings Panel component

Backend database logging can be configured via environment variables:
```
# Enable/disable database logging
LOG_TO_DB=true

# Enable/disable data scrubbing
ENABLE_LOG_SCRUBBING=true
```

### 4.5. Verification and Testing

A comprehensive test interface is available in the GameLobby component to verify the logging system's functionality:
- Test API endpoint connectivity
- Force enable remote logging
- Send test logs both directly and via the enhanced logger
- Monitor queue state before and after flushing

### 4.6. Log Retention and Archival Strategy

To maintain database performance and keep storage costs reasonable while preserving necessary log history, the system implements an automated retention policy:

#### Planned Retention Implementation

- **Retention Window**: Logs older than a configurable period (default: 14 days) are automatically removed from the main table by a scheduled job.
  
- **Selective Retention By Severity**: Different retention periods can be configured based on log severity:
  - `error`: Longest retention (e.g., 30 days)
  - `warn`: Medium retention (e.g., 14 days)
  - `info`: Shorter retention (e.g., 7 days)
  - `debug`: Minimal retention (e.g., 3 days)

- **Archival Before Deletion**: Before purging, logs can be exported to cold storage:
  - Options include: S3 buckets, compressed files, or archive tables
  - Archives are organized by date ranges for easier recovery if needed
  
- **Schedule Configuration**: The retention job runs on a configurable schedule (default: daily at 3 AM server time) using node-cron.

#### Field Size Management

- **Message Truncation**: Very large message fields (>10,000 characters) are automatically truncated with an indicator.
  
- **Metadata Compression**: Large metadata objects can be compressed before storage using built-in PostgreSQL capabilities.
  
- **Selective Debug Data**: In production, detailed debug data in metadata may be automatically summarized or removed.

#### Implementation Details

The retention job implementation will be located in `packages/backend/src/logger/log-retention.service.ts` and scheduled through the application's task scheduling system.

### 4.7. MCP Server Integration (Planned)

Future implementation will expose log querying capabilities via MCP (Model Context Protocol) server endpoints:

- **Endpoint: `/mcp/logs/query`**
  - Supports filtering by: timestamp range, source, severity, context, tags, user/session/game IDs
  - Pagination support for large result sets
  - Sorting options (default: reverse chronological)
  
- **Advanced Filtering**
  - Full-text search across message content
  - Tag-based filtering with AND/OR operators
  - Metadata field search for structured data

- **Access Controls**
  - Role-based permissions for log access
  - Audit trail for sensitive log queries
  
The MCP integration will enable AI assistance tools to help with troubleshooting by retrieving context-relevant logs across both frontend and backend systems.

### 4.8. Admin Interface (Planned)

A web-based admin interface is planned to provide the following capabilities:

- **Real-time Log Viewer**
  - Live streaming of new log entries
  - Advanced filtering and search
  - Syntax highlighting for structured data
  
- **Log Analysis Tools**
  - Error frequency visualization
  - Trend analysis for error rates
  - User session correlation
  
- **Configuration Management**
  - Adjust retention policies
  - Configure log levels and filters
  - Manage alert thresholds

- **Export and Reporting**
  - CSV/JSON export of filtered logs
  - Automated regular reports
  - Integration with monitoring dashboards

### 4.9. Pending Enhancements

- Automated log analysis for pattern detection
- Machine learning for anomaly detection
- Integration with external monitoring systems
- Enhanced data visualization and dashboards

## 5. General Logging Principles

-   **Consistency:** Adhere to the defined levels, context naming, and tagging conventions across both frontend and backend.
-   **Purposeful Logging:** Log information that is genuinely useful for debugging, monitoring, or understanding system behavior. Avoid excessive or trivial logs.
-   **Context is Key:** Always provide meaningful context to quickly identify the source and relevance of a log message.
-   **Performance Awareness:** While detailed logging is invaluable for debugging, be mindful of its performance impact, especially in production or high-frequency code paths. Utilize the provided configuration mechanisms to manage verbosity.
-   **Security:** Do not log sensitive information (passwords, API keys, personal data). The data scrubbing system will attempt to remove sensitive data, but it's best practice to avoid logging it in the first place.
-   **Structured Data:** Use the metadata field for structured data rather than embedding JSON strings in the message field.
-   **Batch Where Appropriate:** For high-volume logging scenarios, consider using batched writes to reduce overhead.
-   **Documentation:** Keep this document (`docs/logging-approach.md`) updated as logging strategies or tools evolve.

### Best Practices for Log Usage

#### Backend Logging
```typescript
// Error handling with comprehensive context
try {
  // Operation that might fail
} catch (error) {
  logger.error(
    'ServiceName', 
    `Operation failed: ${error instanceof Error ? error.message : String(error)}`, 
    { error, operationId, additionalContext } // Structured data goes in metadata
  );
}

// Critical path entry/exit logging
logger.info('ServiceName', '[TAG_OPERATION_START] Starting critical operation', { operationId });
// ...operation code...
logger.info('ServiceName', '[TAG_OPERATION_END] Completed critical operation', { operationId, result: 'success', duration });
```

#### Frontend Logging
```typescript
// Using the enhanced logger for remote-capable logging
import { enhancedLogger } from '../utils/enhanced-logger';

// Component lifecycle events
enhancedLogger.info('ComponentName', 'Component mounted', { props: { id, name } });

// User interaction events with tags
enhancedLogger.info('ComponentName', '[USER_ACTION] Button clicked', { buttonId, currentState });

// Error logging with structured data
try {
  // Operation that might fail
} catch (error) {
  enhancedLogger.error('ComponentName', `Error during operation: ${error.message}`, {
    error,
    componentState,
    operationName
  });
}
```

By following these guidelines, we can maintain a clear, useful, and manageable logging system throughout the `farming_game` project.

## 7. Log Retention & Cleanup System

To prevent unlimited growth of logs and maintain database performance, the farming game incorporates an automated Log Retention & Cleanup system.

### 7.1. Overview

The Log Retention & Cleanup system implements an automated mechanism to prevent unlimited growth of the unified logging database. This system ensures the database remains performant while preserving valuable historical data through configurable retention policies and archiving capabilities.

### 7.2. Configuration

Log retention is configured through environment variables:

```
# Basic retention settings
LOG_RETENTION_ENABLED=true
LOG_RETENTION_SCHEDULE="0 2 * * *"  # Default: 2 AM daily
LOG_RETENTION_BATCH_SIZE=1000
LOG_RETENTION_MAX_RUNTIME_SECONDS=3600  # 1 hour

# Default retention periods in days
LOG_RETENTION_ERROR_DAYS=90
LOG_RETENTION_WARN_DAYS=60
LOG_RETENTION_INFO_DAYS=30
LOG_RETENTION_DEBUG_DAYS=7

# Context-specific retention overrides (JSON format)
LOG_RETENTION_CONTEXT_OVERRIDES={"GameService":{"error":180,"warn":90},"PaymentService":{"error":365}}

# Archiving settings
LOG_ARCHIVE_ENABLED=true
LOG_ARCHIVE_COMPRESSION_LEVEL=6

# Storage settings
LOG_ARCHIVE_STORAGE_TYPE=filesystem  # or 's3'
LOG_ARCHIVE_PATH=/var/log/farming_game/archives  # For filesystem storage

# S3 settings (if using S3 storage)
LOG_ARCHIVE_S3_BUCKET=your-log-bucket-name
LOG_ARCHIVE_S3_PREFIX=log-archives/
LOG_ARCHIVE_S3_REGION=us-east-1
LOG_ARCHIVE_S3_ACCESS_KEY=YOUR_ACCESS_KEY
LOG_ARCHIVE_S3_SECRET_KEY=YOUR_SECRET_KEY

# Field size management
LOG_FIELD_SIZE_MANAGEMENT_ENABLED=true
LOG_MAX_MESSAGE_SIZE=10000
LOG_MAX_METADATA_SIZE=32000
LOG_TRUNCATE_OVERSIZED=true
LOG_FIELD_SIZE_WARNINGS=true

# Reporting
LOG_RETENTION_REPORT_PATH=/var/log/farming_game/reports
```

### 7.3. Implementation Details

The system is implemented in three phases, all of which are now completed:

1. **Phase 1: Basic Retention**
   - Time-based log deletion with configurable retention periods per severity level
   - Scheduled operations using node-cron
   - Batched deletion to minimize database impact

2. **Phase 2: Archiving Capability**
   - Pre-deletion archiving to compressed JSONL files
   - Filesystem storage provider with efficient compression
   - Archive organization by date, with metadata
   - Integrated into retention workflow to preserve historical data

3. **Phase 3: Advanced Features** (Completed as of May 2025)
   - Context-based retention policies for service-specific retention periods
   - S3 storage provider for cloud-based archival
   - Enhanced reporting and job monitoring
   - Field size management to prevent oversized log entries

### 7.4. Key Features

#### 7.4.1. Context-based Retention Policies

The system supports different retention periods based on the log context (service/component):

```typescript
// Example configuration with context-specific retention
const config = {
  retention: {
    defaultRetentionDays: {
      error: 90, // Default retention for errors
      warn: 60,
      info: 30,
      debug: 7
    },
    contextOverrides: {
      // GameService errors kept for 6 months instead of 90 days
      'GameService': {
        error: 180,
        warn: 90
      },
      // PaymentService errors kept for a year
      'PaymentService': {
        error: 365
      }
    }
  }
};
```

#### 7.4.2. Context-Based Retention Policies

The system now supports fine-grained retention control based on service context:

- Configure different retention periods for specific services/modules
- Examples:
  - Keep GameService error logs for 6 months
  - Keep PaymentService error logs for 12 months
  - Keep general logs for standard periods (30-90 days)
- Retention periods are fully configurable via code or environment variables

#### 7.4.3. Archiving and Storage Options

Logs are archived before deletion to preserve historical data. Two storage options are supported:

1. **Filesystem Storage**: Archives logs to the local filesystem
   - Organized by date (YYYY/MM/DD)
   - Compressed with gzip for efficient storage
   - Includes metadata files for easy searching

2. **S3 Storage**: Archives logs to S3-compatible object storage (Phase 3 feature)
   - Supports AWS S3 and other S3-compatible services
   - Configurable bucket, prefix, and region
   - Supports custom endpoints for compatibility with services like MinIO
   - Stream-based processing for efficient handling of large log volumes
   - Automatic compression and metadata generation

#### 7.4.4. Field Size Management (Phase 3 feature)

Automatically handles oversized log entries:

- Truncates large message fields to configurable limits
- Summarizes large metadata objects instead of storing them in full
- Logs warnings about oversized entries for monitoring
- Configurable size limits for different field types

#### 7.4.5. Reporting and Monitoring (Phase 3 feature)

The system tracks statistics about retention operations:

- Number of logs archived and deleted
- Space reclaimed and archive sizes
- Duration of retention jobs
- Breakdown by severity and context
- Historical reporting with trend analysis
- Job performance metrics for monitoring

For detailed information about the Log Retention & Cleanup system, see the [PRD document](../prd-active/PRD-log-retention-and-cleanup.md) and implementation in `/packages/backend/log-retention/`. For guidance on using the Phase 3 features, see the [README-phase3.md](/home/saberone/code/farming_game/packages/backend/log-retention/README-phase3.md) file. 