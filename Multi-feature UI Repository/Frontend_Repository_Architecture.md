# PeopleOps Lite --- React Frontend Repository Architecture

## 1. Architecture Overview

PeopleOps Lite follows a **Feature-Driven Modular Monolith**
architecture.

The primary goals are:

-   Keep every business feature independently organized.
-   Keep reusable application-wide building blocks in `common`.
-   Keep application-wide infrastructure and state in `core`.
-   Keep `app` focused on composition and wiring.
-   Make the application easy to extend with new pages/features.
-   Keep a future migration path toward **Micro Frontends**.

------------------------------------------------------------------------

## 2. Overall `src` Structure

``` text
src/
│
├── app/
├── features/
├── common/
├── core/
├── assets/
├── styles/
└── main.tsx
```

### Responsibility

  -----------------------------------------------------------------------
  Directory                           Responsibility
  ----------------------------------- -----------------------------------
  `app/`                              Application composition, routing
                                      aggregation, layouts and app shell

  `features/`                         Business/domain modules such as
                                      Employee, Leave, Payroll

  `common/`                           Generic reusable UI components,
                                      hooks, utilities and shared types

  `core/`                             Application-wide infrastructure,
                                      authentication, settings, API and
                                      providers

  `assets/`                           Static assets such as images, icons
                                      and fonts

  `styles/`                           Global CSS, themes and styling
                                      configuration

  `main.tsx`                          React application entry point and
                                      bootstrap
  -----------------------------------------------------------------------

### Justification

This creates a clear separation between **business domains**, **generic
reusable code**, **global infrastructure**, and **application
composition**.

A useful mental model is:

> **app = composition**\
> **features = business**\
> **common = reusable**\
> **core = infrastructure**

------------------------------------------------------------------------

# 3. `features/` --- Business Feature Modules

Each major business capability is an independent module.

``` text
features/
│
├── employee/
├── leave/
├── attendance/
├── payroll/
└── performance/
```

### Justification

Each feature becomes a self-contained unit with its own UI, state,
services and types.

This makes it easier to:

-   Add new features.
-   Modify an existing feature without affecting others.
-   Apply clear module boundaries.
-   Lazy-load features.
-   Eventually extract a feature into a Micro Frontend if required.

------------------------------------------------------------------------

# 4. Feature Module Structure

Example: `features/employee/`

``` text
features/
└── employee/
    │
    ├── pages/
    ├── components/
    ├── routes/
    ├── store/
    ├── services/
    ├── types/
    ├── hooks/
    ├── tests/
    └── index.ts
```

The same structure is followed by other business features.

### Responsibility

  -----------------------------------------------------------------------
  Directory                           Responsibility
  ----------------------------------- -----------------------------------
  `pages/`                            Route-level screens/pages

  `components/`                       Components specific to the feature

  `routes/`                           Feature-owned route definitions,
                                      including child/nested routes

  `store/`                            Feature-specific state management

  `services/`                         Feature-specific API calls and
                                      service logic

  `types/`                            Feature-specific TypeScript
                                      types/interfaces

  `hooks/`                            Feature-specific custom React hooks

  `tests/`                            Unit/integration tests for the
                                      feature

  `index.ts`                          Public API/boundary of the feature
  -----------------------------------------------------------------------

### Example

``` text
employee/
│
├── pages/
│   ├── EmployeeListPage.tsx
│   ├── EmployeeProfilePage.tsx
│   └── EmployeeFormPage.tsx
│
├── components/
│   ├── EmployeeTable.tsx
│   ├── EmployeeCard.tsx
│   └── EmployeeForm.tsx
│
├── routes/
│   ├── EmployeeRoutes.tsx
│   └── employee.routes.tsx
│
├── store/
│   └── employee.store.ts
│
├── services/
│   └── employee.api.ts
│
├── types/
│   └── employee.types.ts
│
├── hooks/
│   └── useEmployees.ts
│
├── tests/
│   ├── EmployeeTable.test.tsx
│   └── EmployeeService.test.ts
│
└── index.ts
```

