# Digital Marketplace Styling Guide

This document provides a comprehensive guide on how to use and customize styles in the Digital Marketplace project.

## Table of Contents

- [Overview](#overview)
- [Core Technologies](#core-technologies)
- [Style Architecture](#style-architecture)
- [Color System](#color-system)
- [Typography](#typography)
- [Layout and Spacing](#layout-and-spacing)
- [Components and Utilities](#components-and-utilities)
- [Bootstrap Integration](#bootstrap-integration)
- [Custom Classes](#custom-classes)
- [React Integration](#react-integration)
- [Best Practices](#best-practices)
- [Examples](#examples)

## Overview

The Digital Marketplace uses a custom Bootstrap-based theming system with SASS for styling. The styling approach emphasizes consistency, reusability, and maintainability across the application.

## Core Technologies

- **Bootstrap 4**: Provides the foundation for components and utilities
- **SASS**: Used for preprocessing CSS with variables, mixins, and functions
- **Grunt**: Processes SASS files and applies PostCSS transformations
- **React + Reactstrap**: Components are styled using Bootstrap classes through React

## Style Architecture

The project's SASS files are organized as follows:

```
src/front-end/sass/
├── _bootstrap.scss    # Bootstrap imports configuration
├── _font.scss         # Font declarations
├── _reboot.scss       # Bootstrap reboot overrides
└── index.scss         # Main SASS file with variable definitions and custom styles
```

The build process compiles these files into a single CSS file:

1. SASS files are compiled into CSS
2. PostCSS processes the CSS with autoprefixer for browser compatibility
3. In production, CSS is minified and compressed

## Color System

The project uses a comprehensive color system defined in `index.scss`. Colors are categorized into:

### Base Colors

```scss
// Grays
$gray-100: #f8f9fa;
$gray-200: #e9ecef;
// ... more gray variants

// Core Colors
$white: #ffffff;
$black: #000000;
$blue: #0c99d6;
$blue-alt: #17a2b8;
$blue-dark: #003366;
// ... more color definitions
```

### Theme Colors

These map to Bootstrap's theme system:

```scss
$primary: $blue;
$secondary: $gray-600;
$info: $blue-dark-alt-2;
$warning: $orange;
$danger: $red;
$success: $green;
$body: $body-color;
$light: $gray-100;
$dark: $gray-800;
```

### Application-Specific Utility Colors

The project has an extensive set of application-specific colors for different contexts:

```scss
// App Views
$c-body-bg: $blue-dark;
$c-nav-bg: $blue-dark;
$c-nav-bg-alt: $blue-dark-alt;
// ... more app-specific colors
```

All these colors are available as utility classes:
- `.bg-primary`, `.bg-secondary` (Bootstrap theme colors)
- `.bg-c-nav-bg`, `.bg-c-body-bg` (App-specific colors)

## Typography

The application uses the BC Sans font family as its primary typeface:

```scss
@font-face {
  font-family: "BCSans";
  font-style: normal;
  font-weight: 400;
  src: url(prefix-path("/fonts/BCSans/BCSans-Regular.woff2")) format("woff2"),
    url(prefix-path("/fonts/BCSans/BCSans-Regular.woff")) format("woff");
}
@font-face {
  font-family: "BCSans";
  font-weight: 500 700;
  src: url(prefix-path("/fonts/BCSans/BCSans-Bold.woff2")) format("woff2"),
    url(prefix-path("/fonts/BCSans/BCSans-Bold.woff")) format("woff");
}
```

Typography settings:

```scss
$font-family-sans-serif: "BCSans", -apple-system, BlinkMacSystemFont, "Segoe UI",
  Roboto, "Helvetica Neue", Arial, "Noto Sans", sans-serif, "Apple Color Emoji",
  "Segoe UI Emoji", "Segoe UI Symbol", "Noto Color Emoji" !default;
$font-weight-bold: 500;
$font-weight-bolder: 700;
$font-size-base: 1rem;
$h1-font-size: $font-size-base * 2.5;
```

Use text utility classes for styling text:
- `.font-weight-bold`, `.font-weight-normal`
- `.text-primary`, `.text-secondary`
- `.text-uppercase`, `.text-capitalize`

## Layout and Spacing

The project uses Bootstrap's grid system with custom breakpoints:

```scss
$grid-breakpoints: (
  xs: 0,
  sm: 576px,
  md: 1100px,
  lg: 1200px
);

$container-max-widths: (
  sm: 720px,
  md: 960px,
  lg: 1140px
);
```

### Spacing System

The project extends Bootstrap's spacing utilities with additional sizes:

```scss
$spacer: 1rem;

// 'h' stands for 'half'
$spacers: (
  n4h: ($spacer * -2),
  n6: ($spacer * -4),
  // ... more spacers
  4h: ($spacer * 2),
  6: ($spacer * 4),
  // ... more spacers
);
```

Use these spacing utilities in your components:
- Margin: `.m-1`, `.mt-2`, `.mr-3`, `.mb-4`, `.ml-5`
- Padding: `.p-1`, `.pt-2`, `.pr-3`, `.pb-4`, `.pl-5`
- Extended sizes: `.mt-6`, `.mb-7`, `.pt-8`, `.pb-9`, `.m-10`
- Negative margins: `.mt-n1`, `.mt-n4h`, etc.

## Components and Utilities

### Flex Utilities

The project makes extensive use of flexbox utilities:

```
.d-flex
.flex-row
.flex-column
.justify-content-start
.justify-content-center
.justify-content-between
.align-items-center
.align-self-start
```

### Responsive Utilities

Apply styles conditionally based on screen size:

```
.d-none                  // Hidden on all screen sizes
.d-md-block              // Visible as block on md screens and up
.flex-column             // Column layout on all screen sizes
.flex-md-row             // Row layout on md screens and up
```

## Bootstrap Integration

The project uses a customized subset of Bootstrap components defined in `_bootstrap.scss`:

```scss
@import "node_modules/bootstrap/scss/functions";
@import "node_modules/bootstrap/scss/variables";
@import "node_modules/bootstrap/scss/mixins";
@import "node_modules/bootstrap/scss/root";
@import "node_modules/bootstrap/scss/type";
// ... more component imports
```

## React Integration

In React components, use the `className` prop to apply styles:

```jsx
// Basic component with styles
<div className="mt-3 p-4 bg-light">
  Content with margin-top and padding
</div>

// Combining multiple classes
<div className={`d-flex flex-row flex-nowrap align-items-stretch ${className}`}>
  <div className="font-weight-bold align-self-start">Label:</div>
  <div className="ml-3 d-flex align-items-center">Content</div>
</div>

// Responsive layouts
<Row className="flex-grow-1 align-content-start align-content-md-stretch">
  <Col xs="12" md={4} className="sidebar bg-light pr-md-4 pr-lg-5 pt-4 pt-md-6">
    Sidebar content
  </Col>
  <Col xs="12" md={8} className="pt-md-6 pb-6">
    Main content
  </Col>
</Row>
```

## Best Practices

1. **Use Bootstrap utilities first**
   Before creating custom styles, check if Bootstrap's utility classes can solve your styling needs.

2. **Maintain consistent spacing**
   Use the spacing utilities (m-* and p-*) for margins and padding to ensure consistency.

3. **Create reusable components**
   For repeated UI patterns, create reusable components that encapsulate the styling logic.

4. **Follow responsive patterns**
   Design mobile-first and use responsive utilities to adapt to larger screens.

5. **Use theme colors**
   Stick to the defined color palette by using theme color utilities instead of hardcoding color values.

6. **Avoid inline styles**
   Use className with utility classes instead of React's inline style prop when possible.

## Examples

### Basic Button

```jsx
<button className="btn btn-primary" onClick={handleClick}>
  Submit
</button>
```

### Card Component

```jsx
<div className="card mb-4">
  <div className="card-header bg-primary text-white">
    Card Title
  </div>
  <div className="card-body">
    <p className="card-text">Card content goes here...</p>
    <button className="btn btn-outline-primary">Action</button>
  </div>
</div>
```

### Responsive Grid Layout

```jsx
<Row>
  <Col xs="12" md="6" lg="4" className="mb-4">
    <div className="p-4 bg-light h-100">Column 1</div>
  </Col>
  <Col xs="12" md="6" lg="4" className="mb-4">
    <div className="p-4 bg-light h-100">Column 2</div>
  </Col>
  <Col xs="12" md="12" lg="4" className="mb-4">
    <div className="p-4 bg-light h-100">Column 3</div>
  </Col>
</Row>
```

### Custom Component with Flexbox

```jsx
<div className="d-flex flex-column flex-md-row justify-content-between align-items-center p-4 bg-light">
  <div className="mb-3 mb-md-0">
    <h4 className="mb-0">Section Title</h4>
    <p className="text-secondary mb-0">Section description</p>
  </div>
  <div className="d-flex">
    <button className="btn btn-outline-secondary mr-2">Cancel</button>
    <button className="btn btn-primary">Save</button>
  </div>
</div>
```

### Form Controls

```jsx
<div className="form-group">
  <label htmlFor="exampleInput">Input Label</label>
  <input
    type="text"
    className="form-control"
    id="exampleInput"
    placeholder="Enter text..."
  />
  <small className="form-text text-muted">
    Helper text for the input field
  </small>
</div>
```

## Customizing the Theme

To customize the theme to match your own style guide:

1. Update color variables in `src/front-end/sass/index.scss`
2. Modify the fonts in `src/front-end/sass/_font.scss`
3. Adjust spacing and other variables as needed
4. Add or remove Bootstrap component imports in `_bootstrap.scss`
