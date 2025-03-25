# Server Architecture Documentation

## Overview

The server component of the Digital Marketplace application is a critical part of the backend infrastructure, providing a type-safe, custom HTTP server framework built on top of Express. Located in `/src/back-end/lib/server`, this module implements a comprehensive system for handling HTTP requests, managing sessions, processing different types of request bodies, and producing standardized responses.

The architecture is designed with the following key principles:

1. **Type Safety** - Extensive use of TypeScript generics to ensure type safety across the request/response cycle
2. **Separation of Concerns** - Clear separation between request parsing, validation, and response generation
3. **Flexibility** - Support for different types of request/response bodies (JSON, HTML, Files, Text)
4. **Immutability** - All state is treated as immutable to prevent unexpected mutations
5. **Extensibility** - The framework is designed to be extended with custom behaviors via hooks and adapters

## Core Components

The server framework consists of two main files:

1. **`index.ts`** - Defines the core server types, interfaces, and utilities
2. **`adapters.ts`** - Implements the adapter pattern to connect the framework to Express

### Key Abstractions

#### Request and Response Modeling

The server models HTTP requests and responses using generic interfaces:

```typescript
export interface Request<Body, Session> {
  readonly id: string;            // Unique request ID
  readonly path: string;          // Request path
  readonly headers: IncomingHttpHeaders;  // HTTP headers
  readonly logger: DomainLogger;  // Request-specific logger
  readonly method: ServerHttpMethod;  // HTTP method
  readonly session: Session;      // User session
  readonly params: Record<string, string>;  // Route parameters
  readonly query: Record<string, string>;   // Query parameters
  readonly body: Body;            // Request body
}

export interface Response<Body, Session> {
  readonly code: number;          // HTTP status code
  readonly headers: OutgoingHttpHeaders;  // Response headers
  readonly session: Session;      // Updated session
  readonly body: Body;            // Response body
}
```

#### Request Body Types

The server supports multiple request body types, modeled as Algebraic Data Types (ADTs):

1. **JsonRequestBody** - For JSON data
2. **FileRequestBody** - For file uploads
3. **TextRequestBody** - For plain text

#### Response Body Types

Similarly, the server supports multiple response body types:

1. **JsonResponseBody** - For JSON responses
2. **HtmlResponseBody** - For HTML responses
3. **FileResponseBody** - For file downloads
4. **TextResponseBody** - For plain text responses
5. **ErrorResponseBody** - For standardized error responses

#### Route Handlers

Handlers process requests and produce responses:

```typescript
export interface Handler<
  IncomingReqBody,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ResBody,
  Session
> {
  readonly parseRequestBody: ParseRequestBody<
    IncomingReqBody,
    ParsedReqBody,
    Session
  >;
  readonly validateRequestBody: ValidateRequestBody<
    ParsedReqBody,
    ValidatedReqBody,
    ReqBodyErrors,
    Session
  >;
  readonly respond: Respond<
    Validation<ValidatedReqBody, ReqBodyErrors>,
    ResBody,
    Session
  >;
}
```

A handler is responsible for:
1. Parsing the request body
2. Validating the parsed body
3. Responding to the request

#### Routes

Routes connect HTTP methods and paths to handlers:

```typescript
export interface Route<
  IncomingReqBody,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ResBody,
  HookState,
  Session
> {
  readonly method: ServerHttpMethod;
  readonly path: string;
  readonly handler: Handler<...>;
  readonly hook?: RouteHook<...>;
}
```

#### Hooks

Hooks provide a way to execute code before and after request processing:

```typescript
export interface RouteHook<
  IncomingReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ResBody,
  State,
  Session
> {
  readonly before: BeforeHook<IncomingReqBody, State, Session>;
  readonly after?: AfterHook<
    Validation<ValidatedReqBody, ReqBodyErrors>,
    ResBody,
    State,
    Session
  >;
}
```

## Express Adapter

The Express adapter (`adapters.ts`) connects the abstract server framework to Express.js. It:

1. Creates an Express application
2. Sets up middleware for parsing JSON, cookies, and handling CORS
3. Maps framework routes to Express routes
4. Handles session management via cookies
5. Processes multipart file uploads
6. Executes route hooks before and after request handling