------------------------------------------------------------------------

# 5. Feature Routing

Routes belong to the feature that owns the business functionality.

``` text
features/employee/routes/
├── EmployeeRoutes.tsx
└── employee.routes.tsx
```

The feature defines its own page hierarchy and child routes.

The `app` layer only **composes/mounts feature routes**.

### Justification

This prevents `app/routes.tsx` from becoming a large central routing
file containing every page in the application.

It also means that when a feature grows, its routing structure grows
with the feature.

Conceptually:

``` text
App Router
    │
    ├── Employee Module
    │      ├── Employee List
    │      ├── Employee Profile
    │      └── Employee Form
    │
    ├── Leave Module
    │      ├── Leave Dashboard
    │      ├── Apply Leave
    │      └── Leave Approval
    │
    └── Payroll Module
           ├── Payroll Dashboard
           ├── Run Payroll
           └── Payslips
```

------------------------------------------------------------------------

# 6. Lazy Loading at Feature Boundary

Feature modules can be lazy-loaded from the application composition
layer.

``` text
app/
└── routes.tsx
        │
        ├── lazy → EmployeeRoutes
        ├── lazy → LeaveRoutes
        ├── lazy → AttendanceRoutes
        ├── lazy → PayrollRoutes
        └── lazy → PerformanceRoutes
```

### Justification

Lazy loading at the feature boundary provides:

-   Code splitting.
-   Smaller initial JavaScript bundle.
-   Independent feature loading.
-   A clean boundary for future Micro Frontend adoption.

The important architectural principle is:

> **Feature owns its routes; `app` decides when/how the feature is
> mounted.**

------------------------------------------------------------------------

# 7. `common/` --- Generic Reusable Building Blocks

``` text
common/
│
├── components/
├── hooks/
├── utils/
├── constants/
├── types/
└── index.ts
```

## `common/components/`

Contains generic, reusable UI primitives.

``` text
common/
└── components/
    ├── Button/
    ├── Input/
    ├── Checkbox/
    ├── Select/
    ├── Dropdown/
    ├── Modal/
    ├── Table/
    └── Navigation/
```

Examples:

-   Button
-   Input
-   Checkbox
-   Dropdown
-   Select
-   Modal
-   Table
-   Generic navigation components

### Justification

These components should be **business-agnostic**.

For example:

``` text
common/components/Button
```

should not know anything about:

``` text
Employee
Leave
Payroll
Attendance
```

A component belongs in `common` when it is generic enough to be reused
by multiple features.

------------------------------------------------------------------------

## `common/hooks/`

Contains generic reusable hooks.

Examples:

``` text
useDebounce()
usePagination()
useToggle()
useClickOutside()
```

### Rule

A hook that contains business/domain knowledge should remain inside its
feature.

``` text
common/hooks/useDebounce        ✅
features/leave/hooks/useLeaveBalance   ✅
common/hooks/useLeaveBalance   ❌
```

------------------------------------------------------------------------

## `common/utils/`

Contains generic, stateless utility functions.

Examples:

``` text
date.ts
format.ts
validation.ts
number.ts
```

These utilities should not contain business-specific logic.

------------------------------------------------------------------------

## `common/constants/`

Contains constants that are genuinely shared across the application.

Examples:

``` text
common.constants.ts
pagination.constants.ts
```

Feature-specific constants remain inside the feature.

------------------------------------------------------------------------

## `common/types/`

Contains generic/shared TypeScript types.

Examples:

``` text
Pagination.ts
ApiResponse.ts
CommonTypes.ts
```

Feature-specific types remain inside the corresponding feature.

------------------------------------------------------------------------

# 8. `core/` --- Application-Wide Infrastructure

``` text
core/
│
├── api/
├── auth/
├── settings/
├── providers/
├── config/
└── logger/
```

### Responsibility

`core` contains application-wide infrastructure and concerns that are
not owned by a particular business feature.

