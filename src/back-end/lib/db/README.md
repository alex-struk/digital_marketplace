# Database (db) Module Documentation

## Overview

The `db` module provides a standardized interface for interacting with the application's database. It's built on top of Knex.js and follows a consistent pattern for database operations. This module implements data access operations for all entities in the Digital Marketplace application.

## Purpose

The primary purposes of this module are:

1. Provide a consistent interface for database operations
2. Handle error management and logging
3. Define type-safe database entities and operations
4. Ensure data validation throughout database operations
5. Separate database logic from business logic

## Structure

The `db` module is organized as follows:

```
/src/back-end/lib/db/
├── index.ts                    # Core utilities and exports
├── affiliation.ts              # User-organization affiliations
├── content.ts                  # Content management
├── counter.ts                  # Counter operations for sequences
├── file.ts                     # File storage and retrieval
├── metrics.ts                  # System metrics
├── organization.ts             # Organization management
├── service-area.ts             # Service area definitions
├── session.ts                  # User session management
├── user.ts                     # User management
├── opportunity/                # Opportunity types
│   ├── code-with-us.ts         # Code With Us opportunities
│   ├── sprint-with-us.ts       # Sprint With Us opportunities
│   └── team-with-us.ts         # Team With Us opportunities
├── proposal/                   # Proposal types
│   ├── code-with-us.ts         # Code With Us proposals
│   ├── sprint-with-us.ts       # Sprint With Us proposals
│   └── team-with-us.ts         # Team With Us proposals
└── subscribers/                # Subscription management
    ├── code-with-us.ts         # Code With Us subscribers
    ├── sprint-with-us.ts       # Sprint With Us subscribers
    └── team-with-us.ts         # Team With Us subscribers
```

## Core Components

### Connection Interface

The module uses a `Connection` type that wraps a Knex connection:

```typescript
export type Connection = Knex;
export type Transaction = Knex.Transaction;
```

### Error Handling

The module provides a standardized approach to error handling with the `tryDb` utility function:

```typescript
export function tryDb<Args extends unknown[], Valid>(
  fn: DbFn<Args, Valid>
): DbFn<Args, Valid> {
  return async (connection, ...args) => {
    try {
      return await fn(connection, ...args);
    } catch (e) {
      const error = e as Error;
      logger.error("database operation failed", {
        message: error.message,
        stack: error.stack
      });
      return invalid(null);
    }
  };
}
```

This function wraps database operations, catching any errors, logging them, and returning a validation object.

### Validation

The module uses a `DatabaseValidation` type for handling validation:

```typescript
export type DatabaseValidation<Valid> = Validation<Valid, null>;
```

## Common Usage Patterns

### Reading Data

Database read operations follow this pattern:

```typescript
export const readOneEntity = tryDb<[Id], Entity | null>(
  async (connection, id) => {
    const result = await connection<RawEntity>("entities")
      .where({ id })
      .select("*")
      .first();
    return valid(result ? await rawEntityNameToEntityName(connection, result) : null);
  }
);
```

### Creating Data

Database create operations follow this pattern:

```typescript
export const createEntity = tryDb<[CreateEntityParams], Entity>(
  async (connection, entity) => {
    const now = new Date();
    const [result] = await connection<RawEntity>("entities").insert(
      {
        ...entity,
        id: generateUuid(),
        createdAt: now,
        updatedAt: now
      } as CreateEntityParams,
      ["*"]
    );
    return valid(await rawEntityNameToEntityName(connection, result));
  }
);
```

### Updating Data

Database update operations follow this pattern:

```typescript
export const updateEntity = tryDb<[Id, UpdateEntityParams], Entity>(
  async (connection, id, entity) => {
    const now = new Date();
    const [result] = await connection<RawEntity>("entities")
      .where({ id })
      .update(
        {
          ...entity,
          updatedAt: now
        },
        ["*"]
      );
    return valid(await rawEntityNameToEntityName(connection, result));
  }
);
```

### Deleting Data

Database delete operations follow this pattern:

```typescript
export const deleteEntity = tryDb<[Id], Entity>(async (connection, id) => {
  const [result] = await connection<RawEntity>("entities")
    .where({ id })
    .delete(["*"]);
  return valid(await rawEntityNameToEntityName(connection, result));
});
```

## Entity Structure

Each entity module typically contains:

1. **Type Definitions**:
   - `Entity` - The domain model
   - `RawEntity` - The database representation
   - `CreateEntityParams` - Parameters for creating an entity
   - `UpdateEntityParams` - Parameters for updating an entity

2. **Conversion Functions**:
   - `rawEntityNameToEntityName` - Converts a raw database record to a domain entity (e.g., `rawUserToUser`, `rawSessionToSession`, `rawContentToContent`)

3. **CRUD Operations**:
   - `createEntity` - Creates a new entity
   - `readOneEntity` - Reads a single entity by ID
   - `readManyEntities` - Reads multiple entities (with optional filters)
   - `updateEntity` - Updates an entity
   - `deleteEntity` - Deletes an entity

