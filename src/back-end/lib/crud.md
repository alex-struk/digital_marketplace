# CRUD Operations Documentation

This document provides comprehensive documentation for the CRUD (Create, Read, Update, Delete) operations implementation in the back-end framework.

## Table of Contents
- [Overview](#overview)
- [Core Types](#core-types)
- [CRUD Operations](#crud-operations)
- [Resource Interface](#resource-interface)
- [Router Creation](#router-creation)
- [Type Parameters](#type-parameters)
- [Usage Examples](#usage-examples)

## Overview

The CRUD module provides a type-safe implementation of standard CRUD operations for the back-end framework. It includes comprehensive TypeScript types and helper functions to create RESTful endpoints for resources.

## Core Types

### CrudAction

The base type for all CRUD operations. It defines the structure for handling requests and responses:

```typescript
type CrudAction<
  IncomingReqBody,
  ParsedReqBody,
  ValidatedReqBody,
  ReqBodyErrors,
  ResBody,
  Session,
  Connection,
  PickFromHandler extends keyof Handler<...>
>
```

## CRUD Operations

### Create Operation

Creates a new resource based on the provided request body.

```typescript
type Create<
  Session,
  Connection,
  ParsedReqBody = null,
  ValidatedReqBody = null,
  ReqBodyErrors = null,
  SupportedRequestBodies = DefaultSupportedRequestBodies,
  SupportedResponseBodies = DefaultSupportedResponseBodies
>
```

- **Method**: POST
- **Path**: "/"
- **Features**:
  - Parses request body
  - Validates request body
  - Creates new resource
  - Returns created resource

### ReadOne Operation

Retrieves a single resource by ID.

```typescript
type ReadOne<
  Session,
  Connection,
  ValidatedReqBody = null,
  ReqBodyErrors = null,
  SupportedResponseBodies = DefaultSupportedResponseBodies
>
```

- **Method**: GET
- **Path**: "/:id"
- **Features**:
  - Validates request parameters
  - Returns single resource

### ReadMany Operation

Retrieves multiple resources.

```typescript
type ReadMany<
  Session,
  Connection,
  SupportedResponseBodies = DefaultSupportedResponseBodies
>
```

- **Method**: GET
- **Path**: "/"
- **Features**:
  - Returns multiple resources
  - No request body validation

### Update Operation

Updates an existing resource.

```typescript
type Update<
  Session,
  Connection,
  ParsedReqBody = null,
  ValidatedReqBody = null,
  ReqBodyErrors = null,
  SupportedRequestBodies = DefaultSupportedRequestBodies,
  SupportedResponseBodies = DefaultSupportedResponseBodies
>
```

- **Method**: PUT
- **Path**: "/:id"
- **Features**:
  - Parses request body
  - Validates request body
  - Updates existing resource
  - Returns updated resource

### Delete Operation

Deletes an existing resource.

```typescript
type Delete<
  Session,
  Connection,
  ValidatedReqBody = null,
  ReqBodyErrors = null,
  SupportedResponseBodies = DefaultSupportedResponseBodies
>
```

- **Method**: DELETE
- **Path**: "/:id"
- **Features**:
  - Validates request parameters
  - Deletes resource
  - Returns success response

## Resource Interface

The `Resource` interface defines the structure for a complete CRUD resource:

```typescript
interface Resource<
  SupportedRequestBodies,
  SupportedResponseBodies,
  CreateParsedReqB,
  CreateValidatedReqB,
  CreateReqBErrors,
  ReadOneValidatedReqB,
  ReadOneReqBErrors,
  UpdateParsedReqB,
  UpdateValidatedReqB,
  UpdateReqBErrors,
  DeleteValidatedReqB,
  DeleteReqBErrors,
  Session,
  Connection
>
```

### Properties
- `routeNamespace`: string - The namespace for the resource routes
- `create?`: Create operation handler
- `readOne?`: ReadOne operation handler
- `readMany?`: ReadMany operation handler
- `update?`: Update operation handler
- `delete?`: Delete operation handler

## Router Creation

The `makeRouter` function creates a router for a given resource:

```typescript
function makeRouter<...>(
  resource: Resource<...>
): (connection: Connection) => Router<...>
```

### Features
- Automatically creates routes for all defined CRUD operations
- Handles route namespacing
- Manages HTTP methods and paths
- Provides type-safe request/response handling

## Type Parameters

### Common Type Parameters
- `Session`: The session type for authentication/authorization
- `Connection`: The database connection type
- `SupportedRequestBodies`: Supported request body types
- `SupportedResponseBodies`: Supported response body types

### Operation-Specific Parameters
- `ParsedReqBody`: Parsed request body type
- `ValidatedReqBody`: Validated request body type
- `ReqBodyErrors`: Request body validation errors type

## Usage Examples

### Basic Resource Definition

```typescript
const userResource: Resource<...> = {
  routeNamespace: "users",
  create: (connection) => ({
    parseRequestBody: async (req) => {...},
    validateRequestBody: async (parsed) => {...},
    respond: async (validated) => {...}
  }),
  readOne: (connection) => ({
    validateRequestBody: async (req) => {...},
    respond: async (validated) => {...}
  }),
  // ... other operations
};
```

### Creating a Router

```typescript
const router = makeRouter(userResource)(connection);
```

## Best Practices

1. **Type Safety**
   - Always specify appropriate types for request/response bodies
   - Use strict validation for request bodies
   - Handle all possible error cases

2. **Resource Organization**
   - Use meaningful route namespaces
   - Implement only the necessary CRUD operations
   - Keep resource definitions focused and single-purpose

3. **Error Handling**
   - Implement proper validation error handling
   - Return appropriate HTTP status codes
   - Provide meaningful error messages

4. **Performance**
   - Optimize database queries
   - Implement proper connection handling
   - Use appropriate caching strategies when needed

## Notes

- All CRUD operations are optional in the Resource interface
- The framework supports both JSON and form data request bodies
- Response bodies are flexible and can be customized per operation
- The system is designed to be type-safe and maintainable
