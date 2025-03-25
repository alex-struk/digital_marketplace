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

## Functional Core, Imperative Shell

The server framework is heavily influenced by the "Functional Core, Imperative Shell" (FCIS) architectural pattern, as first described by Gary Bernhardt. This pattern separates systems into two distinct layers:

### Functional Core

The "functional core" consists of pure business logic with these characteristics:
- **Pure Functions** - Functions that always return the same output for the same input without side effects
- **Immutability** - Data is never mutated in-place; instead, new data structures are created with changes
- **No Side Effects** - No I/O operations, database calls, network requests, or other external interactions
- **Deterministic** - Behavior is predictable and reproducible, making testing straightforward

In the Digital Marketplace, the functional core includes:

#### 1. Validation Functions

The server uses pure validation functions that take inputs and return validation results without side effects:

```typescript
// In shared/lib/validation/index.ts
export function validateGenericString(
  value: string,
  name: string,
  min = 1,
  max = 100,
  characters = "characters"
): Validation<string> {
  if (value.length < min || value.length > max) {
    return invalid([
      `${name} must be between ${min} and ${max} ${characters} long.`
    ]);
  } else {
    return valid(value);
  }
}

// In shared/lib/validation/organization.ts
export function validateLegalName(raw: string): Validation<string> {
  return validateGenericString(raw, "Legal Name");
}
```

#### 2. Request Transformation

Pure functions for transforming request data between different formats:

```typescript
// In back-end/lib/server/index.ts
export function composeTransformRequest<RBA, RBB, RBC, Session>(
  a: TransformRequest<RBA, RBB, Session>,
  b: TransformRequest<RBB, RBC, Session>
): TransformRequest<RBA, RBC, Session> {
  return async (request) => {
    const bodyA = await a(request);
    return await b({
      ...request,
      body: bodyA
    });
  };
}
```

#### 3. Response Mapping

Functions that transform responses in a pure manner:

```typescript
export function mapRespond<ReqB, ResBA, ResBB, Session>(
  respond: Respond<ReqB, ResBA, Session>,
  fn: (response: Response<ResBA, Session>) => Response<ResBB, Session>
): Respond<ReqB, ResBB, Session> {
  return async (request) => {
    const response = await respond(request);
    return fn(response);
  };
}
```

#### 4. Domain Models and Types

Immutable data types that represent the domain:

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
```

### Imperative Shell

The "imperative shell" is a thin layer surrounding the functional core that handles:
- **Side Effects** - All I/O operations (file system, network, database)
- **External Systems** - Interactions with third-party services and APIs
- **State Management** - Maintaining and persisting application state
- **Coordination** - Orchestrating the flow of data between the functional core and external world

In this server framework, the imperative shell is primarily implemented through:

#### 1. Express Adapter

The Express adapter in `adapters.ts` is the primary imperative shell component, handling all HTTP-specific interactions:

```typescript
export function express<
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  HookState,
  Session,
  FileUploadMetaData
