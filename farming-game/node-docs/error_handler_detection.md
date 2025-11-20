# Enhanced External Error Handler Detection

The log audit tool has been enhanced to detect cases where error handling is delegated to external functions.
This feature prevents false positives when analyzing catch blocks that use centralized error handling patterns.

## How It Works

When analyzing catch blocks, the tool now looks for patterns where the caught error is passed to another function
that is likely to perform proper error logging. This includes:

1. Method calls that match common error handling naming patterns
2. Object method calls that indicate error handling delegation
3. Regex patterns for more complex delegation scenarios

## Configuration

The error handler detection can be configured through a JSON configuration file.

### Default Configuration File

A default configuration file is provided at:
```
scripts/log_audit/config/error_handlers.json
```

### Configuration Format

The configuration file uses the following format:

```json
{
    "patterns": {
        "method_calls": [
            "handleError",
            "processError",
            "logError"
        ],
        "object_methods": [
            "this.handleError",
            "errorHandler.handle",
            "logger.error"
        ],
        "regex": [
            "\\w+\\.handle(?:Error|Exception)"
        ]
    },
    "match_confidence": {
        "high": [
            "handleError",
            "logger.error"
        ],
        "medium": [
            "processError"
        ],
        "low": []
    }
}
```

#### Pattern Categories

- `method_calls`: Simple function names that are likely to handle errors (e.g., `handleError(error)`)
- `object_methods`: Object method calls that handle errors (e.g., `this.handleError(error)` or `errorHandler.process(error)`)
- `regex`: Regular expression patterns for more complex matches

#### Match Confidence

- `high`: Patterns that are very likely to properly log errors
- `medium`: Patterns that might log errors in most cases
- `low`: Patterns that sometimes log errors

## Command-Line Usage

You can specify a custom error handlers configuration file using the `--error-handlers-config` flag:

```bash
python scripts/log_audit/log_audit.py --error-handlers-config=path/to/config.json
```

## Example Detection

The tool can detect error delegation patterns like:

```javascript
try {
    // Some operation
} catch (error) {
    this.handleError(error, context);  // ✓ Detected as proper error handling
}
```

```javascript
try {
    // Some operation
} catch (error) {
    errorHandler.process(error);  // ✓ Detected as proper error handling
}
```

```javascript
try {
    // Some operation
} catch (error) {
    await logAndThrow(error);  // ✓ Detected as proper error handling
}
```

## Report Output

When a delegated error handling pattern is detected, it will be noted in the report:

```
### 1. path/to/file.ts:25
Error parameter: `error`

**Error Delegation Detected**: `this.handleError(error)`

```

This indicates that the catch block was recognized as properly handling the error through delegation.