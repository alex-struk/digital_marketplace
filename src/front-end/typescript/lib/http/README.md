# HTTP Module Documentation

## Overview

The HTTP module in `src/front-end/typescript/lib/http` provides a standardized interface for making API calls from the front-end to the back-end services in the Digital Marketplace application. It implements a CRUD (Create, Read, Update, Delete) pattern for interacting with back-end resources and handles request/response validation.

## Core Components

### Module Structure

```
src/front-end/typescript/lib/http/
├── crud.ts           # Core CRUD utilities for HTTP requests
└── api/              # API endpoint implementations
    ├── index.ts      # Main entry point and re-exports
    ├── user.ts       # User-related API endpoints
    ├── organization/ # Organization-related endpoints
    ├── opportunity/  # Opportunity-related endpoints
    ├── ...           # Other resource endpoints
```

### CRUD Module (`crud.ts`)

The `crud.ts` file provides the foundation for HTTP request handling with these key components:

1. **API Namespace Utilities**:
   - `apiNamespace()`: Prefixes API paths correctly for the application

2. **Type Definitions**:
   - `HandleResponse<ValidResponse, InvalidResponse, Msg>`: Callback for handling HTTP responses
   - `CreateAction<RequestBody, ValidResponse, InvalidResponse, Msg>`: Creates a new resource
   - `CreateManyAction<RequestBodyItem, ValidItem, InvalidItem, Msg>`: Creates multiple resources
   - `ReadManyAction<ValidItem, InvalidResponse, Msg>`: Retrieves multiple resources
   - `ReadOneAction<ValidResponse, InvalidResponse, Msg>`: Retrieves a single resource by ID
   - `UpdateAction<RequestBody, ValidResponse, InvalidResponse, Msg>`: Updates an existing resource
   - `DeleteAction<ValidResponse, InvalidResponse, Msg>`: Deletes a resource

3. **Action Factories**:
   - `makeCreateAction()`: Factory for creating POST requests
   - `makeCreateManyAction()`: Factory for creating multiple resources in sequence
   - `makeReadManyAction()`: Factory for creating GET requests for collections
   - `makeReadOneAction()`: Factory for creating GET requests for individual resources
   - `makeUpdateAction()`: Factory for creating PUT requests
   - `makeDeleteAction()`: Factory for creating DELETE requests

### API Module (`api/`)

The `api/` directory contains implementations for specific API endpoints, organized by resource type:

- Each file implements CRUD operations for a specific resource
- Endpoints use the CRUD factories from `crud.ts`
- All endpoints are re-exported through `api/index.ts`

## Integration with Back-end

The HTTP module interfaces with back-end resources defined in:

1. **Database Layer**: `src/back-end/lib/db/`
   - Contains raw database operations
   - Implements data access and transformation
   - Handles database connections and transactions

2. **Resource Layer**: `src/shared/lib/resources/`
   - Defines shared type definitions for resources
   - Contains validation schemas used by both front-end and back-end
   - Provides resource-specific utility functions

## Usage Patterns

### Making API Requests

To make an API request, import the API module and use the appropriate function:

```typescript
import * as api from "front-end/lib/http/api";
import { Cmd } from "front-end/lib/framework";
import { User } from "shared/lib/resources/user";
import { adt, ADT } from "shared/lib/types";

// Define your message type
type Msg = ADT<"gotUser", api.ResponseValidation<User, string[]>>;

// Create a command to fetch a user
const getUserCmd: Cmd<Msg> =
  api.users.readOne<Msg>()("userId", response => adt("gotUser", response));
```

### Handling Responses

Responses are wrapped in a `ResponseValidation` type that can be:
- `valid`: Contains successfully processed data
- `invalid`: Contains validation errors
- `unhandled`: Represents an unexpected error

```typescript
// Example response handler
function handleUserResponse(response: api.ResponseValidation<User, string[]>): Msg {
  if (api.isValid(response)) {
    // Handle valid response
    return adt("userLoaded", response.value);
  } else if (api.isInvalid(response)) {
    // Handle validation errors
    return adt("userLoadingError", response.value);
  } else {
    // Handle unhandled/unexpected errors
    return adt("userLoadingError", ["An unexpected error occurred"]);
  }
}
```

