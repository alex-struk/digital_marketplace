# Custom Front-End Framework Documentation

## Overview

This document provides comprehensive documentation for the custom front-end framework used in this application. The framework is a unidirectional data flow architecture inspired by The Elm Architecture (TEA), implemented in TypeScript and React. It emphasizes type safety, component composability, and separation of concerns.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Architecture](#architecture)
- [Components](#components)
- [State Management](#state-management)
- [Routing](#routing)
- [Commands (Cmd)](#commands-cmd)
- [Building Pages](#building-pages)
- [Building Applications](#building-applications)
- [Best Practices](#best-practices)

## Core Concepts

The framework is built around the following core concepts:

1. **Unidirectional Data Flow**: Data flows in one direction, making the application state easier to reason about and debug.
2. **Type Safety**: Extensive use of TypeScript to ensure type safety across the application.
3. **Immutable State**: Application state is immutable, preventing unexpected mutations.
4. **Component-Based Architecture**: The UI is composed of reusable, composable components.
5. **Message-Driven Updates**: State changes are driven by messages dispatched through the system.
6. **Separation of Concerns**: Clear separation between state, view, and update logic.

## Architecture

The framework is structured around three primary patterns:

1. **Model (State)**: The complete state of the application, represented as immutable records.
2. **View**: Pure functions that render the UI based on the current state.
3. **Update**: Functions that take the current state and a message, and return a new state and commands.

### Core Pattern Implementations

Each component in the system implements:

- **init**: Function that initializes the component's state.
- **update**: Function that handles messages and updates the state.
- **view**: Function that renders the component based on the current state.

## Components

### Component Types

The framework defines several component types:

1. **Base Component**: The fundamental building block of all components.
2. **Page Component**: Represents a full page in the application.
3. **App Component**: The root component that manages routing and page transitions.
4. **Global Component**: Components with application-wide scope.

### Component Interface

```typescript
interface Component<Params, State, Msg> {
  init: Init<Params, State, Msg>;
  update: Update<State, Msg>;
  view: View<Props>;
}
```

Where:
- `Params`: The parameters needed to initialize the component.
- `State`: The component's state.
- `Msg`: The message type that the component can handle.
- `Props`: The properties passed to the view function.

### Component Composition

Components can be composed through:

1. **Child Components**: Components can contain other components.
2. **Message Mapping**: Messages from child components can be mapped to parent component messages.
3. **State Propagation**: Parent components can update child component states.

#### Helper Functions

- `updateChild`: Updates a child component's state within a parent's update function.
- `mapDispatch`: Creates a dispatch function for child components inside parent components.

## State Management

### Immutable State

All state in the framework is immutable, using the `immutable` helper function:

```typescript
function immutable<State>(state: State): Immutable<State>;
```

#### Why Immutability Matters

Immutability is a cornerstone of this framework for several important reasons:

1. **Predictable State Changes**: By making state immutable, we ensure that it can only change through explicit update functions. This creates a clear, traceable path for every state change in the application, making the system more predictable and easier to reason about.

2. **Simplified Debugging**: When state can't be mutated directly, it's much easier to track down the source of bugs. Since every state change produces a new state object rather than modifying the existing one, you can compare states before and after an update to identify exactly what changed.

3. **Pure Functions**: Immutability enables the use of pure functions throughout the application. Pure functions always produce the same output for the same input and have no side effects, making them easier to test, debug, and understand.

4. **Efficient Change Detection**: React and other modern UI frameworks can optimize rendering by checking if references to objects have changed. With immutable data, we simply need to check if the reference to the state object has changed to determine if a re-render is needed, rather than performing deep comparisons.

5. **Time-Travel Debugging**: Immutable state makes it possible to implement time-travel debugging, where you can step backward and forward through state changes in your application, because previous states are preserved.

6. **Concurrent Operations**: Immutable data structures can be safely shared between different parts of an application without worrying about one part unexpectedly modifying data that another part depends on.

7. **Better Developer Experience**: By enforcing immutability, developers are guided toward patterns that lead to more maintainable code. This reduces accidental complexity and makes the codebase more approachable for new team members.

In practice, the `immutable` helper function in this framework creates a deeply immutable version of your state, ensuring that neither the top-level state object nor any nested objects within it can be directly modified. Instead, to update state, you return a new state object from your update function, leaving the original untouched.

### State Updates

State updates occur through the update function:

```typescript
type Update<State, Msg> = (params: UpdateParams<State, Msg>) => UpdateReturnValue<State, Msg>;
```

The update function returns a tuple containing the new state and an array of commands to be executed.

## Routing

The router manages navigation within the application:

```typescript
interface Router<Route> {
  routes: Array<RouteDefinition<Route>>;
  routeToUrl(route: Route): string;
}
```

### Route Definition

```typescript
interface RouteDefinition<Route> {
  path: string;
  makeRoute(params: RouteDefinitionParams): Route;
}
```

### Navigation

Navigation can be triggered using:

- `pushUrlState`: Navigates to a new URL and adds it to the browser history.
- `replaceUrlState`: Navigates to a new URL and replaces the current entry in the browser history.
- `newRouteMsg`: Creates a message to navigate to a new route.
- `replaceRouteMsg`: Creates a message to replace the current route.

## Commands (Cmd)

Commands represent side effects in the framework. They allow you to perform operations such as API calls, local storage operations, and DOM manipulations.

```typescript
type Cmd<Msg> = ADT<"async", () => Promise<Msg>>;
```

### Common Commands

- `dispatch`: Dispatches a message immediately.
- `delayedDispatch`: Dispatches a message after a specified delay.
- `httpRequest`: Performs an HTTP request and handles the response.
- `localStorage`: Operations for interacting with localStorage.
- `setPageMetadata`: Updates the page metadata (e.g., title).
- `pushUrlState`: Navigates to a new URL.
- `replaceUrlState`: Replaces the current URL.

### Command Composition

Commands can be composed using helper functions:

- `map`: Transforms a command's result message.
- `andThen`: Chains commands together.
- `sequence`: Executes an array of commands in sequence.
- `join`: Combines two commands and their results.

## Building Pages

Pages are components with additional features like metadata, breadcrumbs, alerts, and modals.

```typescript
interface Component<
  RouteParams,
  SharedState,
  State,
  PageMsg,
  Route
> {
  fullWidth?: boolean;
  simpleNav?: boolean;
  backgroundColor?: ThemeColor;
  init: Init<RouteParams, SharedState, State, PageMsg, Route>;
  update: Update<State, PageMsg, Route>;
  view: View<State, PageMsg, Route>;
  sidebar?: Sidebar<State, Msg<PageMsg, Route>>;
  getMetadata: GetMetadata<State>;
  getAlerts?: GetAlerts<State, Msg<PageMsg, Route>>;
  getBreadcrumbs?: GetBreadcrumbs<State, Msg<PageMsg, Route>>;
  getModal?: GetModal<State, Msg<PageMsg, Route>>;
  getActions?: GetActions<State, Msg<PageMsg, Route>>;
}
```

### Page Initialization

Pages are initialized with:

- `routePath`: The current route path.
- `routeParams`: Parameters from the route.
- `shared`: Shared state from the application.

```typescript
interface Params<RouteParams, SharedState> {
  routePath: string;
  routeParams: Readonly<RouteParams>;
  shared: Readonly<SharedState>;
}
```

## Building Applications

Applications are created using the `start` function, which wires together the component, routing, and DOM:

```typescript
function start<AppState extends object, AppMsg, Route>(
  app: app.Component<AppState, AppMsg, Route>,
  element: HTMLElement,
  debug: boolean
): Process<AppState, app.Msg<AppMsg, Route>>;
```

### Application Process

The start function returns a `Process` object, which provides:

- `dispatch`: Function to dispatch messages to the application.
- `subscribe`: Function to subscribe to application events.
- `unsubscribe`: Function to unsubscribe from application events.
- `getState`: Function to get the current application state.

## Best Practices

### Component Organization

1. **Single Responsibility**: Components should have a single responsibility.
2. **Clear Interfaces**: Define clear interfaces for component inputs and outputs.
3. **Composability**: Design components to be composable with other components.

### State Management

1. **Immutable State**: Always treat state as immutable.
2. **Type Safety**: Define explicit types for state and messages.
3. **Minimal State**: Keep component state as minimal as necessary.

### Message Handling

1. **Typed Messages**: Use TypeScript's discriminated unions for message types.
2. **Message Mapping**: Map child component messages to parent component messages.
3. **Pure Updates**: Keep update functions pure, using commands for side effects.

### Routing

1. **Typed Routes**: Define a type for your application's routes.
2. **Route Organization**: Group related routes together.
3. **URL Consistency**: Ensure URLs match the application's navigation hierarchy.

### Performance

1. **Memoization**: Use memoization for expensive computations.
2. **Selective Updates**: Update only the parts of the state that change.
3. **Lazy Loading**: Load components and data only when needed.

---

This framework provides a robust foundation for building maintainable, type-safe web applications. By following the patterns and best practices outlined in this documentation, you can create complex UIs with predictable behavior and excellent developer experience.
