# Debugging with the Process Type

This document explains how to use the `Process` type to introspect running applications in debug mode within the Digital Marketplace project.

## Overview

The `Process` type represents a running App component and provides several methods for interacting with and inspecting the application state. This feature is particularly useful during development for debugging purposes.

## How Debug Mode Works

When an application is started with debug mode enabled, two key things happen:

1. The application's `Process` object is exposed globally through the browser's `window` object
2. All process events (state changes and message dispatches) are logged to the console

## Accessing the Process Object

In development environment, the Process object is automatically exposed as `window.process`. You can access it from your browser's developer console:

```javascript
// From the browser console
console.log(window.process);
```

## Process Interface

The Process object provides the following methods:

```typescript
interface Process<State, Msg> {
  dispatch: Dispatch<Msg>;        // Dispatch messages to the application
  subscribe: Subscribe<State, Msg>; // Subscribe to application events
  unsubscribe: Unsubscribe<State, Msg>; // Unsubscribe from application events
  getState(): Immutable<State>;   // Get the current application state
}
```

## Debugging Techniques

### 1. Inspecting Current State

You can examine the current application state at any time:

```javascript
// Get the full application state
const state = window.process.getState();
console.log(state);

// Inspect specific parts of the state
console.log(state.get('nav'));
console.log(state.getIn(['pages', 'dashboard']));
```

### 2. Dispatching Messages

You can dispatch messages to the application to test behavior:

```javascript
// Import adt from the console
// (assumes you've imported it in your application)
const { adt } = window;

// Dispatch a navigation message
window.process.dispatch(adt("nav", adt("toggleMobileMenu", true)));

// Dispatch a page-specific message
window.process.dispatch(adt("pageDashboard", adt("someAction", { id: "123" })));
```

### 3. Subscribing to Events

You can listen for state changes and message dispatches:

```javascript
// Subscribe to all events
const subscription = event => {
  if (event.tag === "stateChanged") {
    console.log("State changed:", event.value.state);
  } else if (event.tag === "msgDispatched") {
    console.log("Message dispatched:", event.value);
  }
};

window.process.subscribe(subscription);

// Later, to stop listening
window.process.unsubscribe(subscription);
```

### 4. Monitoring Specific State Changes

You can create targeted subscriptions to watch specific parts of the state:

```javascript
const monitorNavState = event => {
  if (event.tag === "stateChanged") {
    console.log("Nav state:", event.value.state.get('nav'));
  }
};

window.process.subscribe(monitorNavState);
```

### 5. Watching Console Logs

When debug mode is enabled, all process events are automatically logged to the console. Look for:
- "process event published" messages
- Events with the tag "msgDispatched" (shows messages being processed)
- Events with the tag "stateChanged" (shows state updates)

## Practical Examples

### Example 1: Debug a Form Submission

```javascript
// Watch for form submission messages
const formSubmissionWatcher = event => {
  if (event.tag === "msgDispatched" &&
      event.value.tag === "someForm" &&
      event.value.value.tag === "submit") {
    console.log("Form submitted with data:", event.value.value.value);
  }
};

window.process.subscribe(formSubmissionWatcher);
```

### Example 2: Monitor API Responses

```javascript
// Watch for API response state changes
const apiResponseWatcher = event => {
  if (event.tag === "stateChanged") {
    const apiData = event.value.state.getIn(['pages', 'somePage', 'apiData']);
    if (apiData) {
      console.log("API data updated:", apiData);
    }
  }
};

window.process.subscribe(apiResponseWatcher);
```

## Tips and Best Practices

1. **Create Focused Subscriptions**: Create specific subscriptions that filter for exactly what you need rather than logging everything.

2. **Temporary Debugging**: Add console.log statements in the `update` function of components you're working on:
   ```typescript
   export const update: Update<State, Msg> = ({ state, msg }) => {
     console.log('Component received message:', msg);
     // ...
   }
   ```

3. **Clean Up Subscriptions**: Always unsubscribe when you're done debugging to avoid memory leaks.

4. **Use Browser DevTools**: The React and Redux DevTools browser extensions can complement this approach.

5. **Introspect Child Components**: You can access and monitor nested child component states:
   ```javascript
   // Get state of a child component
   const childState = window.process.getState().getIn(['pages', 'childComponentPath']);
   console.log(childState);
   ```

## Enabling Debug Mode

Debug mode is enabled automatically in development environments. In the application's entry point:

```typescript
// src/front-end/typescript/index.ts
const debug = NODE_ENV === "development";
const process_ = process.start(app, element, debug);

// Expose application process in development
if (debug) {
  set(window, "process", process_);
}
```

If you need to enable debug mode manually in other environments, you can pass `true` as the third parameter to `process.start`.
