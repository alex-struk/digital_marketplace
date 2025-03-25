# Backend Logging System

The logging system in the Digital Marketplace backend provides a flexible and consistent way to log application events at various levels of importance. This document outlines how the logging system works and how to use it effectively in your code.

## Architecture

The logging system is built around the following components:

1. **Log Levels**: The system supports multiple logging levels (`debug`, `info`, `warn`, `error`, `none`) to categorize the importance of log messages.

2. **Adapters**: Adapters abstract the actual logging implementation. The default is a console adapter, but the architecture allows for additional adapters (e.g., file logging, remote logging services).

3. **Domain Loggers**: To provide context to log messages, the system uses domain loggers that automatically tag logs with a domain identifier.

4. **Hooks**: The system includes hooks for HTTP requests to automatically log request/response information.

## Log Levels

The available log levels, in order of verbosity (most to least):

| Level   | Description                                          | Color  |
|---------|------------------------------------------------------|--------|
| `debug` | Detailed debugging information                       | Blue   |
| `info`  | General informational messages                       | Green  |
| `warn`  | Warning conditions                                   | Yellow |
| `error` | Error conditions                                     | Red    |
| `none`  | No logging                                           | -      |

The system will only output log messages at or below the configured log level. For example, if `LOG_LEVEL` is set to `info`, then only `info`, `warn`, and `error` messages will be displayed, while `debug` messages will be suppressed.

## Configuration

The log level is determined by the `LOG_LEVEL` environment variable, with a default that depends on the environment:
- Development: `debug` (most verbose)
- Production/others: `info`

Additionally, the `LOG_MEM_USAGE` environment variable (boolean) can be set to enable memory usage logging.

## Usage

### Creating a Domain Logger

The recommended way to use the logging system is to create a domain logger:

```typescript
import { makeDomainLogger } from "back-end/lib/logger";
import { console as consoleAdapter } from "back-end/lib/logger/adapters";

// Create a domain logger for a specific component
const logger = makeDomainLogger(consoleAdapter, "my-component");

// Now use it to log messages
logger.info("Application started");
logger.debug("Processing payload", { id: "123", size: 1024 });
logger.warn("Resource usage is high");
logger.error("Failed to connect to database", { error: "Connection refused" });
```

### Additional Data

Each log method accepts an optional data object that will be serialized and included in the log message:

```typescript
logger.info("User logged in", { userId: "user-123", method: "oauth" });
```

This will output something like:

```
[my-component] User logged in userId="user-123" method="oauth"
```

### Request Logging

The system automatically logs HTTP requests and responses when using the logger hook:

```typescript
import loggerHook from "back-end/lib/hooks/logger";

// Add the logger hook to your server/router configuration
router.hook(loggerHook);
```

This will log:
- Request information (`-> METHOD /path`)
- Response information (`<- STATUS_CODE time_ms`)
- Memory usage (if enabled)

## Adapters

The current implementation includes a console adapter that outputs colorized log messages to the console. The adapter system is extensible, allowing for additional adapters to be created for different output destinations.

## Performance Considerations

- For inactive log levels (e.g., `debug` logs when `LOG_LEVEL` is set to `info`), the system uses a no-op function to minimize performance impact.
- Consider the verbosity of logs in production environments to avoid unnecessary I/O overhead.
- Use the domain logger to provide context rather than repeating the same information in every log message.

## Memory Usage Logging

When `LOG_MEM_USAGE` is enabled, the system will include memory usage statistics with each response log:

```
memory usage rss="123MB" heapTotal="45MB" heapUsed="40MB" external="5MB" arrayBuffers="2MB"
```

This is helpful for monitoring the application's memory consumption over time.
