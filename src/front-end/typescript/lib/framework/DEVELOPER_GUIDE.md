# Developer Guide: Building with the Custom Front-End Framework

This guide provides practical examples and step-by-step instructions for developers working with the custom front-end framework. It complements the more conceptual overview provided in the README.md.

## Table of Contents

- [Creating a Basic Component](#creating-a-basic-component)
- [Building a Page Component](#building-a-page-component)
- [Handling State Updates](#handling-state-updates)
- [Working with Commands](#working-with-commands)
- [Component Composition](#component-composition)
- [Routing](#routing)
- [Working with Forms](#working-with-forms)
- [Making API Calls](#making-api-calls)
- [Common Patterns](#common-patterns)
- [Debugging Tips](#debugging-tips)

## Creating a Basic Component

### Step 1: Define Types

Start by defining the component's types:

```typescript
// Types for your component
export interface State {
  counter: number;
  message: string;
}

// Define messages the component can handle
export type Msg =
  | ADT<"increment">
  | ADT<"decrement">
  | ADT<"setMessage", string>;
```

### Step 2: Create Initial State

Define the initial state function:

```typescript
// Init function for your component
export const init: component.base.Init<null, State, Msg> = (params) => {
  return [
    {
      counter: 0,
      message: "Hello, World!"
    },
    [] // No initial commands
  ];
};
```

### Step 3: Create Update Function

Define how your component handles messages:

```typescript
// Update function for your component
export const update: component.base.Update<State, Msg> = ({ state, msg }) => {
  switch (msg.tag) {
    case "increment":
      return [
        immutable({
          ...state,
          counter: state.counter + 1
        }),
        []
      ];
    case "decrement":
      return [
        immutable({
          ...state,
          counter: state.counter - 1
        }),
        []
      ];
    case "setMessage":
      return [
        immutable({
          ...state,
          message: msg.value
        }),
        []
      ];
  }
};
```

### Step 4: Create View Function

Define how to render your component:

```typescript
// View function for your component
export const view: component.base.ComponentView<State, Msg> = (props) => {
  const { state, dispatch } = props;
  return (
    <div className="mt-3">
      <p>{state.message}</p>
      <p>Counter: {state.counter}</p>
      <div className="btn-group">
        <button
          className="btn btn-primary"
          onClick={() => dispatch(adt("increment"))}>
          +
        </button>
        <button
          className="btn btn-secondary"
          onClick={() => dispatch(adt("decrement"))}>
          -
        </button>
      </div>
      <input
        className="form-control mt-2"
        value={state.message}
        onChange={(e) => dispatch(adt("setMessage", e.target.value))}
      />
    </div>
  );
};
```

### Step 5: Export the Component

Combine everything into a component:

```typescript
export const component: component.base.Component<null, State, Msg> = {
  init,
  update,
  view
};
```

## Building a Page Component

Pages handle routing, metadata, and have additional features like sidebars and modals.

### Step 1: Define Page Types

```typescript
export interface RouteParams {
  id?: string;
}

export interface State {
  title: string;
  content: string;
  isLoading: boolean;
}

export type Msg =
  | ADT<"setTitle", string>
  | ADT<"setContent", string>
  | ADT<"loadContent", ContentResponse>;
```

### Step 2: Create Page Init Function

```typescript
export const init: page.Init<RouteParams, SharedState, State, Msg, Route> =
  ({ routeParams, shared }) => {
  const id = routeParams.id;
  const state: State = {
    title: "Loading...",
    content: "",
    isLoading: true
  };

  return [
    state,
    [
      api.getContent<Msg>(id, (response) => {
        return adt("loadContent", response);
      })
    ]
  ];
};
```

### Step 3: Create Page Update Function

```typescript
export const update: page.Update<State, Msg, Route> = ({ state, msg }) => {
  switch (msg.tag) {
    case "setTitle":
      return [
        immutable({
          ...state,
          title: msg.value
        }),
        []
      ];
    case "setContent":
      return [
        immutable({
          ...state,
          content: msg.value
        }),
        []
      ];
    case "loadContent":
      if (msg.value.tag === "valid") {
        return [
          immutable({
            ...state,
            title: msg.value.value.title,
            content: msg.value.value.content,
            isLoading: false
          }),
          [
            cmd.setPageMetadata({
              title: msg.value.value.title
            }, adt("noop"))
          ]
        ];
      } else {
        return [
          immutable({
            ...state,
            title: "Error",
            content: "Failed to load content",
            isLoading: false
          }),
          []
        ];
      }
  }
};
```

### Step 4: Create Page View

```typescript
export const view: page.View<State, Msg, Route> = ({ state, dispatch }) => {
  if (state.isLoading) {
    return <LoadingSpinner />;
  }

  return (
    <div className="container">
      <h1>{state.title}</h1>
      <div className="content">{state.content}</div>
      <button
        className="btn btn-primary"
        onClick={() => dispatch(adt("setTitle", "New Title"))}>
        Change Title
      </button>
    </div>
  );
};
```

### Step 5: Add Page Metadata and Export

```typescript
export const getMetadata: page.GetMetadata<State> = (state) => ({
  title: state.title || "Content Page"
});

export const component: page.Component<
  RouteParams,
  SharedState,
  State,
  Msg,
  Route
> = {
  init,
  update,
  view,
  getMetadata,
  fullWidth: false
};
```

## Handling State Updates

### Example: Complex State Update

```typescript
case "updateUserProfile":
  const newProfile = {...state.profile};

  if (msg.value.name) {
    newProfile.name = msg.value.name;
  }

  if (msg.value.email) {
    newProfile.email = msg.value.email;
  }

  // Handle nested state updates
  if (msg.value.preferences) {
    newProfile.preferences = {
      ...newProfile.preferences,
      ...msg.value.preferences
    };
  }

  return [
    immutable({
      ...state,
      profile: newProfile,
      lastUpdated: new Date().toISOString()
    }),
    []
  ];
```

### Example: Using State Update Helpers

```typescript
// Update an item in an array
case "updateTodoItem":
  const todoIndex = state.todos.findIndex(todo => todo.id === msg.value.id);
  if (todoIndex === -1) return [state, []];

  const newTodos = [...state.todos];
  newTodos[todoIndex] = {
    ...newTodos[todoIndex],
    ...msg.value.changes
  };

  return [
    immutable({
      ...state,
      todos: newTodos
    }),
    []
  ];
```

## Working with Commands

Commands handle side effects in your application.

### Example: HTTP Request

```typescript
// API module helper
export function fetchUserData<Msg>(
  userId: string,
  onResponse: (response: ResponseValidation<UserData, string>) => Msg
): Cmd<Msg> {
  return cmd.httpRequest({
    method: "GET",
    url: `/api/users/${userId}`,
    transformResponse: (raw) => raw as UserData,
    handleResponse: onResponse
  });
}

// In component update function
case "loadUserData":
  return [
    immutable({
      ...state,
      isLoading: true
    }),
    [
      api.fetchUserData(msg.value, (response) => {
        return adt("userDataLoaded", response);
      })
    ]
  ];

case "userDataLoaded":
  if (msg.value.tag === "valid") {
    // Success case
    return [
      immutable({
        ...state,
        userData: msg.value.value,
        isLoading: false,
        error: null
      }),
      []
    ];
  } else {
    // Error case
    return [
      immutable({
        ...state,
        isLoading: false,
        error: "Failed to load user data"
      }),
      []
    ];
  }
```

### Example: Delayed Actions

```typescript
case "startTimer":
  return [
    immutable({
      ...state,
      timerRunning: true
    }),
    [cmd.delayedDispatch(1000, adt("tick"))]
  ];

case "tick":
  const newCount = state.count + 1;
  const cmds = state.timerRunning
    ? [cmd.delayedDispatch(1000, adt("tick"))]
    : [];

  return [
    immutable({
      ...state,
      count: newCount
    }),
    cmds
  ];
```

## Component Composition

The framework implements a prescriptive approach to component composition via helper functions that standardize how components interact with each other in a type-safe and predictable way.

### Key Helper Functions

The framework provides two main helper functions for component composition:

1. **updateChild**: A function that handles updating a child component's state within a parent component. This function:
   - Takes the parent state
   - Finds the child state using a path
   - Calls the child's update function with the appropriate message
   - Updates the parent state with the new child state
   - Maps the child's commands to the parent's command type
   - Allows for additional processing after the child update

2. **mapDispatch**: A function that creates a wrapper around the parent's dispatch function to allow child components to dispatch messages that will be transformed into parent messages. This function:
   - Takes the parent's dispatch function
   - Takes a mapping function that converts child messages to parent messages
   - Returns a new dispatch function that can be passed to the child
   - Ensures type-safety throughout the message flow
   - Allows child components to be decoupled from their parent context

### Example: Parent-Child Component Relationship

This example demonstrates how a parent component integrates a child component using the framework's composition helpers.

Child component:

```typescript
// In child.ts
export interface State {
  value: string;
}

export type Msg = ADT<"setValue", string>;

export const init: component.base.Init<null, State, Msg> = () => [
  { value: "" },
  []
];

export const update: component.base.Update<State, Msg> = ({ state, msg }) => {
  switch (msg.tag) {
    case "setValue":
      return [
        immutable({
          ...state,
          value: msg.value
        }),
        []
      ];
  }
};

export const view: component.base.ComponentView<State, Msg> = ({ state, dispatch }) => (
  <input
    value={state.value}
    onChange={e => dispatch(adt("setValue", e.target.value))}
  />
);

export const component: component.base.Component<null, State, Msg> = {
  init,
  update,
  view
};
```

Parent component:

```typescript
// In parent.ts
import * as Child from './child';

export interface State {
  title: string;
  childState: Child.State; // Store child's state inside parent state
}

// Define parent's message types, including a wrapper for child messages
export type Msg =
  | ADT<"setTitle", string>
  | ADT<"childMsg", Child.Msg>; // Wrapper for child messages

export const init: component.base.Init<null, State, Msg> = () => {
  // Initialize the child component
  const [childState, childCmds] = Child.init(null);

  return [
    {
      title: "Parent Component",
      childState // Store child's initial state
    },
    // Map child's commands to parent's message type
    childCmds.map(cmd => cmd.map(childMsg => adt("childMsg", childMsg)))
  ];
};

export const update: component.base.Update<State, Msg> = ({ state, msg }) => {
  switch (msg.tag) {
    case "setTitle":
      return [
        immutable({
          ...state,
          title: msg.value
        }),
        []
      ];

    case "childMsg":
      // Handle messages from the child component
      // using the updateChild helper function
      return component.base.updateChild({
        state: immutable(state),
        childStatePath: ["childState"], // Path to child state in parent state
        childUpdate: Child.update,      // Child's update function
        childMsg: msg.value,            // The actual child message
        mapChildMsg: (childMsg) => adt("childMsg", childMsg) // How to map child commands
      });
  }
};

export const view: component.base.ComponentView<State, Msg> = ({ state, dispatch }) => (
  <div>
    <h2>{state.title}</h2>
    <Child.view
      state={state.childState}
      dispatch={component.base.mapDispatch(
        dispatch,
        (childMsg) => adt("childMsg", childMsg)
      )}
    />
  </div>
);
```

### Understanding Child Message Flow in the Example

Let's break down how messages flow in this parent-child relationship:

1. **Message Wrapping**:
   The parent defines a message type `ADT<"childMsg", Child.Msg>` that wraps any message from the child component. This creates a clear boundary between parent and child messages.

2. **Dispatch Transformation**:
   When rendering the child component, the parent uses `mapDispatch` to transform the child's dispatch function:
   ```typescript
   dispatch={component.base.mapDispatch(
     dispatch,  // Parent's dispatch function
     (childMsg) => adt("childMsg", childMsg)  // Wrapping function
   )}
   ```
   This means when the child calls `dispatch(adt("setValue", "new value"))`, it gets automatically transformed to the parent's dispatch call: `dispatch(adt("childMsg", adt("setValue", "new value")))`.

3. **Message Handling**:
   When the parent receives a message with the `"childMsg"` tag, it uses `updateChild` to:
   - Extract the actual child message (`msg.value`)
   - Find the child's state in the parent state
   - Call the child's update function with the child message
   - Update the parent state with the new child state
   - Map any commands returned by the child

4. **Command Mapping**:
   Both during initialization and in `updateChild`, any commands from the child are mapped to the parent's message type using the same `(childMsg) => adt("childMsg", childMsg)` mapping function.

This approach creates a clean separation between parent and child while ensuring type safety throughout the message flow. The child component doesn't need to know anything about its parent, and the parent has a structured way to manage child component state and messages.

### State Ownership and Management

In this framework, an important pattern to understand is how component state is owned and managed in a component hierarchy:

1. **State Storage vs. State Logic**: While the parent component physically stores the child component's state in its state tree, the child component still encapsulates the logic for managing that state.

2. **Hierarchical State Tree**: The framework uses a hierarchical state tree where:
   ```typescript
   // Parent state contains child state
   export interface ParentState {
     title: string;
     childState: ChildState; // Nested state
   }
   ```

3. **Encapsulated Update Logic**: Each component defines its own update logic. The parent doesn't need to understand how the child state works internally:
   ```typescript
   // Child defines its own update logic
   export const update: component.base.Update<State, Msg> = ({ state, msg }) => {
     switch (msg.tag) {
       case "setValue":
         return [
           immutable({
             ...state,
             value: msg.value
           }),
           []
         ];
     }
   };
   ```

4. **Delegated State Management**: When a child message occurs, the parent delegates the state update to the child's update function using `updateChild`:
   ```typescript
   case "childMsg":
     return component.base.updateChild({
       state: immutable(state),
       childStatePath: ["childState"],
       childUpdate: Child.update,  // Child's update logic is used
       childMsg: msg.value,
       mapChildMsg: (childMsg) => adt("childMsg", childMsg)
     });
   ```

This pattern combines the benefits of:
- **Single State Tree**: Similar to Redux/Elm, all state flows from a single source of truth
- **Component Encapsulation**: Each component manages its own logic
- **Type Safety**: Proper message mapping ensures type safety across components
- **Predictable Data Flow**: Data flows down, events flow up
- **Testability**: Components can be tested in isolation

This is a core part of the "prescriptive approach to component composition" that the framework implements. It provides a structured way to build complex UIs from simple, composable components.

### Component Factory Functions

The framework also provides factory functions for creating composite components:

```typescript
// Example of a component factory function
export function makeComponent<Value, ChildParams, ChildState, InnerChildMsg>(
  params: ChildComponent<Value, ChildParams, ChildState, InnerChildMsg>
): Component<Value, ChildParams, ChildState, InnerChildMsg> {
  return {
    init: makeInit(params.init),
    update: makeUpdate(params.update),
    view: makeView(params.view)
  };
}
```

These factory functions enforce a consistent structure for common component patterns and reduce boilerplate code when creating components that follow these patterns.

### Benefits of the Prescriptive Approach

1. **Type Safety**: The generic type parameters ensure that components are composed correctly.
2. **Reusability**: Components can be easily reused in different contexts.
3. **Separation of Concerns**: Each component only needs to handle its own specific logic.
4. **Consistent Patterns**: All components follow the same patterns for state management and message passing.
5. **Reduced Boilerplate**: Helper functions reduce the amount of code needed for component composition.

## Routing

### Defining Routes

```typescript
// In routes.ts
export type Route =
  | ADT<"home">
  | ADT<"userProfile", { userId: string }>
  | ADT<"settings">
  | ADT<"notFound">;

export const router: Router<Route> = {
  routes: [
    {
      path: "/",
      makeRoute: () => adt("home")
    },
    {
      path: "/users/:userId",
      makeRoute: ({ params }) => adt("userProfile", { userId: params.userId || "" })
    },
    {
      path: "/settings",
      makeRoute: () => adt("settings")
    }
  ],

  routeToUrl(route) {
    switch (route.tag) {
      case "home": return "/";
      case "userProfile": return `/users/${route.value.userId}`;
      case "settings": return "/settings";
      case "notFound": return "/not-found";
    }
  }
};
```

### Using Routes in Components

```typescript
// Navigation in a component
const onProfileClick = () => {
  dispatch(adt("navigate", router.newRouteMsg(adt("userProfile", { userId: "123" }))));
};

// In update function
case "navigate":
  return [
    state,
    [cmd.pushUrlState(router.routeToUrl(msg.value.route), adt("noop"))]
  ];
```

## Working with Forms

### Form State

```typescript
export interface FormState {
  username: string;
  email: string;
  password: string;
  validationErrors: {
    username?: string;
    email?: string;
    password?: string;
  };
  isSubmitting: boolean;
}

export type FormMsg =
  | ADT<"setUsername", string>
  | ADT<"setEmail", string>
  | ADT<"setPassword", string>
  | ADT<"validate">
  | ADT<"submit">
  | ADT<"submitSuccess">
  | ADT<"submitError", string>;
```

### Form Update Function

```typescript
function validateForm(state: FormState): FormState {
  const errors: FormState["validationErrors"] = {};

  if (!state.username) {
    errors.username = "Username is required";
  }

  if (!state.email) {
    errors.email = "Email is required";
  } else if (!/\S+@\S+\.\S+/.test(state.email)) {
    errors.email = "Invalid email format";
  }

  if (!state.password) {
    errors.password = "Password is required";
  } else if (state.password.length < 8) {
    errors.password = "Password must be at least 8 characters";
  }

  return {
    ...state,
    validationErrors: errors
  };
}

// Update function
case "setUsername":
  return [
    immutable({
      ...state,
      username: msg.value
    }),
    []
  ];

case "validate":
  const validatedState = validateForm(state);
  const hasErrors = Object.keys(validatedState.validationErrors).length > 0;

  return [
    immutable(validatedState),
    hasErrors ? [] : [cmd.dispatch(adt("submit"))]
  ];

case "submit":
  return [
    immutable({
      ...state,
      isSubmitting: true
    }),
    [
      api.createUser({
        username: state.username,
        email: state.email,
        password: state.password
      }, (response) => {
        if (response.tag === "valid") {
          return adt("submitSuccess");
        } else {
          return adt("submitError", "Failed to create user");
        }
      })
    ]
  ];
```

## Making API Calls

### API Module

```typescript
// In api.ts
export function getUser<Msg>(
  userId: string,
  onResponse: (response: ResponseValidation<UserData, string>) => Msg
): cmd.Cmd<Msg> {
  return cmd.httpRequest({
    method: "GET",
    url: `/api/users/${userId}`,
    transformResponse: (raw) => raw as UserData,
    handleResponse: onResponse
  });
}

export function createUser<Msg>(
  userData: { username: string; email: string; password: string },
  onResponse: (response: ResponseValidation<UserData, ValidationErrors>) => Msg
): cmd.Cmd<Msg> {
  return cmd.httpRequest({
    method: "POST",
    url: "/api/users",
    body: userData,
    transformResponse: (raw) => raw as UserData,
    handleResponse: onResponse
  });
}
```

### Using API in Components

```typescript
// In component update function
case "loadUser":
  return [
    immutable({
      ...state,
      isLoading: true
    }),
    [
      api.getUser(msg.value, (response) => {
        if (response.tag === "valid") {
          return adt("userLoaded", response.value);
        } else {
          return adt("userLoadError", "Failed to load user");
        }
      })
    ]
  ];

case "userLoaded":
  return [
    immutable({
      ...state,
      user: msg.value,
      isLoading: false
    }),
    []
  ];
```

## Common Patterns

### Loading States

```typescript
export interface State {
  data: Data | null;
  isLoading: boolean;
  error: string | null;
}

export type Status = "idle" | "loading" | "success" | "error";

// Helper function to determine status
function getStatus(state: State): Status {
  if (state.isLoading) return "loading";
  if (state.error) return "error";
  if (state.data) return "success";
  return "idle";
}

// In view function
export const view: component.base.ComponentView<State, Msg> = ({ state, dispatch }) => {
  const status = getStatus(state);

  return (
    <div>
      {status === "loading" && <LoadingSpinner />}
      {status === "error" && <ErrorMessage message={state.error} />}
      {status === "success" && <DataView data={state.data} />}
      {status === "idle" && <EmptyState />}
    </div>
  );
};
```

### Modal Dialogs

```typescript
// In state
export interface State {
  // ...other state properties
  modal: {
    isOpen: boolean;
    type: "confirmation" | "form" | "alert" | null;
    title: string;
    message: string;
    data?: any;
  };
}

// In messages
export type Msg =
  // ...other messages
  | ADT<"openModal", { type: "confirmation" | "form" | "alert"; title: string; message: string; data?: any }>
  | ADT<"closeModal">
  | ADT<"confirmModal">
  | ADT<"cancelModal">;

// In update function
case "openModal":
  return [
    immutable({
      ...state,
      modal: {
        isOpen: true,
        type: msg.value.type,
        title: msg.value.title,
        message: msg.value.message,
        data: msg.value.data
      }
    }),
    []
  ];

case "closeModal":
  return [
    immutable({
      ...state,
      modal: {
        ...state.modal,
        isOpen: false
      }
    }),
    []
  ];
```

## Debugging Tips

### Console Logging

When debugging, you can add console logs to track messages and state changes:

```typescript
export const update: component.base.Update<State, Msg> = ({ state, msg }) => {
  console.log('Received message:', msg);

  const result = (() => {
    switch (msg.tag) {
      // ... message handling
    }
  })();

  console.log('New state:', result[0]);
  return result;
};
```

### Using the Debug Flag

When initializing your application, set the debug flag to true to see detailed logs:

```typescript
const process = framework.process.start(App.component, element, true);
```

### Troubleshooting Child Components

When child components aren't updating as expected:

1. Check that `childStatePath` is correct in `updateChild`
2. Verify that child messages are correctly mapped to parent messages
3. Ensure the child component's view receives the correct state and dispatch function

```typescript
// Debugging child component issues
console.log('Child state path:', childStatePath);
console.log('Child state:', state.getIn(childStatePath));
console.log('Child update result:', Child.update({
  state: state.getIn(childStatePath),
  msg: childMsg
}));
```

---

This developer guide provides practical examples for common patterns and scenarios when working with the custom front-end framework. For more conceptual information, refer to the README.md file.
