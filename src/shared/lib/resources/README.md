# Resources Module Documentation

## Overview

The `resources` module is a core component of the Digital Marketplace platform, providing type definitions, interfaces, and utility functions for managing the platform's key data entities. It serves as the foundation for data modeling throughout the application, ensuring consistent representation of entities such as users, organizations, opportunities, and proposals.

## Purpose

This module's primary purpose is to:

1. Define TypeScript interfaces for all major data entities
2. Provide type validation and parsing functions
3. Establish relationships between different entities
4. Define state transitions for complex workflows
5. Support validation for request/response bodies in API calls
6. Offer utility functions for entity manipulation and state checking

## Module Structure

The resources module is organized into domain-specific files, each representing a major entity or group of related entities:

### Core Entities

- **user.ts**: User accounts, permissions, and profiles
- **organization.ts**: Business entities that can participate in the marketplace
- **file.ts**: File attachments and document management
- **session.ts**: User authentication and session management
- **addendum.ts**: Supplementary information for opportunities
- **affiliation.ts**: Relationships between users and organizations
- **metrics.ts**: System performance and usage statistics
- **counter.ts**: Counters for various system operations
- **content.ts**: Content management functionality
- **service-area.ts**: Service categories and classifications
- **email-notifications.tsx**: Email notification templates and configuration

### Opportunity Types

The `opportunity` subdirectory contains definitions for different types of opportunities:

- **code-with-us.ts**: Short-term, fixed-price coding tasks
- **sprint-with-us.ts**: Team-based development sprints
- **team-with-us.ts**: Long-term team augmentation opportunities

### Proposal Types

The `proposal` subdirectory contains definitions for proposal submissions related to each opportunity type.

## Key Concepts

### Entity Interfaces

Each major entity is defined as a TypeScript interface with properties representing its attributes. For example, the `User` interface defines properties like `id`, `name`, `email`, and `type`.

### Slim Variants

Many entities have "slim" variants (e.g., `UserSlim`, `OrganizationSlim`) that contain only essential properties for list views or references, reducing payload size in API responses.

### State Enums

Entities with complex lifecycles have state enums (e.g., `UserStatus`, `CWUOpportunityStatus`) that define all possible states and valid transitions between them.

### Request/Response Types

API request and response bodies are defined as interfaces, providing type safety for client-server communication.

### Validation Types

Each entity has associated validation error types for form validation and API error handling.

### Utility Functions

The module provides numerous utility functions for:
- Checking entity state (e.g., `isAdmin`, `isCWUOpportunityPublic`)
- State transitions (e.g., `isValidStatusChange`)
- Entity conversion (e.g., `userToUserSlim`)
- Permission checking (e.g., `canAddAddendumToCWUOpportunity`)

## State Transitions in Complex Workflows

The resources module implements a sophisticated system for managing state transitions in complex workflows:

### State Management Architecture

1. **State Enums**: Each entity with a lifecycle has a corresponding enum that defines all possible states. For example:
   ```typescript
   export enum CWUOpportunityStatus {
     Draft = "DRAFT",
     UnderReview = "UNDER_REVIEW",
     Published = "PUBLISHED",
     Evaluation = "EVALUATION",
     Awarded = "AWARDED",
     Suspended = "SUSPENDED",
     Canceled = "CANCELED"
   }
   ```

2. **Transition Validation Functions**: Functions that verify whether a transition from one state to another is allowed based on business rules:
   ```typescript
   export function isValidCWUOpportunityStatusChange(
     from: CWUOpportunityStatus,
     to: CWUOpportunityStatus,
     proposalDeadline: Date
   ): boolean {
     // Implementation of transition validation logic
   }
   ```

3. **State Guard Functions**: Functions that check whether an entity meets all requirements for a particular state transition:
   ```typescript
   export function canCWUOpportunityBeAwarded(
     opportunity: CWUOpportunity
   ): boolean {
     // Check if the opportunity has proposals, evaluation completed, etc.
   }
   ```

