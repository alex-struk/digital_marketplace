# Database Migrations Documentation

## Overview

Database migrations are a way to manage and version control database schema changes over time. They allow for seamless evolution of the database structure as the application grows and changes, providing a clear history of all modifications made to the database schema.

This documentation provides details about how migrations work in this project, how to create and run them, and best practices to follow when working with migrations.

## Architecture

The migration system in this project is built using [Knex.js](http://knexjs.org/), a SQL query builder for Node.js that provides a flexible interface for creating and running database migrations. Migrations are written in TypeScript, allowing for type safety and better integration with the rest of the codebase.

### Directory Structure

```
src/migrations/
├── tasks/               # Contains all migration files
│   ├── YYYYMMDDHHMMSS_name.ts  # Individual migration files
├── knexfile.ts          # Knex configuration for migrations
├── stub.ts              # Template file for new migrations
└── tsconfig.json        # TypeScript configuration for migrations
```

### Migration Files

Migration files are located in the `src/migrations/tasks` directory and follow a specific naming convention:

```
YYYYMMDDHHMMSS_description.ts
```

Where:
- `YYYYMMDDHHMMSS` is a timestamp (year, month, day, hour, minute, second)
- `description` is a brief description of what the migration does

Each migration file exports two async functions:

1. **up**: Defines the changes to be applied to the database.
2. **down**: Defines how to revert the changes applied by the "up" function.

Example migration file:

```typescript
import { makeDomainLogger } from "back-end/lib/logger";
import { console as consoleAdapter } from "back-end/lib/logger/adapters";
import { Knex } from "knex";

const logger = makeDomainLogger(consoleAdapter, "migrations");

export async function up(connection: Knex): Promise<void> {
  // Create a new table
  await connection.schema.createTable("example_table", (table) => {
    table.uuid("id").primary().unique().notNullable();
    table.string("name").notNullable();
    table.timestamp("createdAt").notNullable();
  });
  logger.info("Created example_table");
}

export async function down(connection: Knex): Promise<void> {
  // Drop the table created in the 'up' function
  await connection.schema.dropTable("example_table");
}
```

### Configuration

The `knexfile.ts` contains the configuration for the Knex migration system:

```typescript
import { DB_MIGRATIONS_TABLE_NAME, PG_CONFIG } from "back-end/config";

module.exports = {
  client: "pg",
  connection: PG_CONFIG,
  migrations: {
    tableName: DB_MIGRATIONS_TABLE_NAME,
    directory: "./tasks"
  }
};
```

Key configuration points:
- `client`: The database client (PostgreSQL in this case)
- `connection`: Database connection details
- `migrations.tableName`: The name of the table that tracks migrations
- `migrations.directory`: Where the migration files are located

## Using Migrations

### Creating a New Migration

To create a new migration, you'll need to create a new file in the `src/migrations/tasks` directory with the appropriate timestamp and a descriptive name. You can use the `stub.ts` file as a template.

The migration filename should follow the pattern: `YYYYMMDDHHMMSS_description.ts`

### Writing Migrations

When writing migrations, follow these guidelines:

1. **Idempotent Operations**: Migrations should be idempotent, meaning they can be run multiple times without causing errors or changing the final result.

2. **Up and Down Functions**: Always implement both the `up` and `down` functions. The `up` function applies the changes, and the `down` function reverts them.

3. **Atomic Changes**: Each migration should make a small, atomic change to the database. This makes it easier to identify and fix issues.

4. **Logging**: Use the logger to log important actions for better debugging and visibility.

5. **Forward Compatibility**: Ensure migrations are compatible with future changes by avoiding hard-coded values or assumptions about the database structure.

### Common Migration Operations

#### Creating Tables

```typescript
await connection.schema.createTable("table_name", (table) => {
  table.uuid("id").primary().unique().notNullable();
  table.string("name").notNullable();
  table.timestamp("createdAt").notNullable();
  table.timestamp("updatedAt").notNullable();

  // Add foreign keys
  table.uuid("userId").references("id").inTable("users");

  // Add indices
  table.index(["name"]);
});
```

#### Altering Tables

```typescript
// Add a column
await connection.schema.alterTable("table_name", (table) => {
  table.string("new_column");
});

// Modify a column
await connection.schema.alterTable("table_name", (table) => {
  table.string("column_name", 100).alter();
});

// Drop a column
await connection.schema.alterTable("table_name", (table) => {
  table.dropColumn("column_name");
});
```

#### Creating Indices

```typescript
await connection.schema.alterTable("table_name", (table) => {
  table.index(["column_name"]);
});
```

#### Adding Foreign Keys

```typescript
await connection.schema.alterTable("table_name", (table) => {
  table.uuid("foreign_id").references("id").inTable("foreign_table");
});
```

#### Seeding Data

```typescript
await connection("table_name").insert([
  { id: "uuid-1", name: "Example 1", createdAt: new Date() },
  { id: "uuid-2", name: "Example 2", createdAt: new Date() }
]);
```

### Running Migrations

Migrations are typically run using Knex's CLI commands:

```bash
# Run all pending migrations
npx knex migrate:latest --knexfile src/migrations/knexfile.ts

# Rollback the most recent migration
npx knex migrate:rollback --knexfile src/migrations/knexfile.ts

# Rollback all migrations
npx knex migrate:rollback --all --knexfile src/migrations/knexfile.ts

# Run a specific migration
npx knex migrate:up <migration_name> --knexfile src/migrations/knexfile.ts

# Rollback a specific migration
npx knex migrate:down <migration_name> --knexfile src/migrations/knexfile.ts
```

## Best Practices

1. **Never Modify Existing Migrations**: Once a migration has been applied to any environment (especially production), never modify it. Instead, create a new migration to make additional changes.

2. **Use Descriptive Names**: Give your migrations descriptive names that clearly indicate what they do.

3. **Test Migrations**: Test migrations thoroughly, including both the `up` and `down` functions, before applying them to production.

4. **Keep Migrations Small**: Make each migration perform a single logical change to the database for easier debugging and rollbacks.

5. **Document Complex Migrations**: Add comments to explain complex operations, especially for migrations that involve data transformations.

6. **Use Transactions**: Wrap complex operations in transactions to ensure database integrity.

   ```typescript
   export async function up(connection: Knex): Promise<void> {
     await connection.transaction(async (trx) => {
       // Operations within a transaction
     });
   }
   ```

7. **Maintain Backwards Compatibility**: Ensure that migrations don't break existing code by maintaining backwards compatibility where necessary.

## Troubleshooting

### Common Issues

1. **Migration Lock**: If a migration process is interrupted, the migration lock may remain, preventing future migrations from running. You can release the lock using:

   ```bash
   npx knex migrate:unlock --knexfile src/migrations/knexfile.ts
   ```

2. **Failed Migrations**: If a migration fails, fix the issue and try running it again. If it continues to fail, you may need to manually fix the database and mark the migration as completed.

3. **Migration Order**: Migrations run in the order of their timestamps. If you have migrations that depend on others, ensure their timestamps reflect the correct order.

## Migration History

The system maintains a record of all migrations that have been applied in the database table specified by `DB_MIGRATIONS_TABLE_NAME` in the configuration. This table includes:

- The name of the migration
- The batch number
- The timestamp when it was run

This information is useful for debugging and for understanding the current state of the database schema.

## Conclusion

Database migrations are a powerful tool for managing database schema changes over time. By following the guidelines in this documentation, you can ensure that your database evolves smoothly alongside your application code, with a clear history of all changes and the ability to roll back if necessary.
