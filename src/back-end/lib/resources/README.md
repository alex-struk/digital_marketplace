# Resources Module

The `resources` module in `src/back-end/lib/resources` is a core part of the backend architecture, implementing RESTful API endpoints for different entities in the system. This module follows a CRUD (Create, Read, Update, Delete) pattern and provides a consistent interface for handling various resources.

## Overview

The resources module implements API endpoints for various entities in the system, such as:
- Users
- Organizations
- Sessions
- Opportunities
- Proposals
- Files
- Affiliations
- Avatars
- Content
- Counters
- Email notifications
- Metrics
- Subscribers

Each resource module implements a standardized interface that maps to HTTP methods for RESTful APIs:
- Create → HTTP POST
- ReadOne → HTTP GET for a single resource
- ReadMany → HTTP GET for multiple resources
- Update → HTTP PUT/PATCH
- Delete → HTTP DELETE

## Architecture

### Resource Structure

Each resource file typically exports a default resource object that implements the `crud.BasicCrudResource` interface. The resource object contains methods for CRUD operations and a `routeNamespace` string that defines the URL path for the resource.

```typescript
const resource: crud.BasicCrudResource<Session, Connection> = {
  routeNamespace,  // e.g., "users", "organizations"
  create,          // Optional - HTTP POST handler
  readOne,         // Optional - HTTP GET handler for a single resource
  readMany,        // Optional - HTTP GET handler for multiple resources
  update,          // Optional - HTTP PUT/PATCH handler
  delete: delete_  // Optional - HTTP DELETE handler
};

export default resource;
```

### CRUD Operations

Each CRUD operation is a function that takes a database connection and returns a handler for the corresponding HTTP method. The handler is responsible for:

1. Parsing the request body (for Create/Update operations)
2. Validating the request body
3. Performing permission checks
4. Interacting with the database
5. Returning an appropriate response

#### Create Operation

The Create operation handles HTTP POST requests and creates a new resource:

```typescript
const create: crud.Create<
  Session,
  Connection,
  ParsedReqBody,
  ValidatedReqBody,
  ValidationErrors
> = (connection: Connection) => {
  return {
    async parseRequestBody(request) {
      // Parse the request body from JSON to a typed object
    },
    async validateRequestBody(request) {
      // Validate the parsed request body
      // Return valid() or invalid() result
    },
    async respond(request) {
      // Check permissions
      // Create the resource in the database
      // Return a response with appropriate status code
    }
  };
};
```

#### ReadOne Operation

The ReadOne operation handles HTTP GET requests for a single resource:

```typescript
const readOne: crud.ReadOne<Session, Connection> = (connection: Connection) => {
  return nullRequestBodyHandler(async (request) => {
    // Check permissions
    // Read the resource from the database
    // Return a response with the resource and appropriate status code
  });
};
```

#### ReadMany Operation

The ReadMany operation handles HTTP GET requests for multiple resources:

```typescript
const readMany: crud.ReadMany<Session, Connection> = (connection: Connection) => {
  return nullRequestBodyHandler(async (request) => {
    // Check permissions
    // Read resources from the database
    // Return a response with the resources and appropriate status code
  });
};
```

#### Update Operation

The Update operation handles HTTP PUT/PATCH requests to update an existing resource:

```typescript
const update: crud.Update<
  Session,
  Connection,
  ParsedReqBody,
  ValidatedReqBody,
  ValidationErrors
> = (connection: Connection) => {
  return {
    async parseRequestBody(request) {
      // Parse the request body from JSON to a typed object
    },
    async validateRequestBody(request) {
      // Validate the parsed request body
      // Return valid() or invalid() result
    },
    async respond(request) {
      // Check permissions
      // Update the resource in the database
      // Return a response with appropriate status code
    }
  };
};
```

#### Delete Operation

The Delete operation handles HTTP DELETE requests to remove a resource:

```typescript
const delete_: crud.Delete<
  Session,
  Connection,
  ValidatedReqBody,
  ValidationErrors
> = (connection: Connection) => {
  return {
    async validateRequestBody(request) {
      // Validate the request (typically checking if the resource exists)
      // Return valid() or invalid() result
    },
    async respond(request) {
      // Check permissions
      // Delete the resource from the database
      // Return a response with appropriate status code
    }
  };
};
```

## Request and Response Flow

1. **Request Parsing**: For operations that accept a request body (Create and Update), the request body is parsed from JSON to a typed object.

2. **Request Validation**: The parsed request body is validated to ensure it meets the requirements for the operation.

3. **Permission Checking**: The user's permissions are checked to ensure they have the right to perform the requested operation.

