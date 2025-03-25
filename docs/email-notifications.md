# Email Notification System

This document provides a comprehensive overview of the email notification system in the Digital Marketplace project.

## Overview

The Digital Marketplace uses a robust email notification system to keep users informed about various events and actions within the platform. The system is built using NodeMailer and supports both development (Gmail) and production (SMTP) configurations.

## Configuration

### Environment Variables

The email system is configured through the following environment variables:

#### Development (Gmail)
- `MAILER_GMAIL_USER`: Gmail account username
- `MAILER_GMAIL_PASS`: Gmail account password

#### Production (SMTP)
- `MAILER_HOST`: SMTP server host
- `MAILER_PORT`: SMTP server port
- `MAILER_USERNAME`: SMTP server username
- `MAILER_PASSWORD`: SMTP server password
- `MAILER_FROM`: Sender email address in format "Name <email@domain.tld>"
- `MAILER_REPLY`: Reply-to email address
- `MAILER_BATCH_SIZE`: Maximum number of recipients per email (default: 50)
- `DISABLE_NOTIFICATIONS`: Flag to disable all email notifications
- `SHOW_TEST_INDICATOR`: Flag to add "[TEST]" prefix to email subjects

## Email Templates

The system uses React components to generate HTML email templates. Templates are located in `src/back-end/lib/mailer/templates/` and include:

- Base layout with Digital Marketplace branding
- Responsive design
- Support for test/production environments
- Unsubscribe link in footer

### Template Technical Details

#### Template Structure
- Templates are React components that return HTML
- Use the `templates.simple()` helper for consistent layout
- Support TypeScript for type safety
- Located in `src/back-end/lib/mailer/templates/`

#### Template Components
1. **Base Layout**
   - Located in `src/back-end/lib/mailer/templates.tsx`
   - Provides consistent structure and styling
   - Includes header, body, and footer sections
   - Handles responsive design

2. **Template Helpers**
   - `templates.simple()`: Creates a standardized email layout
   - `templates.Link`: Creates styled links
   - `templates.Button`: Creates styled buttons
   - `templates.DescriptionList`: Creates formatted lists

3. **Styling System**
   - Uses inline styles for email compatibility
   - Defined in `styles` object in `templates.tsx`
   - Includes utility classes for common styles
   - Supports responsive design

### Creating New Templates

1. **Create Template File**
   ```typescript
   // src/back-end/lib/mailer/notifications/your-feature.tsx
   import { Emails } from "back-end/lib/mailer";
   import * as templates from "back-end/lib/mailer/templates";
   import { makeSend } from "back-end/lib/mailer/transport";
   import React from "react";
   ```

2. **Define Template Function**
   ```typescript
   export async function yourNotificationT(
     recipient: User,
     data: YourDataType
   ): Promise<Emails> {
     const title = "Your Notification Title";
     return [
       {
         to: recipient.email || [],
         subject: title,
         html: templates.simple({
           title,
           description: "Your notification description",
           body: (
             <div>
               {/* Your notification content */}
             </div>
           )
         })
       }
     ];
   }
   ```

3. **Create Send Function**
   ```typescript
   export const yourNotification = makeSend(yourNotificationT);
   ```

4. **Template Best Practices**
   - Use TypeScript for type safety
   - Follow existing naming conventions (`*T` for template functions)
   - Include proper error handling
   - Use the template helpers for consistent styling
   - Test with different content lengths
   - Verify mobile responsiveness

5. **Testing New Templates**
   - Add to the email notification reference route
   - Test with various data scenarios
   - Verify email client compatibility
   - Check spam score

### Example Template

```typescript
import { Emails } from "back-end/lib/mailer";
import * as templates from "back-end/lib/mailer/templates";
import { makeSend } from "back-end/lib/mailer/transport";
import React from "react";
import { User } from "shared/lib/resources/user";

export async function welcomeEmailT(user: User): Promise<Emails> {
  const title = "Welcome to Digital Marketplace";
  return [
    {
      to: user.email || [],
      subject: title,
      html: templates.simple({
        title,
        description: "Thank you for joining our platform",
        body: (
          <div>
            <p>Hello {user.name},</p>
            <p>Welcome to the Digital Marketplace!</p>
            <p>We're excited to have you on board.</p>
            <templates.Button
              text="Get Started"
              url="https://marketplace.example.com/dashboard"
            />
          </div>
        )
      })
    }
  ];
}

export const welcomeEmail = makeSend(welcomeEmailT);
```

## Notification Types

### User Account Notifications
- Account registration
- Account deactivation/reactivation
- Terms and conditions updates

### Organization Notifications
- Team invitations
- Membership requests
- Organization archival
- Team member changes

### Opportunity Notifications

#### Code With Us (CWU)
- New opportunity published
- Opportunity updates
- Opportunity cancellation/suspension
- Proposal submissions
- Award notifications

#### Sprint With Us (SWU)
- New opportunity published
- Opportunity updates
- Opportunity cancellation/suspension
- Proposal submissions
- Award notifications

#### Team With Us (TWU)
- New opportunity published
- Opportunity updates
- Opportunity cancellation/suspension
- Proposal submissions
- Award notifications

## Email Reference Route

Administrators can access a comprehensive email notification reference at `/email-notification-reference`. This route displays all email templates with sample content, making it easier to review and maintain the notification system.

### Access Requirements
- User must be logged in
- User must have admin privileges
- Route is protected by `permissions.isAdmin()` check

## Implementation Details

### Transport Layer
- Uses NodeMailer for email delivery
- Supports both HTML and plain text versions
- Implements batch processing for multiple recipients
- Includes error handling and logging

### Template System
- React-based template generation
- Consistent styling across all emails
- Support for dynamic content
- Responsive design for mobile devices

### Security Features
- BCC for multiple recipients
- Configurable sender address
- Test environment indicators
- Unsubscribe functionality

## Best Practices

1. **Batch Processing**
   - Use `MAILER_BATCH_SIZE` to limit recipients per email
   - Prevents overwhelming email servers

2. **Error Handling**
   - All email sending errors are logged
   - Failed emails don't crash the application

3. **Testing**
   - Use `SHOW_TEST_INDICATOR` in development
   - Review templates using the email reference route

4. **Security**
   - Never expose email credentials in code
   - Use environment variables for sensitive data
   - Implement proper access controls

## Troubleshooting

Common issues and solutions:

1. **Emails not sending**
   - Check `DISABLE_NOTIFICATIONS` setting
   - Verify SMTP/Gmail credentials
   - Check server logs for errors

2. **Template rendering issues**
   - Review React component syntax
   - Check for missing props
   - Verify HTML structure

3. **Delivery problems**
   - Verify sender domain configuration
   - Check spam folder
   - Review email server logs