4. **History Tracking**: Most entities track their state change history in a dedicated array:
   ```typescript
   export interface CWUOpportunityHistoryRecord {
     id: Id;
     createdAt: Date;
     createdBy: UserSlim | null;
     type: ADT<"status", CWUOpportunityStatus> | ADT<"event", CWUOpportunityEvent>;
     note: string;
     attachments: FileRecord[];
   }
   ```

5. **Event-Based State Changes**: State changes can be triggered by events, which are also defined in enums:
   ```typescript
   export enum CWUOpportunityEvent {
     Edited = "EDITED",
     AddendumAdded = "ADDENDUM_ADDED",
     NoteAdded = "NOTE_ADDED"
   }
   ```

### Workflow Examples

#### Opportunity Lifecycle

1. An opportunity starts in `Draft` status
2. User submits for review → status changes to `UnderReview`
3. Admin approves → status changes to `Published`
4. Proposal deadline passes → status changes to `Evaluation`
5. Evaluation completes → status changes to `Awarded`

Each transition is validated using state-specific validation functions to ensure business rules are followed, such as:
- Draft opportunities can't be awarded
- Published opportunities can't return to Draft
- Awarded opportunities can't be republished

#### User Status Changes

1. A new user starts with status `Active`
2. User deactivates account → status changes to `InactiveByUser`
3. Admin can reactivate account → status changes back to `Active`
4. Admin can deactivate account → status changes to `InactiveByAdmin`

## Validation Architecture

The resources module implements a comprehensive validation system to ensure data integrity:

### Type-Level Validation

The module uses TypeScript's type system to enforce validation at compile time:

1. **Error Type Generation**: The `ErrorTypeFrom<T>` utility type automatically generates validation error types from entity types:
   ```typescript
   export type UpdateProfileValidationErrors = ErrorTypeFrom<UpdateProfileRequestBody>;
   ```

2. **Field-Level Error Types**: Each field in an entity has a corresponding error field that can contain validation messages:
   ```typescript
   // If CreateRequestBody has a 'title' field
   // CreateValidationErrors will have a 'title?: string[]' field
   ```

3. **ADT for Complex Validations**: Algebraic Data Types (ADT) are used to represent different types of validation errors:
   ```typescript
   type UpdateADTErrors =
     | ADT<"updateProfile", UpdateProfileValidationErrors>
     | ADT<"updateCapabilities", string[][]>
     | ADT<"acceptTerms", string[]>
     | ADT<"parseFailure">;
   ```

### Runtime Validation

1. **Parser Functions**: Functions that validate and parse raw input data:
   ```typescript
   export function parseCWUOpportunityStatus(
     raw: string
   ): CWUOpportunityStatus | null {
     return isCWUOpportunityStatus(raw) ? raw : null;
   }
   ```

2. **Validator Functions**: Functions that perform complex validation logic:
   ```typescript
   export function validateCWUOpportunityProposalDeadline(
     date: string,
     now: Date
   ): string[] {
     const parsed = new Date(date);
     const errors: string[] = [];

     if (isDateInThePast(parsed, now)) {
       errors.push("Proposal deadline must be in the future.");
     }

     return errors;
   }
   ```

3. **Request Body Validation**: Structured validation for API request bodies:
   ```typescript
   export interface CreateValidationErrors
     extends Omit<
       ErrorTypeFrom<CreateRequestBody> & BodyWithErrors,
       "skills" | "attachments"
     > {
     skills?: string[][];
     attachments?: string[][];
   }
   ```

4. **Cross-Field Validation**: Validation that depends on multiple fields:
   ```typescript
   function validateDateSequence(
     proposalDeadline: Date,
     assignmentDate: Date
   ): string[] {
     return assignmentDate <= proposalDeadline
       ? ["Assignment date must be after the proposal deadline."]
       : [];
   }
   ```

### Validation Integration

1. **Frontend Form Validation**: Validation types are used with form libraries to validate user input before submission.

2. **API Request Validation**: Backend routes use validation functions to ensure incoming requests meet requirements.

3. **Database Validation**: Database operations use validation functions to ensure data integrity.

4. **Error Response Standardization**: The `BodyWithErrors` interface provides a consistent structure for error responses:
   ```typescript
   export interface BodyWithErrors {
     errors?: string[];
   }
   ```

