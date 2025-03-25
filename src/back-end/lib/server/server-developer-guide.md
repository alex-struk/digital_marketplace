# Server Developer Guide

This guide provides practical information for developers who need to work with the server infrastructure in the Digital Marketplace application. It includes examples, best practices, and common patterns.

## Table of Contents

1. [Introduction](#introduction)
2. [Creating a New Route](#creating-a-new-route)
3. [Handling Different Request Types](#handling-different-request-types)
4. [Request Validation](#request-validation)
5. [File Uploads and Downloads](#file-uploads-and-downloads)
6. [Session Management](#session-management)
7. [Working with Hooks](#working-with-hooks)
8. [Error Handling](#error-handling)
9. [Common Patterns](#common-patterns)
10. [Debugging Tips](#debugging-tips)

## Introduction

The Digital Marketplace server infrastructure is built on a custom, type-safe framework that sits on top of Express.js. It's designed to provide strong typing throughout the request/response cycle, with clear separation between request parsing, validation, and response generation.

Before diving into development, make sure you understand the core abstractions:

- **Request and Response** - Generic interfaces for HTTP requests and responses
- **Handlers** - Functions that parse, validate, and respond to requests
- **Routes** - Connections between HTTP methods, paths, and handlers
- **Hooks** - Functions that execute before and after request handling
- **Adapters** - Functions that connect the framework to an HTTP server (like Express)

## Creating a New Route

Creating a new route involves several steps:

### 1. Define Your Request and Response Types

```typescript
// Define the validated request body
interface CreateUserRequest {
  name: string;
  email: string;
  type: UserType;
}

// Define the response body
interface CreateUserResponse {
  id: string;
  name: string;
  email: string;
  type: UserType;
}
```

### 2. Create a Handler

```typescript
// Create a handler that parses and validates the request body
const createUserHandler: Handler<
  JsonRequestBody, // Incoming request body type (JSON)
  any,             // Parsed request body type (any JSON)
  CreateUserRequest, // Validated request body type
  string[],        // Request body error type
  JsonResponseBody<CreateUserResponse>, // Response body type
  Session          // Session type
> = {
  // Parse the request body
  parseRequestBody: async (request) => {
    if (request.body.tag === 'json') {
      return request.body.value;
    }
    throw new Error('Invalid request body');
  },

  // Validate the parsed body
  validateRequestBody: async (request) => {
    const body = request.body;
    const errors: string[] = [];

    if (!body.name) {
      errors.push('Name is required');
    }
    if (!body.email) {
      errors.push('Email is required');
    }
    if (!body.type) {
      errors.push('User type is required');
    }

    if (errors.length > 0) {
      return invalid(errors);
    }

    return valid({
      name: body.name,
      email: body.email,
      type: body.type
    });
  },

  // Respond to the request
  respond: wrapRespond({
    valid: async (request) => {
      const { name, email, type } = request.body;

      // Create a user in the database
      const user = await createUser({ name, email, type });

      // Return a response
      return {
        code: 201, // Created
        headers: {},
        session: request.session,
        body: makeJsonResponseBody({
          id: user.id,
          name: user.name,
          email: user.email,
          type: user.type
        })
      };
    },
    invalid: async (request) => {
      return {
        code: 400, // Bad Request
        headers: {},
        session: request.session,
        body: makeJsonResponseBody({ errors: request.body })
      };
    }
  })
};
```

### 3. Create a Route

```typescript
const createUserRoute: Route<
  JsonRequestBody,
  any,
  CreateUserRequest,
  string[],
  JsonResponseBody<CreateUserResponse>,
  null,
  Session
> = {
  method: ServerHttpMethod.Post,
  path: '/api/users',
  handler: createUserHandler
};
```

### 4. Add the Route to a Router

```typescript
const userRouter: Router<
  JsonRequestBody,
  any,
  CreateUserRequest,
  string[],
  JsonResponseBody<CreateUserResponse>,
  null,
  Session
> = [
  createUserRoute
];
```

## Handling Different Request Types

The framework supports different types of request bodies:

### JSON Requests

```typescript
// Parse a JSON request body
parseRequestBody: async (request) => {
  if (request.body.tag === 'json') {
    return request.body.value;
  }
  throw new Error('Invalid request body');
}
```

### File Upload Requests

```typescript
// Parse a file upload request body
parseRequestBody: async (request) => {
  if (request.body.tag === 'file') {
    return {
      name: request.body.value.name,
      path: request.body.value.path,
      metadata: request.body.value.metadata
    };
  }
  throw new Error('Invalid request body');
}
```

### No Body (GET Requests)

For GET requests or other requests without bodies, use the `nullRequestBodyHandler` utility:

```typescript
const getUsersHandler = nullRequestBodyHandler<
  JsonResponseBody<User[]>,
  Session
>(async (request) => {
  // Get users from the database
  const users = await getUsers();

  // Return a response
  return {
    code: 200,
    headers: {},
    session: request.session,
    body: makeJsonResponseBody(users)
  };
});
```

## Request Validation

Request validation is a crucial step in the request processing pipeline. It ensures that the request body meets your expectations before you process it.

### Basic Validation

```typescript
validateRequestBody: async (request) => {
  const body = request.body;
  const errors: string[] = [];

  if (!body.name) {
    errors.push('Name is required');
  }
  if (!body.email) {
    errors.push('Email is required');
  }

  if (errors.length > 0) {
    return invalid(errors);
  }

  return valid({
    name: body.name,
    email: body.email
  });
}
```

### Complex Validation

For more complex validation, you can create separate validation functions:

```typescript
// Validate an email address
function validateEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email);
}

// Validate a user
async function validateUser(user: any): Promise<Validation<User, string[]>> {
  const errors: string[] = [];

  if (!user.name) {
    errors.push('Name is required');
  }

  if (!user.email) {
    errors.push('Email is required');
  } else if (!validateEmail(user.email)) {
    errors.push('Email is invalid');
  }

  if (errors.length > 0) {
    return invalid(errors);
  }

  return valid({
    name: user.name,
    email: user.email
  });
}
```

## File Uploads and Downloads

The framework includes robust support for file uploads and downloads.

### Handling File Uploads

```typescript
// Define the file upload metadata type
interface FileUploadMetadata {
  userId: string;
  description: string;
}

// Create a handler for file uploads
const uploadFileHandler: Handler<
  FileRequestBody<FileUploadMetadata>,
  FileUpload<FileUploadMetadata>,
  FileUpload<FileUploadMetadata>,
  string[],
  JsonResponseBody<{ id: string }>,
  Session
> = {
  // Parse the request body
  parseRequestBody: async (request) => {
    if (request.body.tag === 'file') {
      return request.body.value;
    }
    throw new Error('Invalid request body');
  },

  // Validate the parsed body
  validateRequestBody: async (request) => {
    const file = request.body;
    const errors: string[] = [];

    if (!file.metadata || !file.metadata.userId) {
      errors.push('User ID is required');
    }

    if (errors.length > 0) {
      return invalid(errors);
    }

    return valid(file);
  },

  // Respond to the request
  respond: wrapRespond({
    valid: async (request) => {
      const file = request.body;

      // Save the file to permanent storage
      const fileId = await saveFile(file.path, file.name, file.metadata);

      // Return a response
      return {
        code: 201, // Created
        headers: {},
        session: request.session,
        body: makeJsonResponseBody({ id: fileId })
      };
    },
    invalid: async (request) => {
      return {
        code: 400, // Bad Request
        headers: {},
        session: request.session,
        body: makeJsonResponseBody({ errors: request.body })
      };
    }
  })
};
```

### Serving Files

```typescript
// Create a handler for file downloads
const downloadFileHandler = nullRequestBodyHandler<
  FileResponseBody,
  Session
>(async (request) => {
  const fileId = request.params.id;

  // Get the file from storage
  const file = await getFile(fileId);

  if (!file) {
    return {
      code: 404, // Not Found
      headers: {},
      session: request.session,
      body: makeErrorResponseBody(new Error('File not found'))
    };
  }

  // Create a file response body
  const fileResponseBody = tryMakeFileResponseBody(
    file.path,
    file.contentType,
    undefined,
    `attachment; filename="${file.name}"`
  );

  if (!fileResponseBody) {
    return {
      code: 404, // Not Found
      headers: {},
      session: request.session,
      body: makeErrorResponseBody(new Error('File not found'))
    };
  }

  // Return a response
  return {
    code: 200,
    headers: {},
    session: request.session,
    body: fileResponseBody
  };
});
```

## Session Management

Sessions are managed through cookies, with session IDs converted to session objects using the `sessionIdToSession` function.

### Accessing the Session

```typescript
// Access the session in a handler
respond: async (request) => {
  const session = request.session;

  // Use the session
  if (!session.user) {
    return {
      code: 401, // Unauthorized
      headers: {},
      session,
      body: makeJsonResponseBody({ error: 'Unauthorized' })
    };
  }

  // Return a response
  return {
    code: 200,
    headers: {},
    session,
    body: makeJsonResponseBody({ user: session.user })
  };
}
```

### Updating the Session

```typescript
// Update the session in a handler
respond: async (request) => {
  const oldSession = request.session;

  // Create a new session
  const newSession = {
    ...oldSession,
    lastActive: new Date()
  };

  // Return a response with the updated session
  return {
    code: 200,
    headers: {},
    session: newSession,
    body: makeJsonResponseBody({ success: true })
  };
}
```

## Working with Hooks

Hooks allow you to execute code before and after request handling.

### Creating a Hook

```typescript
// Create a hook that logs requests
const loggingHook: RouteHook<
  any,
  any,
  any,
  any,
  { startTime: number },
  Session
> = {
  before: async (request) => {
    // Log the request
    request.logger.info(`Request started: ${request.method} ${request.path}`);

    // Return state for the after hook
    return { startTime: Date.now() };
  },
  after: async (state, request, response) => {
    // Calculate the request duration
    const duration = Date.now() - state.startTime;

    // Log the response
    request.logger.info(`Request completed: ${request.method} ${request.path} - ${response.code} (${duration}ms)`);
  }
};
```

### Adding Hooks to Routes

```typescript
// Add a hook to a route
const routeWithHook = {
  ...route,
  hook: loggingHook
};

// Or add hooks to multiple routes
const routesWithHooks = addHooksToRoute([loggingHook], routes);
```

## Error Handling

Proper error handling is essential for a robust server application.

### Catching Errors in Handlers

```typescript
// Handle errors in a respond function
respond: async (request) => {
  try {
    // Do something that might throw an error
    const result = await doSomething();

    // Return a successful response
    return {
      code: 200,
      headers: {},
      session: request.session,
      body: makeJsonResponseBody(result)
    };
  } catch (error) {
    // Log the error
    request.logger.error(`Error: ${error.message}`, error);

    // Return an error response
    return {
      code: 500, // Internal Server Error
      headers: {},
      session: request.session,
      body: makeErrorResponseBody(error)
    };
  }
}
```

### Using the wrapRespond Utility

The `wrapRespond` utility makes it easier to handle validation errors:

```typescript
respond: wrapRespond({
  valid: async (request) => {
    // Handle valid requests
    return {
      code: 200,
      headers: {},
      session: request.session,
      body: makeJsonResponseBody({ success: true })
    };
  },
  invalid: async (request) => {
    // Handle invalid requests
    return {
      code: 400, // Bad Request
      headers: {},
      session: request.session,
      body: makeJsonResponseBody({ errors: request.body })
    };
  }
})
```

## Common Patterns

### Resource Routes

A common pattern is to create routes for CRUD operations on a resource:

```typescript
// Create a router for a resource
const resourceRouter: Router<...> = [
  {
    method: ServerHttpMethod.Post,
    path: '/api/resources',
    handler: createResourceHandler
  },
  {
    method: ServerHttpMethod.Get,
    path: '/api/resources/:id',
    handler: getResourceHandler
  },
  {
    method: ServerHttpMethod.Get,
    path: '/api/resources',
    handler: listResourcesHandler
  },
  {
    method: ServerHttpMethod.Put,
    path: '/api/resources/:id',
    handler: updateResourceHandler
  },
  {
    method: ServerHttpMethod.Delete,
    path: '/api/resources/:id',
    handler: deleteResourceHandler
  }
];
```

### Pagination

For endpoints that return multiple items, implement pagination:

```typescript
// Create a handler for a paginated endpoint
const listResourcesHandler = nullRequestBodyHandler<
  JsonResponseBody<{
    items: Resource[];
    total: number;
    offset: number;
    limit: number;
  }>,
  Session
>(async (request) => {
  // Parse pagination parameters
  const offset = parseInt(request.query.offset || '0', 10);
  const limit = parseInt(request.query.limit || '10', 10);

  // Get items from the database
  const { items, total } = await listResources(offset, limit);

  // Return a response
  return {
    code: 200,
    headers: {},
    session: request.session,
    body: makeJsonResponseBody({
      items,
      total,
      offset,
      limit
    })
  };
});
```

### Filtering and Sorting

For endpoints that support filtering and sorting:

```typescript
// Create a handler for a filtered and sorted endpoint
const listResourcesHandler = nullRequestBodyHandler<
  JsonResponseBody<{
    items: Resource[];
    total: number;
  }>,
  Session
>(async (request) => {
  // Parse filter and sort parameters
  const filters = {
    name: request.query.name,
    type: request.query.type
  };

  const sort = {
    field: request.query.sortBy || 'createdAt',
    order: request.query.sortOrder || 'desc'
  };

  // Get items from the database
  const { items, total } = await listResources(filters, sort);

  // Return a response
  return {
    code: 200,
    headers: {},
    session: request.session,
    body: makeJsonResponseBody({
      items,
      total
    })
  };
});
```

## Debugging Tips

### Logging

Use the request-specific logger for debugging:

```typescript
request.logger.debug('Debug information');
request.logger.info('Info message');
request.logger.warn('Warning message');
request.logger.error('Error message', error);
```

### Inspecting Request Details

Log request details for debugging:

```typescript
request.logger.debug({
  path: request.path,
  method: request.method,
  params: request.params,
  query: request.query,
  body: request.body,
  session: request.session
});
```

### Testing Endpoints

Use a tool like Postman or curl to test your endpoints:

```bash
# Test a GET endpoint
curl -X GET http://localhost:3000/api/resources

# Test a POST endpoint
curl -X POST http://localhost:3000/api/resources \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Resource","type":"test"}'

# Test a file upload endpoint
curl -X POST http://localhost:3000/api/files \
  -F "file=@/path/to/file.jpg" \
  -F "metadata={\"userId\":\"123\",\"description\":\"Test file\"}"
```

## Conclusion

The Digital Marketplace server framework provides a robust foundation for building type-safe, maintainable APIs. By following the patterns and best practices outlined in this guide, you can create endpoints that are consistent, reliable, and easy to maintain.

Remember the key principles:
- Use strong typing throughout your code
- Validate all user input
- Handle errors gracefully
- Use hooks for cross-cutting concerns
- Keep your code organized and maintainable

Happy coding!
