# React (TypeScript) Coding Standards

This document defines the coding standards for all React/TypeScript projects using **React 19+, TanStack Router, TanStack Query, and Vite**.

Every developer and AI coding assistant (Claude Code, Opencode, Antigravity, ChatGPT, Copilot, Cursor, etc.) must follow these standards consistently.

Unlike backend frameworks, React does not enforce architecture. Therefore every project, regardless of size, must follow a **domain-based architecture** from the beginning.

Routes must use **TanStack Router with one file per route**.

---

# Technology Stack

Default stack:

- React
- TypeScript (strict mode)
- Vite
- TanStack Router
- TanStack Query
- Axios
- React Hook Form
- Zod
- TailwindCSS
- Shadcn UI (preferred UI library)

---

# Core Principles

- Follow SOLID principles where applicable.
- Enable TypeScript strict mode.
- Domain-first architecture.
- Never mix business logic into components.
- Prefer composition over inheritance.
- Small reusable components.
- Keep components focused.
- Avoid duplicated logic.
- Prefer readability over clever code.

---

# Architecture

Always organize by domain.

```
src/
│
├── app/
│   ├── router.tsx
│   ├── providers/
│   ├── layouts/
│   └── hooks/
│
├── routes/
│   ├── __root.tsx
│   ├── login.tsx
│   ├── dashboard.tsx
│   ├── users/
│   │    ├── index.tsx
│   │    ├── create.tsx
│   │    ├── $userId.tsx
│   │    └── $userId.edit.tsx
│   │
│   └── products/
│
├── domains/
│   ├── auth/
│   ├── users/
│   ├── products/
│   └── dashboard/
│
├── shared/
│   ├── api/
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   ├── types/
│   ├── constants/
│   └── lib/
│
├── assets/
├── styles/
└── main.tsx
```

Routes are responsible only for rendering pages.

Business logic belongs inside domains.

---

# Domain Structure

Each domain owns everything related to that feature.

Example

```
domains/

users/

    api/

    components/

    hooks/

    services/

    schemas/

    types/

    utils/

    constants/

    index.ts
```

A domain exposes a single public API through `index.ts`.

No domain should import another domain's internal files.

---

# Routes

Use file-based routing.

Each route gets its own file.

Good

```
routes/users/index.tsx
routes/users/create.tsx
routes/users/$userId.tsx
routes/users/$userId.edit.tsx
```

Avoid putting multiple routes into one file.

---

# Components

Keep components small.

Components should only:

- render UI
- receive props
- emit events

Avoid:

- API calls
- business logic
- data transformation
- permissions
- calculations

Bad

```tsx
function UserCard() {
    const users = await axios.get(...)

    ...
}
```

Good

```tsx
function UserCard({
    user,
}: Props) {
    ...
}
```

---

# Business Logic

Business logic belongs in:

- hooks
- services
- utilities
- domain helpers

Never inside components.

---

# Hooks

Custom hooks are the application layer.

Good examples

```
useLogin()

useCurrentUser()

useCreateOrder()

useDashboard()

useProducts()

usePermissions()
```

Hooks should encapsulate:

- queries
- mutations
- state
- reusable logic

---

# TanStack Query

Every API request should go through TanStack Query.

Never manually manage loading state for server data.

Bad

```tsx
const [loading, setLoading] = useState(false);
```

Good

```tsx
const users = useQuery(...)
```

Always:

- cache data
- invalidate queries
- use query keys
- use optimistic updates where appropriate

---

# API Layer

Never call Axios directly inside components.

Bad

```tsx
await axios.get(...)
```

Good

```
domains/users/api/user.api.ts
```

Example

```ts
export function getUsers() {
    return api.get('/users');
}
```

---

# Services

Services orchestrate business operations.

Example

```
UserService

OrderService

ForecastService
```

A service may call multiple API endpoints.

---

# Forms

Always use

- React Hook Form
- Zod validation

Never use uncontrolled manual validation.

---

# Validation

Validation belongs in Zod schemas.

```
schemas/

create-user.schema.ts

update-user.schema.ts
```

Never validate inline inside components.

---

# State Management

Prefer:

1. URL state
2. TanStack Query
3. Local component state

Avoid global state unless necessary.

Global state is for:

- authentication
- theme
- feature flags

Not for server data.

---

# React State

Prefer

```
useState
```

for local UI state.

Avoid putting everything into Context.

Context is not a state management solution.

---

# Props

Use typed props.

Bad

```tsx
function UserCard(props:any)
```

Good

```tsx
type Props = {
    user: User;
}

function UserCard({
    user,
}: Props) {

}
```

