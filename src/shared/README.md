# Shared Module Documentation

## Overview
The shared module is a core component of the Digital Marketplace application that provides common functionality, types, and utilities used across both the frontend (app) and backend (api) projects. This module is designed to maintain consistency and reduce code duplication between the two main parts of the application.

## Structure

```
src/shared/
├── config.ts           # Application-wide configuration constants
├── lib/               # Core library functionality
│   ├── data/         # Data-related utilities and types
│   ├── http.ts       # HTTP request handling and validation
│   ├── index.ts      # Main entry point and exports
│   ├── resources/    # Resource-related types and utilities
│   ├── types.ts      # Common TypeScript types and utilities
│   └── validation/   # Validation utilities and types
└── tsconfig.json     # TypeScript configuration for the shared module
```

## Key Components

### Configuration (`config.ts`)
Contains application-wide configuration constants including:
- Environment variable parsing utilities
- System-wide constants (budgets, timezone, etc.)
- Copy text and branding information
- Default values for pagination and display

### HTTP Utilities (`lib/http.ts`)
Provides a robust HTTP client implementation with:
- Type-safe request/response handling
- Response validation utilities
- Error handling and parsing
- Request function abstraction
- URL prefixing capabilities

### Type System (`lib/types.ts`)
Implements a comprehensive type system including:
- Algebraic Data Types (ADT) implementation
- Common type utilities (Id, Defined, IfElse, etc.)
- HTTP method enums
- Response body types and interfaces
- Validation error types

## Usage in App and API Projects

### Integration
The shared module is integrated into both the app and api projects through TypeScript's module resolution system. It's referenced in their respective `tsconfig.json` files and imported using the `shared` path alias.

### Example Usage

```typescript
// In app or api code
import { ADT, adt } from "shared/lib/types";
import { request } from "shared/lib/http";
import { CWU_MAX_BUDGET } from "shared/config";

// Using ADTs
type Status = ADT<"active"> | ADT<"inactive">;
const status: Status = adt("active");

// Making HTTP requests
const response = await request("GET", "/api/endpoint");

// Using configuration
if (budget > CWU_MAX_BUDGET) {
  // Handle budget limit
}
```

## Key Features

### 1. Type Safety
- Comprehensive TypeScript types and interfaces
- Algebraic Data Types for type-safe state management
- Generic type utilities for better type inference

### 2. HTTP Handling
- Type-safe HTTP request/response handling
- Built-in error handling and validation
- Configurable request prefixing
- Response validation utilities

### 3. Configuration Management
- Centralized configuration
- Environment variable parsing
- System-wide constants
- Branding and copy text management

### 4. Validation
- Response validation utilities
- Type-safe validation results
- Error handling and parsing

## Best Practices

1. **Type Usage**
   - Always use the provided type utilities when possible
   - Leverage ADTs for state management
   - Use the predefined interfaces for consistency

2. **HTTP Requests**
   - Use the `request` function for all HTTP calls
   - Implement proper error handling
   - Validate responses using the provided utilities

3. **Configuration**
   - Use the centralized configuration constants
   - Add new configuration values to `config.ts`
   - Document any new configuration additions

## Development Guidelines

1. **Adding New Features**
   - Place new shared functionality in appropriate subdirectories
   - Export through `lib/index.ts`
   - Add proper TypeScript types
   - Include documentation

2. **Modifying Existing Code**
   - Maintain backward compatibility
   - Update documentation
   - Consider impact on both app and api projects

3. **Testing**
   - Ensure changes are tested in both app and api contexts
   - Maintain type safety
   - Verify HTTP functionality

## Dependencies
The shared module has minimal external dependencies to maintain its lightweight nature:
- axios (for HTTP requests)
- TypeScript

## Version Control
The shared module is versioned alongside the main application. Changes to this module should be carefully considered as they affect both the app and api projects.