4. **Database Interaction**: The operation is performed on the database.

5. **Response Building**: An appropriate response is built and returned to the client.

## Resource Types

### Simple Resources

Simple resources like `session.ts` or `avatar.ts` implement a straightforward CRUD interface.

### Complex Resources

Complex resources like `opportunity` or `proposal` are organized into subdirectories with specific implementations for different types (e.g., code-with-us, sprint-with-us, team-with-us).

## Shared Resource Definitions

Resource interfaces are defined in the `shared/lib/resources` directory. These interfaces are used by both the frontend and backend to ensure type safety across the application.

For example, the `User` interface is defined in `shared/lib/resources/user.ts`:

```typescript
export interface User {
  id: Id;
  type: UserType;
  status: UserStatus;
  name: string;
  email: string | null;
  jobTitle: string;
  avatarImageFile: FileRecord | null;
  notificationsOn: Date | null;
  acceptedTermsAt: Date | null;
  lastAcceptedTermsAt: Date | null;
  idpUsername: string;
  deactivatedOn: Date | null;
  deactivatedBy: Id | null;
  capabilities: string[];
  idpId: string;
}
```

## Validation

Request validation is a critical part of the resources module. Validation is performed using the `valid` and `invalid` functions from `shared/lib/validation`:

```typescript
import { valid, invalid, Validation } from "shared/lib/validation";

// Example validation function
async function validateUser(user: User): Promise<Validation<User, string[]>> {
  if (!user.name) {
    return invalid(["Name is required"]);
  }
  return valid(user);
}
```

## Permissions

Permission checks are performed using functions from `back-end/lib/permissions`. These functions check if a user has the right to perform an operation on a resource:

```typescript
import * as permissions from "back-end/lib/permissions";

// Example permission check
if (!permissions.readOneUser(request.session, user.id)) {
  return respond(401, [permissions.ERROR_MESSAGE]);
}
```

## Error Handling

Errors are handled consistently across resources. Each operation returns an appropriate HTTP status code and error message:

- 200/201: Success
- 400: Bad Request (validation errors)
- 401: Unauthorized (permission errors)
- 404: Not Found
- 503: Service Unavailable (database errors)

## Integration with the Web Server

The resources module is integrated with the web server using the `makeRouter` function from `back-end/lib/crud`:

```typescript
import { makeRouter } from "back-end/lib/crud";
import * as resources from "back-end/lib/resources";

// Create routers for each resource
const userRouter = makeRouter(resources.user);
const organizationRouter = makeRouter(resources.organization);

// Add routers to the server
server.addRouter(userRouter);
server.addRouter(organizationRouter);
```

## Adding a New Resource

To add a new resource:

1. Define the resource interface in `shared/lib/resources`.
2. Create a new file in `src/back-end/lib/resources`.
3. Implement the CRUD operations for the resource.
4. Export a default resource object.
5. Add the resource to the server in `src/back-end/index.ts`.

## Best Practices

1. **Type Safety**: Use TypeScript's type system to ensure type safety across the application.
2. **Validation**: Validate all user input to prevent invalid data.
3. **Permission Checking**: Always check permissions before performing operations.
4. **Error Handling**: Handle errors consistently and return appropriate HTTP status codes.
5. **Code Reuse**: Reuse common patterns and functions to reduce code duplication.
6. **Testing**: Write tests for each resource to ensure it works as expected.

## Examples

### Example: Creating a User

```typescript
// Create a user
const createUser = async (connection, userData) => {
  const result = await db.createUser(connection, userData);
  if (isValid(result)) {
    return valid(result.value);
  } else {
    return invalid(result.value);
  }
};
```

### Example: Reading a User

```typescript
// Read a user
const readUser = async (connection, userId) => {
  const result = await db.readOneUser(connection, userId);
  if (isValid(result)) {
    return valid(result.value);
  } else {
    return invalid(["User not found"]);
  }
};
```

### Example: Updating a User

```typescript
// Update a user
const updateUser = async (connection, userId, userData) => {
  const result = await db.updateUser(connection, userId, userData);
  if (isValid(result)) {
    return valid(result.value);
  } else {
    return invalid(result.value);
  }
};
```

### Example: Deleting a User

```typescript
// Delete a user
const deleteUser = async (connection, userId) => {
  const result = await db.deleteUser(connection, userId);
  if (isValid(result)) {
    return valid(null);
  } else {
    return invalid(result.value);
  }
};
```

## Conclusion

The `resources` module is a core part of the backend architecture, providing a consistent interface for handling resources in the system. By following the patterns described in this README, you can create new resources that integrate seamlessly with the existing system.