------------------------------------------------------------------------

## `core/api/`

Responsible for the application's HTTP/API infrastructure.

``` text
core/api/
├── apiClient.ts
├── interceptors.ts
└── api.types.ts
```

Example responsibilities:

-   Axios/fetch configuration.
-   Base URL.
-   Request/response interceptors.
-   Authentication headers.
-   Common error handling.

Feature-specific API calls remain in:

``` text
features/employee/services/
features/leave/services/
features/payroll/services/
```

------------------------------------------------------------------------

## `core/auth/`

Responsible for application-wide authentication.

``` text
core/auth/
├── auth.service.ts
├── auth.context.tsx
├── auth.provider.tsx
└── auth.guard.tsx
```

Examples:

-   Login state.
-   Token/session handling.
-   Current authenticated user.
-   Authentication guards.

------------------------------------------------------------------------

# 9. `core/settings/` --- Application-Wide Settings

Application-wide settings such as:

-   Theme
-   Language
-   Timezone

can be implemented using React Context API.

``` text
core/
└── settings/
    ├── settings.context.tsx
    ├── settings.provider.tsx
    ├── settings.types.ts
    └── useSettings.ts
```

### Justification

These settings are:

-   Global.
-   Shared across features.
-   Not business-domain specific.
-   Suitable for Context because they generally change infrequently.

Usage can remain simple:

``` tsx
const { theme, language, timezone } = useSettings();
```

------------------------------------------------------------------------

# 10. `core/providers/` --- Global Provider Composition

``` text
core/providers/
└── AppProviders.tsx
```

This is where global providers can be composed.

Conceptually:

``` text
AppProviders
│
├── SettingsProvider
├── AuthProvider
├── QueryClientProvider
└── Other Global Providers
```

### Justification

Instead of wrapping the application with many providers directly inside
`main.tsx`, provider composition is centralized.

This keeps `main.tsx` small and focused on bootstrapping.

------------------------------------------------------------------------

# 11. `core/config/`

Contains application configuration.

Examples:

``` text
core/config/
├── env.ts
└── app.config.ts
```

Responsibilities:

-   Environment configuration.
-   API base URL.
-   Application-level configuration.
-   Feature flags, where appropriate.

------------------------------------------------------------------------

# 12. `core/logger/`

Optional application-wide logging infrastructure.

``` text
core/logger/
└── logger.ts
```

Useful for:

-   Centralized logging.
-   Error reporting integration.
-   Production diagnostics.

It is optional and can be introduced when required.

------------------------------------------------------------------------

# 13. `app/` --- Application Composition Layer

``` text
app/
│
├── App.tsx
├── routes.tsx
└── layout/
    ├── MainLayout.tsx
    └── Sidebar.tsx
```

### Responsibility

`app` is the **composition root**.

It wires together:

-   Feature routes.
-   Application layouts.
-   Global application shell.

### `app/routes.tsx`

Responsible for composing feature routes:

``` text
app/routes.tsx
      │
      ├── Employee Routes
      ├── Leave Routes
      ├── Attendance Routes
      ├── Payroll Routes
      └── Performance Routes
```

### Important Rule

`app` should not contain business logic.

It answers:

> "How is the application assembled?"

rather than:

> "How does Leave or Payroll work?"

------------------------------------------------------------------------

# 14. `assets/`

Contains static application assets.

``` text
assets/
├── images/
├── icons/
└── fonts/
```

Examples:

-   Logos.
-   Images.
-   SVG icons.
-   Fonts.

No business logic should exist here.

------------------------------------------------------------------------

# 15. `styles/`

Contains global styling concerns.

``` text
styles/
├── global.css
├── variables.css
└── themes/
    ├── light.css
    └── dark.css
```

### Responsibility

-   Global CSS.
-   CSS variables.
-   Theme definitions.
-   Application-wide styling rules.

Component-specific styling should generally stay close to the component.

------------------------------------------------------------------------

