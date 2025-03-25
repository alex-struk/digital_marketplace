# Algebraic Data Types (ADTs) in Digital Marketplace

This document explains the implementation and usage of Algebraic Data Types (ADTs) in the Digital Marketplace application, providing background, technical details, and examples.

## Table of Contents

- [Introduction to ADTs](#introduction-to-adts)
- [Why ADTs Are Valuable: A Concrete Example](#why-adts-are-valuable-a-concrete-example)
- [Implementation in TypeScript](#implementation-in-typescript)
- [Practical Applications](#practical-applications)
  - [Message Handling (Msg Types)](#message-handling-msg-types)
  - [API Communication](#api-communication)
  - [UI and Navigation](#ui-and-navigation)
  - [Pattern Matching with Switch Statements](#pattern-matching-with-switch-statements)
- [Benefits of ADTs](#benefits-of-adts)
- [Common Patterns and Best Practices](#common-patterns-and-best-practices)
- [Troubleshooting](#troubleshooting)
- [Further Reading](#further-reading)

## Introduction to ADTs

Algebraic Data Types (ADTs) are composite types commonly used in functional programming languages. They allow developers to define a new type along with a set of value constructors for that type. ADTs are particularly useful for modeling domain concepts that can take multiple forms but are conceptually related.

There are primarily two kinds of algebraic types:

1. **Sum Types** (also known as tagged unions or discriminated unions): Represent values that could be one of several alternatives.
2. **Product Types**: Represent values that contain multiple fields simultaneously (like tuples or records).

In functional programming languages like Haskell, ADTs are defined natively:

```haskell
data Color
  = Red
  | Green
  | Blue
  | RGB Int Int Int

foreground :: Color
foreground = Red

white :: Color
white = RGB 255 255 255
```

## Why ADTs Are Valuable: A Concrete Example

Let's explore a concrete example that clearly demonstrates the value of ADTs in real-world application development.

### Form Validation Without ADTs

Consider a form validation system. Without ADTs, we might represent validation results like this:

```typescript
// Without ADTs
interface ValidationResult {
  isValid: boolean;
  errorMessage?: string;
  validatedValue?: any;
  errorCode?: number;
  fieldName?: string;
}

function validateEmail(email: string): ValidationResult {
  if (!email.includes('@')) {
    return {
      isValid: false,
      errorMessage: 'Invalid email format',
      fieldName: 'email'
    };
  }
  return {
    isValid: true,
    validatedValue: email.toLowerCase()
  };
}

function processValidation(result: ValidationResult) {
  if (result.isValid) {
    console.log(`Valid: ${result.validatedValue}`);
    // But wait, validatedValue might be undefined even though isValid is true!
    // TypeScript won't catch this potential error
  } else {
    console.log(`Error in ${result.fieldName}: ${result.errorMessage}`);
    // Again, these properties might be undefined
  }
}
```

This traditional approach has several problems:

1. **Optional Properties**: We're using optional properties that may or may not be present, creating possible runtime errors.
2. **Type Unsafety**: TypeScript doesn't enforce which properties must exist based on `isValid`.
3. **Invalid States**: We could accidentally access `errorMessage` when `isValid` is true.
4. **Lost Type Information**: The type of `validatedValue` is `any`, losing valuable type information.
5. **Runtime Errors**: These issues can lead to runtime errors that TypeScript can't catch at compile time.

### Form Validation With ADTs

Using ADTs, we can model this much more clearly and safely:

```typescript
// With ADTs
type Valid<T> = ADT<'valid', T>;
type Invalid = ADT<'invalid', {
  fieldName: string;
  errorMessage: string;
}>;

type ValidationResult<T> = Valid<T> | Invalid;

function validateEmail(email: string): ValidationResult<string> {
  if (!email.includes('@')) {
    return adt('invalid', {
      fieldName: 'email',
      errorMessage: 'Invalid email format'
    });
  }
  return adt('valid', email.toLowerCase());
}

function validateAge(age: string): ValidationResult<number> {
  const parsedAge = parseInt(age, 10);
  if (isNaN(parsedAge)) {
    return adt('invalid', {
      fieldName: 'age',
      errorMessage: 'Age must be a number'
    });
  }
  if (parsedAge < 18) {
    return adt('invalid', {
      fieldName: 'age',
      errorMessage: 'Must be 18 or older'
    });
  }
  return adt('valid', parsedAge);
}

function processValidation<T>(result: ValidationResult<T>) {
  switch (result.tag) {
    case 'valid':
      // TypeScript knows result.value is of type T here
      console.log(`Valid: ${result.value}`);
      return result.value; // We can safely use the validated value
    case 'invalid':
      // TypeScript knows we have fieldName and errorMessage here
      console.log(`Error in ${result.value.fieldName}: ${result.value.errorMessage}`);
      return null;
  }
}
```

### Composition of Validations

With ADTs, we can easily compose multiple validations:

```typescript
function validateForm(email: string, age: string): ValidationResult<{email: string, age: number}> {
  const emailResult = validateEmail(email);
  const ageResult = validateAge(age);

  if (emailResult.tag === 'invalid') return emailResult;
  if (ageResult.tag === 'invalid') return ageResult;

  // Both are valid, we can safely use their values
  return adt('valid', {
    email: emailResult.value,
    age: ageResult.value
  });
}

// Usage
const formResult = validateForm('user@example.com', '25');

switch (formResult.tag) {
  case 'valid':
    createUser(formResult.value.email, formResult.value.age);
    break;
  case 'invalid':
    showError(formResult.value.fieldName, formResult.value.errorMessage);
    break;
}
```

### Key Benefits Demonstrated

The ADT approach offers significant advantages:

1. **Type Preservation**: Notice how `validateEmail` returns `ValidationResult<string>` and `validateAge` returns `ValidationResult<number>`. The ADT preserves the type of the validated value.

2. **Impossible States Eliminated**: In the non-ADT version, we could have `isValid: true` but no `validatedValue`, which is an invalid state. With ADTs, this is impossible.

3. **Exhaustive Pattern Matching**: TypeScript forces us to handle both the valid and invalid cases in the switch statement.

4. **Composition**: We can easily chain validations together, as shown in the `validateForm` function.

5. **Type Safety in Pattern Matching**: When we check `result.tag === 'valid'`, TypeScript knows that `result.value` has the correct type for that branch.

This example demonstrates why ADTs are valuable in real-world applications. They provide a more robust way to handle cases where data can take one of several distinct forms, making your code safer and more maintainable by preventing invalid states and ensuring all cases are handled properly.

## Implementation in TypeScript

Since TypeScript doesn't natively support ADTs like Haskell or other functional languages, the Digital Marketplace application implements a "synthetic" version of ADTs using TypeScript interfaces and generics. This implementation is located in `src/shared/lib/types.ts`:

```typescript
export interface ADT<Tag, Value = undefined> {
  readonly tag: Tag;
  readonly value: Value;
}

export function adt<T>(tag: T, value?: undefined): ADT<T>;
export function adt<T, V>(tag: T, value: V): ADT<T, V>;
export function adt<T extends ADT<unknown, unknown>>(
  tag: T["tag"],
  value: T["value"]
): ADT<T["tag"], T["value"]> {
  return { tag, value };
}

export function adtCurried<T extends ADT<unknown, unknown>>(
  tag: T["tag"]
): (value: T["value"]) => ADT<T["tag"], T["value"]> {
  return (value) => adt(tag, value);
}
```

This implementation provides:

- An `ADT<Tag, Value>` interface that represents a tagged value
- An `adt()` function with various overloads to construct ADT values
- An `adtCurried()` function for curried creation of ADTs (useful in certain functional contexts)

Using these building blocks, we can model sum types like this:

```typescript
type Color =
  | ADT<"red">
  | ADT<"green">
  | ADT<"blue">
  | ADT<"rgb", [number, number, number]>;

const foreground: Color = adt("red");
const white: Color = adt("rgb", [255, 255, 255]);
```

## Practical Applications

ADTs are used extensively throughout the Digital Marketplace codebase to model various domain concepts in a type-safe manner. Here are some key applications:

### Message Handling (Msg Types)

One of the most common uses is for component messages in the Elm-inspired architecture:

```typescript
// Simple message type
export type Msg = ADT<"increment"> | ADT<"decrement">;

// Message with a payload
type InnerChildMsg = ADT<"onChange", boolean>;

// Complex nested message types
type Msg =
  | ADT<"toggleHelp">
  | ADT<"validate">
  | ADT<"child", ChildMsg<InnerMsg>>;
```

Each component defines its own message types, which represent all possible events that can trigger state changes.

### API Communication

ADTs are used to model API request and response bodies:

```typescript
export type JsonRequestBody = ADT<"json", JsonValue>;
export type TextRequestBody = ADT<"text", string>;
export type FileRequestBody<FileUploadMetadata> = ADT<"file", FormFile<FileUploadMetadata>>;

export type JsonResponseBody<Value = any> = ADT<"json", Value>;
export type HtmlResponseBody = ADT<"html", string>;
export type ErrorResponseBody = ADT<"error", ErrorValue>;
```

This approach allows the type system to ensure that the correct parsing and handling functions are used for each type of request/response.

### UI and Navigation

ADTs help model UI elements and navigation structures:

```typescript
// Navigation links
type RouteDest = ADT<"route", Route>;
type ExternalDest = ADT<"external", string>;
type EmailDest = ADT<"email", [string, string?]>;

// UI elements
type SidebarItem = ADT<"link", SidebarLink> | ADT<"heading", string>;

// Modal dialogs
type Modal<Msg> = ADT<"show", VisibleModal<Msg>> | ADT<"hide">;
```

### Pattern Matching with Switch Statements

The real power of ADTs comes with pattern matching. In TypeScript, this is achieved using switch statements:

```typescript
switch (msg.tag) {
  case "approveAffiliation":
    return [
      state
        .set("approveAffiliationLoading", msg.value.id)
        .set("showModal", null),
      [
        api.affiliations.update<Msg>()(
          msg.value.id,
          adt("approve"),
          (response) =>
            adt("onApproveAffiliationResponse", [
              api.isValid(response),
              msg.value
            ]) as Msg
        )
      ]
    ];

  case "onApproveAffiliationResponse":
    const [valid, affiliation] = msg.value;
    if (valid) {
      // Handle successful response
    } else {
      // Handle error
    }
    return [state, []];

  // ... other cases

  default:
    return [state, []];
}
```

The TypeScript compiler ensures type safety within each case branch, understanding that `msg.value` has the appropriate type based on the tag.

## Benefits of ADTs

Using ADTs in the Digital Marketplace application provides several advantages:

1. **Type Safety**: ADTs allow precise modeling of domain concepts, ensuring that invalid states are unrepresentable.

2. **Exhaustiveness Checking**: When pattern matching on an ADT, TypeScript can check that all cases are handled.

3. **Self-Documenting Code**: The ADT definitions serve as documentation for the possible states of a system.

4. **Functional Programming Style**: ADTs enable a more functional approach to programming, promoting immutability and pure functions.

5. **Testability**: Code that uses ADTs tends to be more testable, as each case can be tested separately.

## Common Patterns and Best Practices

1. **Define Message Types at the Top Level**:
   ```typescript
   export type Msg =
     | ADT<"toggleHelp">
     | ADT<"validate">
     | ADT<"child", ChildMsg>;
   ```

2. **Use Case-Specific Payload Types**:
   ```typescript
   // Instead of:
   type Msg = ADT<"setUser", any>;

   // Prefer:
   type Msg = ADT<"setUser", User>;
   ```

3. **Compose ADTs for Complex States**:
   ```typescript
   type Status = ADT<"active"> | ADT<"inactive">;
   type User = {
     id: string;
     name: string;
     status: Status;
   };
   ```

4. **Explicit Type Casting When Necessary**:
   ```typescript
   // If TypeScript can't infer the type:
   const msg = adt("setUser", user) as Msg;
   ```

5. **Use Pattern Matching for All Cases**:
   ```typescript
   // Always include a default case or handle all possibilities
   switch (status.tag) {
     case "active":
       return renderActiveUser(user);
     case "inactive":
       return renderInactiveUser(user);
     default:
       const exhaustiveCheck: never = status;
       throw new Error(`Unhandled status: ${exhaustiveCheck}`);
   }
   ```

## Troubleshooting

1. **Type Inference Issues**:
   If TypeScript can't infer the correct type, use an explicit type cast:
   ```typescript
   const color = adt("rgb", [255, 255, 255]) as Color;
   ```

2. **Exhaustiveness Checking**:
   To enforce handling all cases in a switch statement:
   ```typescript
   default:
     const exhaustiveCheck: never = msg; // TypeScript error if not all cases covered
     throw new Error(`Unhandled message: ${exhaustiveCheck}`);
   ```

3. **Complex Nested ADTs**:
   For deeply nested ADTs, consider using helper functions:
   ```typescript
   function createChildMsg(innerMsg: InnerMsg): Msg {
     return adt("child", adt("inner", innerMsg));
   }
   ```

## Further Reading

- [Wikipedia Article on ADTs](https://en.wikipedia.org/wiki/Algebraic_data_type)
- [Making Impossible States Impossible](https://elm-lang.org/docs/records#making-impossible-states-impossible)
- [Elm Guide](https://guide.elm-lang.org/) - For understanding the architectural principles
- [TypeScript Handbook: Discriminated Unions](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html#discriminated-unions)
- [Domain Modeling Made Functional](https://pragprog.com/titles/swdddf/domain-modeling-made-functional/) - Book on applying functional programming techniques
