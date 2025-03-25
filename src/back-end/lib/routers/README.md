# Routers Directory

This directory contains all of the router modules for the Digital Marketplace backend. Routers are responsible for handling HTTP requests, processing them, and returning appropriate responses. Each router module provides a set of endpoints that together implement a specific feature of the application.

## Directory Structure

```
routers/
├── admin/
│   ├── index.tsx          # Admin routing and email notification views
│   └── mocks.ts           # Mock data for admin and email testing
├── auth.ts                # Authentication and user session handling
├── front-end.ts           # Static file serving and browser support
├── status.ts              # System status endpoint
└── README.md              # This documentation
```

## Router Files

### auth.ts

**Purpose**: Manages all authentication-related operations using Keycloak as the identity provider.

**Key Features**:
- Sign-in workflow with Keycloak integration
- Session establishment and token handling
- Support for different user types (Vendor, Government)
- Callback handling for OAuth authentication flow
- Error handling and redirect mechanisms

**Integration**:
- Imported in the main application to provide authentication routes
- Uses the database connection to create and manage user sessions
- Integrates with mailer services for registration notifications

**Operations**:
- User sign-in (`/auth/sign-in`)
- Authentication callback processing (`/auth/callback`)
- Development test endpoints for session creation (in non-production environments)

**Critical Notes**:
- Depends on Keycloak configuration in the environment
- Handles identity provider hints for different user types (Vendor/Government)
- Contains logic for both new and returning users

### front-end.ts

**Purpose**: Serves static front-end files and handles browser compatibility.

**Key Features**:
- Serves files from the front-end build directory
- Browser compatibility detection
- Compression support for static files
- Fallback handling for SPA routing

**Integration**:
- Used as a catch-all router for serving the front-end application
- Integrated with the main application server to serve static assets

**Operations**:
- Serves static files with proper content types
- Detects unsupported browsers (Internet Explorer) and shows appropriate warnings
- Handles compression based on Accept-Encoding header
- Falls back to index.html for SPA routing or shows downtime page when needed

**Critical Notes**:
- Configurable fallback HTML file (index.html or downtime.html)
- Contains specific logic to bypass browser checks for asset files (CSS, JS, images, etc.)

### status.ts

**Purpose**: Provides a simple health check endpoint for monitoring the application.

**Key Features**:
- Simple HTTP endpoint that returns "OK" when the server is running

**Integration**:
- Used by monitoring systems to check if the application is responsive
- Imported into the main application as a standalone router

**Operations**:
- GET request to `/status` returns a 200 status code with "OK" message

**Critical Notes**:
- Doesn't perform any deep health checks of dependent services
- Can be expanded to include more detailed system status information

### admin/index.tsx

**Purpose**: Provides admin-specific routes including email notification previews and testing.

**Key Features**:
- Renders email templates for preview and testing
- Displays all notification types supported by the system
- Uses React server-side rendering for HTML generation

**Integration**:
- Used for administrative purposes and email template development
- Integrates with the mailer module to preview all notification types

**Operations**:
- Renders email notification references with mock data
- Groups notifications by type for easy browsing
- Provides visualization of email templates with styling

**Critical Notes**:
- Uses mock data defined in mocks.ts
- For development and testing purposes
- Includes comprehensive samples of all notification types

### admin/mocks.ts

**Purpose**: Provides mock data for testing email notifications and admin functionality.

**Key Features**:
- Contains mock data for users, organizations, opportunities, and proposals
- Provides realistic test data for all entity types
- Used by the admin router to render notification previews

**Integration**:
- Imported by the admin router to provide test data
- Reflects the data structure of the application's models

**Operations**:
- No direct HTTP operations (supporting file)
- Provides mock data for various application scenarios

**Critical Notes**:
- Should be kept in sync with the application's data models
- Includes examples of all major entity types in the system

## Integration with the Application

The routers are integrated into the application through the main server setup, typically in `index.ts` or a similar file. Each router exports a function that creates router instances, which are then added to the Express or custom HTTP server.

The general pattern is:

```typescript
// In the main application file
import authRouter from './lib/routers/auth';
import frontEndRouter from './lib/routers/front-end';
import statusRouter from './lib/routers/status';
import adminRouter from './lib/routers/admin';

// Initialize routers
const routers = [
  authRouter(dbConnection),
  statusRouter,
  adminRouter(),
  // The front-end router should be last as it has a catch-all route
  frontEndRouter('index.html')
];

// Add routers to the server
server.addRouters(routers);
```

## Common Router Patterns

1. **Router Factory Function**: Most routers export a factory function that takes dependencies like database connections and returns a router.

2. **Request Handlers**: Each route defines handlers that process requests, typically using the `nullRequestBodyHandler` helper for GET requests or other handler types for POST/PUT.

3. **Response Construction**: Handlers return objects with status codes, headers, session objects, and response bodies.

4. **Type Safety**: Routers use TypeScript generics to ensure type safety throughout the request/response cycle.

## Security Considerations

1. The authentication router handles sensitive user data and tokens.
2. HTTPS should be used in production to protect authentication data.
3. The front-end router includes browser compatibility checks to prevent security issues with outdated browsers.
4. Admin routers should be properly secured against unauthorized access.

## Extending the Routers

To add a new router:

1. Create a new file in the routers directory
2. Define your routes following the pattern in the existing routers
3. Export a factory function that returns a router array
4. Import and add the router to the main application

## Best Practices

1. Group related routes in the same router file or subdirectory
2. Use TypeScript for strong typing of request and response bodies
3. Implement proper error handling for all routes
4. Document each router's purpose and functionality
5. Use descriptive route paths that follow RESTful conventions when appropriate
