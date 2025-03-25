# Server Implementation Details

This document provides an in-depth analysis of the server implementation in the Digital Marketplace application, focusing specifically on the two core files that make up the server framework: `index.ts` and `adapters.ts`.

## Source Files

The server implementation is located in:
- `/src/back-end/lib/server/index.ts` - Core types and utilities
- `/src/back-end/lib/server/adapters.ts` - Express adapter implementation

## index.ts: Core Server Framework

The `index.ts` file defines the fundamental abstractions of the server framework, establishing a flexible, type-safe foundation for handling HTTP requests and responses.

### Key Components

#### Session Management

```typescript
export type SessionId = Id;

export function parseSessionId(raw?: string): SessionId {
  return raw || "";
}

export type SessionIdToSession<Session> = (
  sessionId?: SessionId
) => Promise<Session>;

export type SessionToSessionId<Session> = (session: Session) => SessionId;
```

This simple but effective session management system allows the application to:
- Generate and parse unique session identifiers
- Convert between session IDs and session objects
- Provide type safety with the generic `<Session>` type

#### Request and Response Types

```typescript
export interface Request<Body, Session> {
  readonly id: string;
  readonly path: string;
  readonly headers: IncomingHttpHeaders;
  readonly logger: DomainLogger;
  readonly method: ServerHttpMethod;
  readonly session: Session;
  readonly params: Record<string, string>;
  readonly query: Record<string, string>;
  readonly body: Body;
}

export interface Response<Body, Session> {
  readonly code: number;
  readonly headers: OutgoingHttpHeaders;
  readonly session: Session;
  readonly body: Body;
}
```

These generic interfaces allow the framework to:
- Handle different types of request and response bodies
- Maintain type safety between request parsing and response generation
- Include session state in both requests and responses
- Track request-specific metadata like IDs and paths

#### Request Body Types

The framework uses Algebraic Data Types (ADTs) to model different types of request bodies:

```typescript
export type TextRequestBody = ADT<"text", string>;
export type JsonRequestBody = ADT<"json", JsonValue>;
export type FileRequestBody<FileUploadMetadata> = ADT<
  "file",
  FileUpload<FileUploadMetadata>
>;
```

This approach allows the framework to:
- Safely handle different types of request bodies
- Maintain type information about each body type
- Use pattern matching (via the `tag` property) to handle each type appropriately

#### Response Body Types

Similarly, the framework defines different types of response bodies:

```typescript
export type HtmlResponseBody = ADT<"html", string>;
export type TextResponseBody = ADT<"text", string>;
export type JsonResponseBody<Value = any> = ADT<"json", Value>;
export type FileResponseBody = ADT<"file", ResponseFile>;
export type ErrorResponseBody = ADT<"error", ErrorValue>;
```

Each response body type has associated utility functions:
- `makeHtmlResponseBody` - Creates an HTML response
- `makeTextResponseBody` - Creates a text response
- `makeJsonResponseBody` - Creates a JSON response
- `tryMakeFileResponseBody` - Safely creates a file response
- `makeErrorResponseBody` - Creates a standardized error response

#### File Handling

The framework includes sophisticated file handling capabilities:

```typescript
export interface ResponseFile {
  readonly buffer: Buffer;
  readonly contentType: string;
  readonly contentEncoding?: string;
  readonly contentDisposition?: string;
}

export function tryMakeFileResponseBody(
  path: string,
  contentType?: string,
  contentEncoding?: string,
  contentDisposition?: string
): FileResponseBody | null {
  // Implementation...
}

export function tryMakeGzipFileResponseBody(
  filePath: string
): FileResponseBody | null {
  // Implementation...
}

export function tryMakeBrotliFileResponseBody(
  filePath: string
): FileResponseBody | null {
  // Implementation...
}
```

These functions enable:
- Safe file loading from disk
- Automatic content type detection
- Support for content encoding and disposition
- Optimized handling of compressed files (Gzip, Brotli)

#### Request Transformation and Response Generation

The framework defines abstractions for transforming requests and generating responses:

```typescript
export type TransformRequest<RBA, RBB, Session> = (
  request: Request<RBA, Session>
) => Promise<RBB>;

export type Respond<ReqB, ResB, Session> = (
  request: Request<ReqB, Session>
) => Promise<Response<ResB, Session>>;

export type ParseRequestBody<IncomingReqBody, ParsedReqBody, Session> = (
  request: Request<IncomingReqBody, Session>
) => Promise<ParsedReqBody>;

export type ValidateRequestBody<
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  Session
> = (
  request: Request<ParsedReqBody, Session>
) => Promise<Validation<ValidatedReqBody, ReqBodyErrors>>;
```

These types allow for:
- Composition of request transformations
- Separation of request parsing, validation, and response generation
- Strong typing throughout the request processing pipeline

#### Handlers

Handlers combine parsing, validation, and response generation:

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

The framework includes utility functions for creating common handler types:
- `passThroughRequestBodyHandler` - For when no special parsing is needed
- `nullRequestBodyHandler` - For requests without bodies (like GET requests)

#### Hooks

Hooks enable code execution before and after request handling:

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

The framework includes utilities for combining hooks:
- `combineHooks` - Combines multiple hooks into a single hook

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

The framework provides utilities for working with routes:
- `namespaceRoute` - Prefixes a route's path with a namespace
- `addHooksToRoute` - Adds hooks to a route
- `createMapRoute` - Creates a function to transform routes

#### Router

A router is simply an array of routes:

```typescript
export type Router<
  IncomingReqBody,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ResBody,
  HookState,
  Session
> = Array<
  Route<
    IncomingReqBody,
    ParsedReqBody,
    ValidatedReqBody,
    ReqBodyErrors,
    ResBody,
    HookState,
    Session
  >
>;
```

#### Response Validation

The framework includes utilities for handling validated responses:

```typescript
export function wrapRespond<
  ValidReqB,
  InvalidReqB extends BodyWithErrors,
  ValidResB,
  InvalidResB,
  Session
>(
  responseValidation: ResponseValidation<...>
): Respond<...> {
  // Implementation...
}
```

This utility makes it easier to handle cases where requests may be valid or invalid.

## adapters.ts: Express Adapter Implementation

The `adapters.ts` file implements the adapter pattern to connect the abstract server framework to Express.js.

### Key Components

#### Adapter Types

```typescript
export interface AdapterRunParams<...> {
  router: Router<...>;
  sessionIdToSession: SessionIdToSession<Session>;
  sessionToSessionId: SessionToSessionId<Session>;
  host: string;
  port: number;
  maxMultipartFilesSize: number;
  parseFileUploadMetadata(raw: any): FileUploadMetaData;
}

export type Adapter<
  App,
  SupportedRequestBodies,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  SupportedResponseBodies,
  HookState,
  Session,
  FileUploadMetaData
> = (
  params: AdapterRunParams<...>
) => App;
```

These types define:
- The parameters needed to run an adapter
- The adapter function type, which creates an application from parameters

#### Express-Specific Types

```typescript
export type ExpressRequestBodies<FileUploadMetaData> =
  | JsonRequestBody
  | FileRequestBody<FileUploadMetaData>;

export type ExpressResponseBodies =
  | HtmlResponseBody
  | JsonResponseBody
  | FileResponseBody
  | TextResponseBody
  | ErrorResponseBody;

export type ExpressAdapter<...> = Adapter<
  expressLib.Application,
  ExpressRequestBodies<FileUploadMetaData>,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ExpressResponseBodies,
  HookState,
  Session,
  FileUploadMetaData
>;
```

These types specify:
- The request body types supported by Express
- The response body types supported by Express
- The Express-specific adapter type

#### Multipart Request Parsing

```typescript
function parseMultipartRequest<FileUploadMetadata>(
  maxSize: number,
  parseFileUploadMetadata: (raw: any) => FileUploadMetadata,
  expressReq: expressLib.Request
): Promise<FileRequestBody<FileUploadMetadata>> {
  // Implementation...
}
```

This function:
- Uses multiparty to parse multipart form data
- Handles file uploads with size limits
- Extracts metadata from form fields
- Creates a typed FileRequestBody

