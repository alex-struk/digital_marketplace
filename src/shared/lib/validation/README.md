# Validation Module Documentation

## Overview

The validation module in the Digital Marketplace application provides a robust, type-safe approach to form validation, data validation, and error handling. It implements a functional programming pattern using Algebraic Data Types (ADTs) to represent valid and invalid states.

This documentation covers the core validation functionality, available validators, and how to effectively use them throughout the application.

## Core Concepts

### Validation Types

The validation module is built around the concept of `Validation<A, B>` which is a union type of either:
- `Valid<A>`: Representing a successful validation with a value of type A
- `Invalid<B>`: Representing a failed validation with error information of type B (by default string arrays)

```typescript
export type Valid<T> = ADT<"valid", T>;
export type Invalid<T> = ADT<"invalid", T>;
export type Validation<A, B = string[]> = Valid<A> | Invalid<B>;
```

### Key Functions

#### Creating Validation Results

```typescript
// Create a valid validation result
export function valid<T>(value: T): Valid<T>

// Create an invalid validation result
export function invalid<T>(value: T): Invalid<T>
```

#### Type Guards

```typescript
// Check if a validation result is valid
export function isValid<Valid>(value: Validation<Valid, any>): value is ADT<"valid", Valid>

// Check if a validation result is invalid
export function isInvalid<Invalid>(value: Validation<any, Invalid>): value is ADT<"invalid", Invalid>

// Check if all validation results are valid
export function allValid(results: Array<Validation<any, any>>): results is Array<ADT<"valid", any>>

// Check if all validation results are invalid
export function allInvalid(results: Array<Validation<any, any>>): results is Array<ADT<"invalid", any>>
```

#### Extracting Values

```typescript
// Get the value from a valid validation result, or fallback if invalid
export function getValidValue<Valid, Fallback = Valid>(
  result: Validation<Valid, any>,
  fallback: Fallback
): Valid | Fallback

// Get the error from an invalid validation result, or fallback if valid
export function getInvalidValue<Invalid, Fallback = Invalid>(
  result: Validation<any, Invalid>,
  fallback: Fallback
): Invalid | Fallback
```

#### Transforming Validation Results

```typescript
// Transform the value of a valid validation result
export function mapValid<A, B, C>(
  value: Validation<A, B>,
  fn: (b: A) => C
): Validation<C, B>

// Transform the error of an invalid validation result
export function mapInvalid<A, B, C>(
  value: Validation<A, B>,
  fn: (b: B) => C
): Validation<A, C>
```

### Optional Values

The validation module provides functions to handle optional values:

```typescript
// Validate a field only if it is truthy
export function optional<Value, Valid, Invalid>(
  v: Value | OptionalNotDefined,
  validate: (v: Value) => Validation<Valid, Invalid>
): Validation<Valid | undefined, Invalid>

// Async version of optional
export async function optionalAsync<Value, Valid, Invalid>(
  v: Value | OptionalNotDefined,
  validate: (v: Value) => Promise<Validation<Valid, Invalid>>
): Promise<Validation<Valid | undefined, Invalid>>
```

## Common Validators

### String Validation

```typescript
// Validate a string with min/max length
export function validateGenericString(
  value: string,
  name: string,
  min = 1,
  max = 100,
  characters = "characters"
): Validation<string>

// Validate a string with min/max word count
export function validateGenericStringWords(
  value: string,
  name: string,
  min = 1,
  max = 3000,
  words = "words"
): Validation<string>

// Validate a string against a set of allowed values
export function validateStringInArray(
  value: string,
  availableValues: immutable.Set<string>,
  name: string,
  indefiniteArticle = "a",
  caseSensitive = false
): Validation<string>

// Validate an email address
export function validateEmail(email: string): Validation<string>

// Validate a URL
export function validateUrl(url: string): Validation<string>

// Validate a phone number
export function validatePhoneNumber(phone: string): Validation<string>
```

### Number Validation

```typescript
// Validate a number with optional min/max
export function validateNumber(
  raw: string | number,
  min?: number,
  max?: number,
  name = "number",
  article = "a",
  format = true,
  integer = true
): Validation<number>

// Validate a number with precision constraints
export function validateNumberWithPrecision(
  raw: string | number,
  min?: number,
  max?: number,
  maxPrecision = 5,
  name = "number",
  article = "a",
  format = true
): Validation<number>
```

### Date Validation

```typescript
// Validate a date with min/max constraints
export function validateGenericDate(
  raw: string,
  name: string,
  preposition: string,
  format: (d: Date) => string,
  minDate?: Date,
  maxDate?: Date,
  modifyParsed?: (_: Date) => Date
): Validation<Date>

// Validate a date and time
export function validateDatetime(
  raw: string,
  minDate?: Date,
  maxDate?: Date,
  modifyParsed?: (_: Date) => Date
): Validation<Date>

// Validate a date only
export function validateDate(
  raw: string,
  minDate?: Date,
  maxDate?: Date,
  modifyParsed?: (_: Date) => Date
): Validation<Date>

// Validate a time only
export function validateTime(
  raw: string,
  minDate?: Date,
  maxDate?: Date
): Validation<Date>
```