## Creating New API Endpoints

To create a new API endpoint for a resource:

1. **Define Resource Types**:
   Create types in `shared/lib/resources/your-resource.ts`:
   ```typescript
   export interface YourResource {
     id: Id;
     name: string;
     // other properties
   }

   export interface CreateRequestBody {
     name: string;
     // other properties
   }

   export interface UpdateRequestBody {
     name?: string;
     // other properties
   }

   export type CreateValidationErrors = string[];
   export type UpdateValidationErrors = string[];
   ```

2. **Create API Client Functions**:
   Create a file in `front-end/typescript/lib/http/api/your-resource.ts`:
   ```typescript
   import * as crud from "front-end/lib/http/crud";
   import * as Resource from "shared/lib/resources/your-resource";

   const NAMESPACE = "your-resources";

   export function create<Msg>(): crud.CreateAction<
     Resource.CreateRequestBody,
     Resource.YourResource,
     Resource.CreateValidationErrors,
     Msg
   > {
     return crud.makeCreateAction(NAMESPACE, (a: Resource.YourResource) => a);
   }

   export function readMany<Msg>(): crud.ReadManyAction<
     Resource.YourResource,
     string[],
     Msg
   > {
     return crud.makeReadManyAction(NAMESPACE, (a: Resource.YourResource) => a);
   }

   export function readOne<Msg>(): crud.ReadOneAction<
     Resource.YourResource,
     string[],
     Msg
   > {
     return crud.makeReadOneAction(NAMESPACE, (a: Resource.YourResource) => a);
   }

   export function update<Msg>(): crud.UpdateAction<
     Resource.UpdateRequestBody,
     Resource.YourResource,
     Resource.UpdateValidationErrors,
     Msg
   > {
     return crud.makeUpdateAction(NAMESPACE, (a: Resource.YourResource) => a);
   }

   export function delete_<Msg>(): crud.DeleteAction<
     Resource.YourResource,
     string[],
     Msg
   > {
     return crud.makeDeleteAction(NAMESPACE, (a: Resource.YourResource) => a);
   }
   ```

3. **Export from API Index**:
   Add your resource to `front-end/typescript/lib/http/api/index.ts`:
   ```typescript
   export * as yourResources from "front-end/lib/http/api/your-resource";
   ```

## Advanced Usage

### Batch Operations

For creating multiple resources at once, use the `makeCreateManyAction` utility:

```typescript
// Create a function to create many resources
export function createMany<Msg>(): crud.CreateManyAction<
  Resource.CreateRequestBody,
  Resource.YourResource,
  Resource.CreateValidationErrors,
  Msg
> {
  return crud.makeCreateManyAction(
    create<crud.CreateManyAccumulator<Resource.YourResource, Resource.CreateValidationErrors>>(),
    [] as Resource.CreateValidationErrors
  );
}
```

### Custom Query Parameters

Add query parameters to your requests:

```typescript
// With query parameters
export function readManyWithFilters<Msg>(): crud.ReadManyAction<
  Resource.YourResource,
  string[],
  Msg
> {
  return crud.makeReadManyAction(NAMESPACE, (a: Resource.YourResource) => a, "activeOnly=true");
}
```

## Best Practices

1. **Type Safety**: Leverage TypeScript types for compile-time checks
2. **Error Handling**: Always handle all response types (valid, invalid, unhandled)
3. **Transformation**: Use transformation functions to adapt API responses
4. **Consistency**: Follow the established CRUD patterns
5. **Code Reuse**: Leverage the existing factories for creating endpoints
6. **Documentation**: Document any non-standard behavior

## Connection to Framework

The HTTP module integrates with the application framework using:

- `component.cmd.httpRequest`: Underlying command for HTTP requests
- `component.cmd.Cmd<Msg>`: Command pattern for handling side effects
- `ResponseValidation`: Consistent response handling

This integration allows HTTP requests to be composed and integrated into the application's message flow while maintaining type safety and proper error handling.