Never use any.

---

# Component Size

Review components over

300 lines

Split components over

500 lines

---

# Page Components

A page should mainly compose components.

Example

```tsx
export default function UsersPage() {

    return (

        <>
            <Header />

            <UsersTable />

            <Pagination />

        </>

    )

}
```

Avoid large pages.

---

# Tables

Extract tables into dedicated components.

Never build 500-line tables inside a page.

---

# Modals

Every reusable modal gets its own component.

Never inline large modal JSX.

---

# Layouts

Shared layouts belong in

```
app/layouts
```

Examples

```
DashboardLayout

AuthLayout

SettingsLayout
```

---

# Loading States

Always provide loading UI.

Prefer Skeletons.

Avoid blank pages.

---

# Error States

Always display meaningful errors.

Never silently fail.

---

# Permissions

Permission logic belongs in hooks.

Example

```
useCanEditUser()

useCanDeleteProduct()
```

Avoid

```tsx
if(role=="admin")
```

inside components.

---

# Constants

Never hardcode:

roles

statuses

permissions

routes

API URLs

Create constants.

---

# Routes Constants

Prefer

```ts
Routes.Users
```

instead of

```ts
"/users"
```

throughout the project.

---

# Styling

Prefer

TailwindCSS

Use Shadcn components before creating custom components.

Avoid inline styles.

---

# Component Reuse

Before creating a new component:

Search for an existing one.

Do not duplicate components.

---

# Icons

Use one icon library consistently.

Recommended

Lucide React

---

# Data Formatting

Formatting belongs in utility functions.

Examples

```
formatCurrency()

formatDate()

formatPhone()
```

Avoid formatting inline.

---

# Date Handling

Store dates in UTC.

Convert only when displaying.

Never rely on browser locale defaults.

---

# API Responses

Never use raw API responses directly.

Transform responses if necessary.

Use typed models.

---

# TypeScript

Always:

- strict mode
- readonly where applicable
- explicit return types
- no implicit any

Prefer

type

for DTOs.

interface

for contracts.

---

# File Naming

Components

```
UserCard.tsx
```

Hooks

```
use-users.ts
```

Services

```
user.service.ts
```

API

```
user.api.ts
```

Schema

```
create-user.schema.ts
```

Types

```
user.types.ts
```

---

# Imports

Use path aliases.

```
@domains

@shared

@app
```

Avoid

```
../../../../../
```

---

# Performance

Always

- memoize expensive calculations
- lazy load routes
- paginate tables
- virtualize very large lists
- debounce searches

Avoid unnecessary re-renders.

---

# React Performance

Do not prematurely optimize.

Use

- useMemo
- useCallback
- React.memo

only when profiling indicates they provide a measurable benefit.

---

# Error Boundaries

Use React Error Boundaries.

Every major layout should have one.

---

# Suspense

Use Suspense for lazy routes.

---

# Accessibility

Always

- semantic HTML
- labels
- keyboard navigation
- aria attributes where necessary

---

# Testing

Prefer

- React Testing Library
- Vitest

Test:

- hooks
- business logic
- user interactions

Avoid snapshot-heavy testing.

---

# Code Quality

Required

- ESLint
- Prettier
- TypeScript
- Husky
- lint-staged

---

# Logging

Never use

```
console.log()
```

Use a logging utility.

---

# AI Assistant Rules

When generating React code:

1. Follow domain architecture.
2. Never place business logic inside components.
3. Use TanStack Query for server state.
4. Use React Hook Form with Zod.
5. Keep page components thin.
6. Create reusable hooks.
7. Create reusable services.
8. Never call Axios directly from components.
9. Prefer composition.
10. Reuse existing components before creating new ones.
11. Use TypeScript strict mode.
12. Never use any.
13. Keep changes minimal.
14. Preserve existing APIs.
15. Follow project conventions.
16. Do not introduce unnecessary dependencies.
17. Prefer refactoring over rewriting.
18. Keep UI components presentational.
19. Separate server state from UI state.
20. Never violate these standards simply because a shorter implementation exists.

---

# Pull Request Checklist

Before merging:

- [ ] ESLint passes
- [ ] Prettier passes
- [ ] TypeScript passes
- [ ] No `any`
- [ ] No duplicate components
- [ ] No business logic in components
- [ ] API calls only through services/API layer
- [ ] TanStack Query used for server state
- [ ] Forms use React Hook Form + Zod
- [ ] Components remain small and reusable
- [ ] No unnecessary re-renders
- [ ] No dead code
- [ ] No unused imports
- [ ] No console.log
- [ ] Tests updated where applicable