#### Express Adapter Implementation

```typescript
export function express<...>(): ExpressAdapter<...> {
  const logger = makeDomainLogger(consoleAdapter, "adapter:express");

  return ({
    router,
    sessionIdToSession,
    sessionToSessionId,
    maxMultipartFilesSize,
    parseFileUploadMetadata
  }) => {
    // Implementation...
  };
}
```

The Express adapter function:
1. Creates a domain-specific logger
2. Returns a function that takes adapter parameters
3. Implements the adapter logic inside that function

#### Response Handling

```typescript
function respond(
  response: Response<ExpressResponseBodies, Session>,
  expressRes: expressLib.Response
): void {
  // Implementation...
}
```

This function:
- Sets the HTTP status code
- Sets cookies for session management
- Sets response headers
- Sends the appropriate response body based on its type

#### Express Request Handler

```typescript
function makeExpressRequestHandler(
  route: Route<...>
): expressLib.RequestHandler {
  function asyncHandler(
    fn: (
      request: expressLib.Request,
      expressRes: expressLib.Response,
      next: expressLib.NextFunction
    ) => Promise<void>
  ): expressLib.RequestHandler {
    // Implementation...
  }

  return asyncHandler(async (expressReq, expressRes, next) => {
    // Implementation...
  });
}
```

This function:
1. Creates an async handler that catches errors
2. Checks if the HTTP method matches the route
3. Creates a session from cookies
4. Parses the request body
5. Creates a framework Request object
6. Runs the before hook
7. Parses, validates, and responds to the request
8. Cleans up temporary files
9. Runs the after hook
10. Sends the response using the respond function

#### CORS Configuration

```typescript
function corsOptionDelegate(
  req: expressLib.Request,
  callback: (err: Error | null, options?: corsLib.CorsOptions) => void
) {
  // Implementation...
}
```

This function configures CORS to allow only specific origins.

#### Express Application Setup

```typescript
// Set up the express app.
const app = expressLib();

// Parse JSON request bodies when provided.
app.use(
  bodyParser.json({
    type: "application/json"
  })
);

// Sign and parse cookies.
app.use(cookieParser(COOKIE_SECRET));

// Set up CORS to limit API access to specific domains
app.use(corsLib(corsOptionDelegate));

// Mount each route to the Express application.
router.forEach((route) => {
  app.all(route.path, makeExpressRequestHandler(route));
  if (SWAGGER_ENABLE) {
    app.use(SWAGGER_UI_PATH, swaggerUI.serve, swaggerUI.setup(specs));
  }
});

return app;
```

This code:
1. Creates an Express application
2. Sets up middleware for parsing JSON
3. Sets up middleware for parsing cookies
4. Sets up CORS middleware
5. Mounts each route to the application
6. Optionally sets up Swagger UI
7. Returns the configured Express application

## Integration with the Rest of the Application

### Connection to Resources

The server framework is designed to work with the resources module, which implements RESTful API endpoints. Resource routes are created using the `makeRouter` function from `back-end/lib/crud`:

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

### Main Application Setup

In the main application file (`src/back-end/index.ts`), the server is set up like this:

```typescript
// Create an Express adapter
const expressAdapter = adapters.express<...>();

// Create a server
const server = expressAdapter({
  router: [...routes],
  sessionIdToSession,
  sessionToSessionId,
  host,
  port,
  maxMultipartFilesSize,
  parseFileUploadMetadata
});

// Start the server
server.listen(port, host, () => {
  logger.info(`server started at ${scheme}://${host}:${port}`);
});
```

## Conclusion

The server implementation in the Digital Marketplace application is a sophisticated, type-safe framework built on top of Express.js. It provides a flexible architecture for handling HTTP requests and responses, with strong support for different body types, file uploads and downloads, session management, and request validation.

The separation between the abstract framework in `index.ts` and the concrete implementation in `adapters.ts` allows the application to potentially support different HTTP server libraries in the future while maintaining the same core abstractions.

The extensive use of TypeScript generics ensures type safety throughout the request/response cycle, reducing the risk of runtime errors and making the codebase more maintainable.