4. **Custom Operations**:
   - Entity-specific operations like `findEntityByName`, etc.

## How to Create a New Entity

To create a new database entity module:

1. **Create a new file**: Create a file named after your entity (e.g., `new-entity.ts`)

2. **Define the types**:
   ```typescript
   import { Connection, tryDb } from "back-end/lib/db";
   import { generateUuid } from "back-end/lib";
   import { valid } from "shared/lib/http";
   import { Id } from "shared/lib/types";

   // Define the domain model
   export interface NewEntity {
     id: Id;
     name: string;
     description: string;
     createdAt: Date;
     updatedAt: Date;
   }

   // Define the raw database model
   export interface RawNewEntity extends NewEntity {}

   // Define creation parameters
   export interface CreateNewEntityParams {
     name: string;
     description: string;
   }

   // Define update parameters
   export interface UpdateNewEntityParams {
     name?: string;
     description?: string;
   }
   ```

3. **Implement the conversion function**:
   ```typescript
   export async function rawNewEntityToNewEntity(
     connection: Connection,
     raw: RawNewEntity
   ): Promise<NewEntity> {
     return raw;
   }
   ```

4. **Implement CRUD operations**:
   ```typescript
   export const createNewEntity = tryDb<[CreateNewEntityParams], NewEntity>(
     async (connection, params) => {
       const now = new Date();
       const [result] = await connection<RawNewEntity>("new_entities")
         .insert(
           {
             ...params,
             id: generateUuid(),
             createdAt: now,
             updatedAt: now
           },
           ["*"]
         );
       return valid(await rawNewEntityToNewEntity(connection, result));
     }
   );

   export const readOneNewEntity = tryDb<[Id], NewEntity | null>(
     async (connection, id) => {
       const result = await connection<RawNewEntity>("new_entities")
         .where({ id })
         .first();
       return valid(
         result ? await rawNewEntityToNewEntity(connection, result) : null
       );
     }
   );

   export const updateNewEntity = tryDb<
     [Id, UpdateNewEntityParams],
     NewEntity | null
   >(async (connection, id, params) => {
     const now = new Date();
     const [result] = await connection<RawNewEntity>("new_entities")
       .where({ id })
       .update(
         {
           ...params,
           updatedAt: now
         },
         ["*"]
       );
     return valid(
       result ? await rawNewEntityToNewEntity(connection, result) : null
     );
   });

   export const deleteNewEntity = tryDb<[Id], NewEntity | null>(
     async (connection, id) => {
       const [result] = await connection<RawNewEntity>("new_entities")
         .where({ id })
         .delete(["*"]);
       return valid(
         result ? await rawNewEntityToNewEntity(connection, result) : null
       );
     }
   );
   ```

5. **Export from index.ts**: Add an export statement to `index.ts`:
   ```typescript
   export * from "back-end/lib/db/new-entity";
   ```

## Best Practices

1. **Use the `tryDb` wrapper**: Always wrap database operations with `tryDb` to ensure consistent error handling.

2. **Type Everything**: Use TypeScript interfaces to define all entities and their relationships.

3. **Validate Input/Output**: Ensure all data going in and out of the database is properly validated.

4. **Keep Queries Simple**: Break complex queries into smaller, more manageable functions.

5. **Use Transactions**: For operations that modify multiple tables, use transactions to ensure data integrity.

6. **Consistent Naming**: Follow the established naming conventions for functions and types:
   - `readOneX` - Read a single entity
   - `readManyX` - Read multiple entities
   - `createX` - Create an entity
   - `updateX` - Update an entity
   - `deleteX` - Delete an entity
   - `rawXToY` - Convert a raw database entity to a domain entity

7. **Handle Relationships**: When entities have relationships, use the conversion functions to resolve these relationships.

## Examples

### Example: Reading a User

```typescript
import { readOneUser } from "back-end/lib/db";
import { Connection } from "back-end/lib/db";

async function getUserById(connection: Connection, id: string) {
  const userResult = await readOneUser(connection, id);
  if (userResult.tag === "invalid") {
    // Handle error
    return null;
  }
  return userResult.value;
}
```

### Example: Creating an Organization

```typescript
import { createOrganization } from "back-end/lib/db";
import { Connection } from "back-end/lib/db";

async function createNewOrganization(connection: Connection, name: string) {
  const orgResult = await createOrganization(connection, { name });
  if (orgResult.tag === "invalid") {
    // Handle error
    return null;
  }
  return orgResult.value;
}
```

## Debugging

When encountering database issues:

1. Check the logs for detailed error messages
2. Use the raw SQL queries to verify database operations
3. Verify data types match between application and database
4. Ensure database migrations are up to date

## Conclusion

The database module provides a consistent, type-safe interface for all database operations in the Digital Marketplace application. By following the patterns established in this module, you can ensure that database operations are reliable, maintainable, and consistent across the application.