### ID Validation

```typescript
// Validate a UUID
export function validateUUID(raw: string): Validation<Id>
```

### Array Validation

The validation module provides specialized functions for validating arrays of values:

```typescript
// Validate an array of values with a custom validator
export function validateArrayCustom<A, B, C>(
  raw: A[],
  validate: (v: A) => Validation<B, C>,
  defaultInvalidValue: C
): ArrayValidation<B, C>

// Validate an array with string[] errors
export function validateArray<A, B>(
  raw: A[],
  validate: (v: A) => Validation<B>
): ArrayValidation<B>

// Async versions for async validation
export async function validateArrayAsync<A, B>(
  raw: A[],
  validate: (v: A) => Promise<Validation<B>>
): Promise<ArrayValidation<B>>

export async function validateArrayCustomAsync<A, B, C>(
  raw: A[],
  validate: (v: A) => Promise<Validation<B, C>>,
  defaultInvalidValue: C
): Promise<ArrayValidation<B, C>>
```

## Domain-Specific Validation

The validation module includes specialized validation functions for different domains within the application:

### File Validation (`file.ts`)

Functions for validating files and file permissions:

```typescript
// Validate avatar filenames
export function validateAvatarFilename(name: string): Validation<string>

// Validate filenames with extension checking
export function validateFileName(
  name: string,
  validExtensions: readonly string[] = []
): Validation<string>

// Validate file permissions
export function validateFilePermissions(
  raw: any
): ArrayValidation<FilePermissions<Id, UserType>>

// Validate a single file permission
export function validateFilePermission(
  raw: any
): Validation<FilePermissions<Id, UserType>>
```

### User Validation (`user.ts`)

Functions for validating user-related data:

```typescript
// Validate user capabilities
export function validateCapabilities(raw: string[]): ArrayValidation<string>

// Validate user name
export function validateName(name: string): Validation<string>

// Validate job title
export function validateJobTitle(v: string): Validation<string>

// Validate user type
export function validateUserType(type: string): Validation<UserType>

// Validate user status
export function validateUserStatus(status: string): Validation<UserStatus>
```

### Organization Validation (`organization.ts`)

Functions for validating organization data, including legal name, website, etc.

### Proposal Validation (`proposal/`)

Specialized validation for different proposal types in the application, including:
- Code With Us proposals
- Sprint With Us proposals

### Opportunity Validation (`opportunity/`)

Validation for different opportunity types in the application, including:
- Code With Us opportunities
- Sprint With Us opportunities
- Team With Us opportunities

## Integration with Components

The validation module integrates seamlessly with the application's component system. Here's how it's typically used:

### Form Field Initialization

```typescript
const [fieldState, fieldCmds] = Field.init({
  errors: [],
  validate: (v) => {
    // Use validation functions
    return validateGenericString(v, "Field name", 1, 100);
  },
  child: {
    value: initialValue,
    id: "field-id"
  }
});
```

### Form Validation

```typescript
export function validate(state: Immutable<State>): Immutable<State> {
  return state
    .update("field1", (s) => FormField.validate(s))
    .update("field2", (s) => FormField.validate(s))
    // Validate all fields in the form
    .update("complexField", (s) => ComplexField.validate(s));
}
```

### Example Usage in Components

```typescript
// Number field with validation
const [budgetField, budgetCmds] = NumberField.init({
  errors: [],
  validate: (v) => {
    if (v === null) {
      return invalid(["Please enter a valid budget."]);
    }
    return validateNumber(v, 0, 1000000, "budget", "a");
  },
  child: {
    value: initialBudget,
    id: "opportunity-budget",
    min: 0
  }
});

// Text field with validation
const [titleField, titleCmds] = TextField.init({
  errors: [],
  validate: (v) => validateGenericString(v, "Title", 1, 100),
  child: {
    value: initialTitle,
    id: "opportunity-title"
  }
});
```

## Error Handling

The validation module's design provides a consistent approach to error handling:

1. Each validation function returns a `Validation<A, B>` type
2. Invalid results contain arrays of error messages
3. For complex objects, error structure matches input structure
4. Errors can be extracted and displayed in UI components

## Best Practices

1. **Composition**: Combine validation functions for complex validation
2. **Reuse**: Use existing validators where possible
3. **Type Safety**: Leverage TypeScript to ensure validation results match expected types
4. **Error Messages**: Provide clear, user-friendly error messages
5. **Validation Location**: Validate both client-side and server-side

## Summary

The validation module provides a comprehensive, type-safe approach to data validation throughout the Digital Marketplace application. Its functional design makes it easy to compose validators, transform results, and handle errors in a consistent way.

By using the validation module, developers can ensure that data is validated consistently across the application, leading to better user experience and data integrity.