>(): ExpressAdapter<...> {
  return ({ router, sessionIdToSession, sessionToSessionId, ... }) => {
    // Setup Express app - Imperative code
    const app = expressLib();

    // Add middleware - Imperative interactions with Express
    app.use(bodyParser.json({ type: "application/json" }));
    app.use(cookieParser(COOKIE_SECRET));
    app.use(corsLib(corsOptionDelegate));

    // Mount routes - Imperative registration of routes
    router.forEach((route) => {
      app.all(route.path, makeExpressRequestHandler(route));
    });

    return app;
  };
}
```

#### 2. Request Processing Flow

The `makeExpressRequestHandler` function implements the imperative shell's coordination logic:

```typescript
function makeExpressRequestHandler(...): expressLib.RequestHandler {
  return asyncHandler(async (expressReq, expressRes, next) => {
    // Side effects: HTTP method checking
    const method = parseServerHttpMethod(expressReq.method) || ServerHttpMethod.Any;
    if (method !== route.method) {
      next();
      return;
    }

    // Side effects: Session management
    const sessionId = parseSessionId(expressReq.signedCookies[SESSION_COOKIE_NAME]);
    const session = await sessionIdToSession(sessionId);

    // Side effects: Request body parsing
    let body: ExpressRequestBodies<FileUploadMetaData> = makeJsonRequestBody(null);
    if (...) {
      body = makeJsonRequestBody(expressReq.body);
    } else if (...) {
      body = await parseMultipartRequest(..., expressReq);
    }

    // Create initial request object (immutable)
    const initialRequest = {
      id: requestId,
      path: expressReq.path,
      method, session, headers, /* ... */
      body
    };

    // Execute core business logic (functional core)
    const hookState = route.hook ? await route.hook.before(initialRequest) : null;
    const parsedRequest = { ...initialRequest, body: await route.handler.parseRequestBody(initialRequest) };
    const validatedRequest = { ...parsedRequest, body: await route.handler.validateRequestBody(parsedRequest) };
    const response = await route.handler.respond(validatedRequest);

    // Side effects: File cleanup
    if (body.tag === "file" && existsSync(body.value.path)) {
      unlinkSync(body.value.path);
    }

    // Side effects: After hook execution
    if (route.hook && route.hook.after) {
      await route.hook.after(hookState, validatedRequest, response);
    }

    // Side effects: HTTP response sending
    respond(response, expressRes);
  });
}
```

#### 3. File Handling

File operations are implemented in the imperative shell:

```typescript
// Parsing multipart requests with file uploads (side effects)
async function parseMultipartRequest(
  maxSize: number,
  parseMetadata: (data: any) => FileUploadMetaData,
  req: expressLib.Request
): Promise<FileRequestBody<FileUploadMetaData>> {
  return new Promise((resolve, reject) => {
    // Side-effect: File system operations
    const form = new multiparty.Form({
      maxFilesSize: maxSize,
      uploadDir: tmpdir()
    });

    form.parse(req, (err, fields, files) => {
      // Handle file upload side effects
      // ...
    });
  });
}
```

### Benefits of FCIS in the Server Framework

1. **Testability** - The functional core can be tested without mocks or complex setup
2. **Reasoning** - The separation makes it easier to reason about system behavior
3. **Maintainability** - Changes to business logic don't require changes to I/O handling
4. **Robustness** - Errors in the imperative shell don't corrupt the functional core
5. **Flexibility** - The framework can adapt to different HTTP libraries by changing only the shell

### Implementation in the Server Framework

The server framework implements the FCIS pattern through these mechanisms:

#### Handler Pattern

The `Handler` interface clearly separates parsing, validation, and response generation:

```typescript
export interface Handler<
  IncomingReqBody,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ResBody,
  Session
> {
  readonly parseRequestBody: ParseRequestBody<...>;  // Pure transformation function
  readonly validateRequestBody: ValidateRequestBody<...>;  // Pure validation function
  readonly respond: Respond<...>;  // May contain impure code when needed
}
```

#### Example Route Implementation

A complete example of a route handler showing the FCIS separation:

```typescript
// Handler with functional core validation logic
const createUserHandler: Handler<JsonRequestBody, any, UserData, string[], JsonResponseBody<User>, Session> = {
  // Pure function to parse request body
  parseRequestBody: async (request) => {
    if (request.body.tag === 'json') {
      return request.body.value;
    }
    throw new Error('Invalid request body');
  },

  // Pure function to validate parsed data
  validateRequestBody: async (request) => {
    const body = request.body;
    const errors: string[] = [];

    if (!body.name) errors.push('Name is required');
    if (!body.email) errors.push('Email is required');

    if (errors.length > 0) {
      return invalid(errors);
    }

    return valid({
      name: body.name,
      email: body.email
    });
  },

  // Function that bridges functional core with imperative operations
  respond: wrapRespond({
    valid: async (request) => {
      const userData = request.body;

      // Side effect: Database operation (imperative shell)
      const user = await db.users.create(userData);

      // Pure response creation (functional core)
      return {
        code: 201,
        headers: {},
        session: request.session,
        body: makeJsonResponseBody(user)
      };
    },
    invalid: async (request) => {
      // Pure error response creation (functional core)
      return {
        code: 400,
        headers: {},
        session: request.session,
        body: makeJsonResponseBody({ errors: request.body })
      };
    }
  })
};
```

While not a perfect implementation of FCIS, the framework makes significant strides toward this ideal, creating a codebase that balances functional purity with practical imperative needs.

## Conclusion

The server framework provides a robust, type-safe foundation for the Digital Marketplace backend. Its flexible design allows for easy extension with new routes and resources, while ensuring type safety throughout the request/response cycle. The use of Express as the underlying HTTP server combines the benefits of a popular, well-tested server with a custom framework tailored to the application's needs.