```typescript
export function express<...>(): ExpressAdapter<...> {
  return ({ router, sessionIdToSession, sessionToSessionId, ... }) => {
    // Setup Express app
    const app = expressLib();

    // Add middleware
    app.use(bodyParser.json({ type: "application/json" }));
    app.use(cookieParser(COOKIE_SECRET));
    app.use(corsLib(corsOptionDelegate));

    // Mount routes
    router.forEach((route) => {
      app.all(route.path, makeExpressRequestHandler(route));
    });

    return app;
  };
}
```

The core of the adapter is the `makeExpressRequestHandler` function, which:

1. Creates an async handler function
2. Checks if the HTTP method matches the route
3. Retrieves the session from cookies
4. Parses the request body (JSON or multipart)
5. Processes the request through the handler's parse, validate, and respond stages
6. Cleans up temporary files
7. Executes after hooks
8. Sends the response to the client

## Session Management

The server implements a straightforward session management system:

1. **SessionId** - A unique identifier for user sessions
2. **SessionIdToSession** - A function to convert session IDs to session objects
3. **SessionToSessionId** - A function to convert session objects to session IDs

Sessions are typically stored in cookies using the name "sid" and are signed using a secret.

## Request Processing Flow

When a request is received:

1. **Method Checking** - The HTTP method is checked against the route
2. **Session Creation** - The session is retrieved or created from cookies
3. **Body Parsing** - The request body is parsed based on content type
4. **Before Hook** - The before hook is executed if present
5. **Request Parsing** - The request is parsed using the handler's parseRequestBody function
6. **Request Validation** - The parsed request is validated using the handler's validateRequestBody function
7. **Response Generation** - The response is generated using the handler's respond function
8. **Cleanup** - Temporary files are deleted
9. **After Hook** - The after hook is executed if present
10. **Response Sending** - The response is sent to the client

## File Handling

The server includes robust support for file uploads and downloads:

### File Uploads

File uploads are processed using the multiparty library and stored in a temporary directory. The framework supports:

- File size limits
- Metadata extraction from form fields
- Content type detection
- Automatic cleanup of temporary files

### File Downloads

File downloads are supported with features like:

- Content type detection
- Content encoding (compression)
- Content disposition (inline vs. attachment)
- Support for compressed files (Gzip, Brotli)

## Integration with Resources

The server framework is designed to work with the resources module, which implements RESTful API endpoints. Resource routes are created using the `makeRouter` function and added to the server:

```typescript
// Create routers from resources
const crudRoutes = flow([
  map((resource: BasicCrudResource) => {
    return makeRouter(resource)(connection);
  }),
  flatten,
  // Additional processing...
])(resources);
```

## Swagger Integration

The server includes optional Swagger integration for API documentation. When enabled, Swagger UI is available at the configured path:

```typescript
if (SWAGGER_ENABLE) {
  app.use(SWAGGER_UI_PATH, swaggerUI.serve, swaggerUI.setup(specs));
}
```

## Best Practices

### Creating Routes

When creating new routes:

1. Define the request and response body types
2. Implement a handler with parse, validate, and respond functions
3. Add the route to the appropriate router

### Error Handling

The framework provides standardized error responses:

```typescript
export function makeErrorResponseBody(error: Error): ErrorResponseBody {
  return {
    tag: "error",
    value: {
      message: error.message,
      stack: error.stack,
      raw: error.toString()
    }
  };
}
```

### Validation

Request validation should:

1. Check all required fields
2. Validate field formats and constraints
3. Return appropriate error messages

Example:
```typescript
async function validateUser(user: User): Promise<Validation<User, string[]>> {
  if (!user.name) {
    return invalid(["Name is required"]);
  }
  return valid(user);
}
```

## Security Considerations

1. **CORS** - The server implements CORS protection to limit API access to specific domains
2. **Cookie Security** - Cookies are signed to prevent tampering
3. **Input Validation** - All user input is validated before processing
4. **File Size Limits** - File uploads have size limits to prevent DoS attacks

## Performance Optimization

1. **Compressed Responses** - Support for Gzip and Brotli compression
2. **Request ID Tracking** - Each request has a unique ID for tracing and debugging
3. **Efficient File Handling** - Files are streamed rather than loaded entirely into memory

## Conclusion

The server framework provides a robust, type-safe foundation for the Digital Marketplace backend. Its flexible design allows for easy extension with new routes and resources, while ensuring type safety throughout the request/response cycle. The use of Express as the underlying HTTP server combines the benefits of a popular, well-tested server with a custom framework tailored to the application's needs.