### Validation Best Practices

1. Always use the provided validation types and functions instead of implementing custom validation
2. When adding new fields, update the corresponding validation types
3. For complex validation logic, break down into smaller, reusable functions
4. Use parser functions to safely convert external data to internal types
5. For multi-step forms, implement validation at each step

## Integration with the Application

### Frontend Usage

On the frontend, these types are used for:
- Form definitions and validation
- State management (Redux store typing)
- Component props typing
- API client request/response typing

### Backend Usage

On the backend, these types are used for:
- Database schema validation
- API endpoint request/response validation
- Business logic and permission checks
- Service layer data processing

## Opportunity Types Explained

### Code With Us (CWU)

Code With Us opportunities are small, discrete pieces of work with a clear definition and fixed price. These are typically shorter-term engagements focused on specific deliverables.

Key attributes:
- Fixed reward amount
- Clear acceptance criteria
- Single developer assignment
- Shorter timeframes

### Sprint With Us (SWU)

Sprint With Us opportunities are more complex projects requiring a team with diverse skills. These are structured as agile sprints with multiple phases.

Key attributes:
- Multi-phase structure (Inception, Prototype, Implementation)
- Team-based proposals
- Skills matrix requirements
- Longer timeframes

### Team With Us (TWU)

Team With Us opportunities focus on team augmentation, where organizations provide skilled resources to work as part of government teams.

Key attributes:
- Resource-based pricing
- Service area qualifications
- Long-term engagements
- Team member profiles and qualifications

## Examples

### Entity Creation

```typescript
// Creating a user entity
const user: User = {
  id: "user-123",
  type: UserType.Vendor,
  status: UserStatus.Active,
  name: "Jane Doe",
  email: "jane@example.com",
  jobTitle: "Developer",
  avatarImageFile: null,
  notificationsOn: new Date(),
  acceptedTermsAt: new Date(),
  lastAcceptedTermsAt: new Date(),
  idpUsername: "janedoe",
  deactivatedOn: null,
  deactivatedBy: null,
  capabilities: ["node.js", "react"],
  idpId: "idp-123"
};
```

### State Checking

```typescript
// Check if a user has admin permissions
if (isAdmin(user)) {
  // Allow admin operations
}

// Check if an opportunity can accept proposals
if (isCWUOpportunityAcceptingProposals(opportunity)) {
  // Show proposal submission form
}
```

### Entity Transformation

```typescript
// Convert a full user entity to a slim version
const userSlim = userToUserSlim(user);
```

### State Transition Example

```typescript
// Attempt to award an opportunity
if (isValidStatusChange(
  opportunity.status,
  CWUOpportunityStatus.Awarded,
  currentUser
)) {
  // Perform the transition
  opportunity.status = CWUOpportunityStatus.Awarded;
  opportunity.history.push({
    id: generateId(),
    createdAt: new Date(),
    createdBy: currentUser,
    type: { tag: "status", value: CWUOpportunityStatus.Awarded },
    note: "Opportunity awarded to selected proposal"
  });
}
```

### Validation Example

```typescript
// Validate an opportunity creation request
const validationErrors = validateCWUOpportunityCreation(requestBody, currentDate);

if (validationErrors.title?.length || validationErrors.proposalDeadline?.length) {
  // Handle validation errors
  return {
    valid: false,
    errors: validationErrors
  };
}
```

## Best Practices

1. Always use the provided types for entity definitions
2. Use utility functions for state checks rather than implementing custom logic
3. Leverage validation types for form error handling
4. Use state transition functions to ensure valid state changes
5. Reference slim variants when full entity data isn't needed
6. Follow the patterns established for extending or modifying entity types

## Related Modules

- **types**: Core type definitions and utilities used across the application
- **validation**: Form and API validation utilities
- **api**: API client implementation using resource types
- **hooks**: React hooks for working with resources
- **redux**: State management for resources

## Contribution Guidelines

When extending or modifying the resources module:

1. Maintain backward compatibility when possible
2. Add appropriate validation types for new properties
3. Include utility functions for common operations
4. Update related components and services
5. Add appropriate documentation
6. Ensure consistent naming conventions
