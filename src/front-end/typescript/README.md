# Digital Marketplace Front-End TypeScript Module

## Overview

The Front-End TypeScript module is the core foundation of the Digital Marketplace application's client-side architecture. It implements a unidirectional data flow pattern inspired by The Elm Architecture (TEA), using TypeScript and React to provide a robust, type-safe, and maintainable front-end framework.

## Core Principles

- **Unidirectional Data Flow**: One-way data flow makes application state predictable and easier to debug
- **Type Safety**: Strong TypeScript typing throughout the codebase to prevent runtime errors
- **Immutable State**: State is treated as immutable to prevent unexpected mutations
- **Component-Based Architecture**: UI built from composable, reusable components
- **Message-Driven Updates**: State changes driven by explicit messages
- **Separation of Concerns**: Clear separation between state, view, and update logic

## Directory Structure

```
src/front-end/typescript/
├── lib/                    # Core framework and utilities
│   ├── app/                # Main application definition
│   ├── components/         # Reusable UI components
│   ├── framework/          # The custom TEA-inspired framework
│   ├── http/               # HTTP client utilities
│   ├── interfaces/         # TypeScript interfaces
│   ├── pages/              # Page components
│   └── views/              # View components
├── config.ts               # Application configuration
├── index.ts                # Main entry point
└── tsconfig.json           # TypeScript configuration
```

## Getting Started

The application is initialized in `index.ts`, which serves as the entry point. The file:

1. Imports the necessary configurations and framework components
2. Initializes the application process
3. Connects the application to the DOM
4. Sets up event listeners (e.g., for window resize)

## Key Components

### Configuration (config.ts)

Contains all configuration settings for the application, including:

- Environment variables
- Application URLs and paths
- Default values
- Timing constants
- Content IDs
- Cost figures

### Framework

A custom-built framework inspired by The Elm Architecture that provides:

- Component model for building UI elements
- Immutable state management
- Message-based state updates
- Routing system
- Command abstractions for side effects

### Application (lib/app)

Defines the root application component that:

- Initializes the application state
- Handles routing
- Manages global state
- Renders the current page

## Architecture

### Component Model

Components follow a standard interface with three main functions:

1. **init**: Initializes component state
2. **update**: Handles messages and updates state
3. **view**: Renders the component based on state

```typescript
interface Component<Params, State, Msg> {
  init: Init<Params, State, Msg>;
  update: Update<State, Msg>;
  view: View<State, Msg>;
}
```

### State Management

All state is immutable and updated through explicit update functions:

```typescript
type Update<State, Msg> = (params: { state: Immutable<State>, msg: Msg }) => [Immutable<State>, Cmd<Msg>[]];
```

Update functions return:
- A new immutable state
- An array of commands to be executed (side effects)

### Routing

The router manages navigation within the application:

- Maps URLs to route objects
- Handles route transitions
- Manages browser history
- Initializes pages based on routes

### Commands (Cmd)

Commands represent side effects like:

- API calls
- Local storage operations
- Navigation actions
- DOM manipulations

## Utility Functions

The module provides various utility functions to help with:

- Loading state management (`makeStartLoading`, `makeStopLoading`)
- Validation handling (`withValid`, `updateValid`, etc.)
- Path prefixing for routes
- File handling
- UI component composition

## Integration with Back-End

The module communicates with the back-end through:

- HTTP requests wrapped in commands
- JSON serialization/deserialization
- Typed API interfaces

## Available Helper Components

The framework includes reusable components for common UI patterns:

- Form elements
- Alerts and notifications
- Modal dialogs
- Navigation elements
- Loading indicators
- Breadcrumbs

## Best Practices

When working with this module:

1. **Maintain Immutability**: Never directly mutate state
2. **Use Typed Messages**: Define explicit types for all messages
3. **Keep Components Small**: Follow single responsibility principle
4. **Leverage Command Pattern**: Use commands for all side effects
5. **Follow Naming Conventions**: Match existing component naming patterns
6. **Reuse Existing Components**: Check for existing components before creating new ones
7. **Add Type Definitions**: Ensure all code has proper TypeScript typing

## Example Usage

A typical component definition:

```typescript
// Define message type
type Msg
  = ADT<'incrementCounter'>
  | ADT<'setName', string>;

// Define state type
interface State {
  counter: number;
  name: string;
}

// Initialize state
const init: component.base.Init<null, State, Msg> = () => {
  return [
    immutable({
      counter: 0,
      name: ''
    }),
    []
  ];
};

// Update function
const update: component.base.Update<State, Msg> = ({ state, msg }) => {
  switch (msg.tag) {
    case 'incrementCounter':
      return [
        state.set('counter', state.counter + 1),
        []
      ];
    case 'setName':
      return [
        state.set('name', msg.value),
        []
      ];
  }
};

// View function
const view: component.base.View<State, Msg> = ({ state, dispatch }) => {
  return (
    <div>
      <h1>Counter: {state.counter}</h1>
      <button onClick={() => dispatch(adt('incrementCounter'))}>Increment</button>
      <input
        value={state.name}
        onChange={e => dispatch(adt('setName', e.target.value))}
      />
    </div>
  );
};

// Component definition
const MyComponent: component.base.Component<null, State, Msg> = {
  init,
  update,
  view
};
```

## Advanced Features

### Nested Components

Components can be nested and composed:

```typescript
// Parent update function
const update: component.base.Update<ParentState, ParentMsg> = ({ state, msg }) => {
  switch (msg.tag) {
    case 'childMsg':
      return updateChild({
        state,
        childStatePath: ['child'],
        childUpdate: ChildComponent.update,
        childMsg: msg.value,
        mapChildMsg: (msg) => adt('childMsg', msg)
      });
  }
};
```

### Routing

Defining routes:

```typescript
const routes: router.Route[] = [
  {
    path: '/dashboard',
    makeRoute: () => adt('dashboard' as const)
  },
  {
    path: '/profile/:id',
    makeRoute: ({ params }) => adt('profile' as const, params.id)
  }
];
```

## Troubleshooting

Common issues:

1. **Immutability Errors**: Ensure you're using state setters instead of direct mutation
2. **Type Errors**: Make sure all message handlers account for all message types
3. **Component Rendering Issues**: Verify the component lifecycle and state updates
4. **Command Execution Problems**: Check that commands are returning expected message types

## Further Reading

For more detailed documentation, refer to the framework documentation files:
- `/src/front-end/typescript/lib/framework/README.md`
- `/src/front-end/typescript/lib/framework/DEVELOPER_GUIDE.md`
- `/src/front-end/typescript/lib/framework/DEBUGGING_GUIDE.md`