# 16. `main.tsx`

The application entry point.

Its responsibility should remain minimal:

``` text
main.tsx
    │
    └── Bootstrap React
            │
            └── AppProviders
                    │
                    └── App
```

It should not contain feature logic, routing definitions or business
logic.

------------------------------------------------------------------------

# 17. Dependency / Import Boundaries

A useful dependency model is:

``` text
                    ┌─────────────┐
                    │    app      │
                    └──────┬──────┘
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
        features         core         common
             │             │             │
             └──────┬──────┘             │
                    ↓                    ↓
                  common  ←──────────────┘
```

### Practical Rules

``` text
features → common    ✅
features → core      ✅
app      → features  ✅
app      → core      ✅
app      → common    ✅

feature A → feature B ❌
common    → features  ❌
common    → core      ❌
core      → features  ❌
```

The exact dependency rules can be tightened further using ESLint
import-boundary rules.

------------------------------------------------------------------------

# 18. Final Repository Structure

Putting everything together:

``` text
src/
│
├── app/
│   ├── App.tsx
│   ├── routes.tsx
│   └── layout/
│       ├── MainLayout.tsx
│       └── Sidebar.tsx
│
├── features/
│   │
│   ├── employee/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── routes/
│   │   ├── store/
│   │   ├── services/
│   │   ├── types/
│   │   ├── hooks/
│   │   ├── tests/
│   │   └── index.ts
│   │
│   ├── leave/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── routes/
│   │   ├── store/
│   │   ├── services/
│   │   ├── types/
│   │   ├── hooks/
│   │   ├── tests/
│   │   └── index.ts
│   │
│   ├── attendance/
│   ├── payroll/
│   └── performance/
│
├── common/
│   ├── components/
│   │   ├── Button/
│   │   ├── Input/
│   │   ├── Checkbox/
│   │   ├── Dropdown/
│   │   ├── Select/
│   │   ├── Modal/
│   │   └── Table/
│   ├── hooks/
│   ├── utils/
│   ├── constants/
│   ├── types/
│   └── index.ts
│
├── core/
│   ├── api/
│   ├── auth/
│   ├── settings/
│   ├── providers/
│   ├── config/
│   └── logger/
│
├── assets/
│   ├── images/
│   ├── icons/
│   └── fonts/
│
├── styles/
│   ├── global.css
│   ├── variables.css
│   └── themes/
│
└── main.tsx
```

------------------------------------------------------------------------

# 19. Architectural Rules --- Quick Reference

  Concern                       Location
  ----------------------------- ---------------------------------
  Business functionality        `features/`

  Feature pages                 `features/<feature>/pages`
  
  Feature-specific components   `features/<feature>/components`

  Feature routes                `features/<feature>/routes`

  Feature state                 `features/<feature>/store`

  Feature API/service           `features/<feature>/services`

  Feature types                 `features/<feature>/types`

  Feature hooks                 `features/<feature>/hooks`

  Feature tests                 `features/<feature>/tests`

  Generic UI primitives         `common/components`

  Generic hooks                 `common/hooks`

  Generic utilities             `common/utils`

  Global API infrastructure     `core/api`

  Authentication                `core/auth`

  Theme/language/timezone       `core/settings`

  Global providers              `core/providers`

  App configuration             `core/config`

  Global logging                `core/logger`

  Route composition             `app/routes.tsx`

  Application layouts           `app/layout`

  Static resources              `assets`

  Global styling                `styles`

  Application bootstrap         `main.tsx`

------------------------------------------------------------------------

# 20. Core Principle

The architecture can ultimately be summarized in four rules:

> **`features/` owns business functionality.**

> **`common/` owns generic reusable building blocks.**

> **`core/` owns application-wide infrastructure and state.**

> **`app/` owns composition and wiring.**

This gives PeopleOps Lite a modular monolith structure today while
keeping each business feature sufficiently isolated for future
code-splitting and, if needed, Micro Frontend extraction.
