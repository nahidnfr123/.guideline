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
- Zustand (global client state — auth, theme, feature flags only)

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
│   ├── config/
│   ├── hooks/
│   ├── lib/
│   ├── store/
│   ├── types/
│   ├── constants/
│   └── utils/
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

    store/        (only if the domain owns global state, e.g. auth)

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

## Query Keys

Every domain defines its query keys in a single file, following a fixed hierarchy: `all` → `lists`/`details` → specific filters/ids. Never inline raw arrays as query keys inside components or hooks.

```ts
// domains/users/api/query-keys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};
```

Usage

```ts
// query
useQuery({ queryKey: userKeys.detail(userId), queryFn: () => getUser(userId) });

// invalidate all user lists after a mutation
queryClient.invalidateQueries({ queryKey: userKeys.lists() });

// invalidate everything user-related
queryClient.invalidateQueries({ queryKey: userKeys.all });
```

Bad

```ts
useQuery({ queryKey: ['users', userId], queryFn: ... }); // inconsistent shape, not reusable for invalidation
```

Rules:

- Query key factories live in `domains/<domain>/api/query-keys.ts`, exported from the domain's `index.ts` only if another domain legitimately needs to invalidate them (rare — prefer events/callbacks instead).
- Filters/params passed into a list key must be the same normalized object used in the query function, so cache entries don't fragment.

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

# Application Layers

The call chain is fixed and must not be skipped:

```
Component → Hook → Service → API layer → HTTP
```

- **Component** — renders UI, calls a hook, nothing else.
- **Hook** — owns TanStack Query (`useQuery`/`useMutation`), local state, and calls a service. This is the only place a component talks to.
- **Service** — pure functions that orchestrate one or more API calls, combine results, apply business rules. No React, no hooks, no query state.
- **API layer** — thin wrappers around Axios. One function per endpoint. No business logic.

A hook may call the API layer directly **only** when the operation is a single, unmodified request with no orchestration (e.g. `useUsers()` calling `getUsers()` directly is fine). As soon as a second endpoint, a transformation, or a business rule is involved, extract a service.

Bad — hook skips the service layer for a multi-step operation

```ts
function useCreateOrder() {
  return useMutation({
    mutationFn: async (input: CreateOrderInput) => {
      const cart = await api.get(`/carts/${input.cartId}`);
      const order = await api.post('/orders', { ...input, items: cart.data.items });
      await api.post(`/carts/${input.cartId}/clear`);
      return order.data;
    },
  });
}
```

Good — orchestration lives in the service

```ts
// domains/orders/services/order.service.ts
export async function createOrderFromCart(input: CreateOrderInput) {
  const cart = await getCart(input.cartId);
  const order = await createOrder({ ...input, items: cart.items });
  await clearCart(input.cartId);
  return order;
}

// domains/orders/hooks/use-create-order.ts
export function useCreateOrder() {
  return useMutation({
    mutationFn: createOrderFromCart,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: orderKeys.all }),
  });
}
```

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

## Global State

Global state is limited to authentication, theme, and feature flags — never server data. Use **Zustand** for these; do not use Context for state that changes over time (Context is fine only for static, rarely-changing values like a theme *config* injected once).

```ts
// domains/auth/store/auth.store.ts
type AuthState = {
  user: User | null;
  setUser: (user: User | null) => void;
  logout: () => void;
};

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
}));
```

Rules:

- One store per concern (`auth.store.ts`, `theme.store.ts`, `feature-flags.store.ts`) — no single monolithic app store.
- Stores live inside the domain they belong to (`domains/auth/store`), except cross-cutting ones like theme/feature-flags, which live in `shared/store`.

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

Split by extracting sub-components first (tables, modals, form sections), then by extracting hooks if the remaining logic is still heavy. A component that's long because it renders many distinct UI regions should become several presentational components; a component that's long because of state/effects/handlers should push that logic into a hook.

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

Every layout wraps its children in an Error Boundary (see **Error Handling**).

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

# Error Handling

## Typed API Errors

Never let a raw Axios error reach a component. Transform it in the API layer into a typed shape.

```ts
// shared/api/api-error.ts
export type ApiError = {
  status: number;
  message: string;
  code?: string;
};

export function toApiError(error: unknown): ApiError {
  if (axios.isAxiosError(error)) {
    return {
      status: error.response?.status ?? 0,
      message: error.response?.data?.message ?? 'Unexpected error',
      code: error.response?.data?.code,
    };
  }
  return { status: 0, message: 'Unexpected error' };
}
```

```ts
// domains/users/api/user.api.ts
export async function getUser(id: string) {
  try {
    const { data } = await api.get<User>(`/users/${id}`);
    return data;
  } catch (error) {
    throw toApiError(error);
  }
}
```

## Query/Mutation Errors

Handle expected errors at the hook or component level via TanStack Query's `error` state. Use a global `QueryCache`/`MutationCache` `onError` only for unexpected/unhandled errors (e.g. to fire a generic toast or log to the monitoring service).

```ts
// app/providers/query-client.ts
export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      if (query.meta?.silent) return;
      logger.error('Query failed', { error, queryKey: query.queryKey });
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      toast.error(isApiError(error) ? error.message : 'Something went wrong');
    },
  }),
});
```

## Error Boundaries

Every layout in `app/layouts` wraps its children in an Error Boundary. Boundaries catch render errors, not query errors — query errors are handled by TanStack Query as above.

```tsx
// app/layouts/dashboard-layout.tsx
<ErrorBoundary fallback={<DashboardErrorFallback />}>
  <Outlet />
</ErrorBoundary>
```

Rules:

- Never display a raw `error.message` from an unknown/unhandled error to the user; map known `ApiError.code` values to friendly copy, and fall back to a generic message otherwise.
- Every mutation that changes data must handle its `onError` case, even if only to show a toast.

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

# Environment Configuration

All environment variables are validated at boot via a Zod schema. Never read `import.meta.env.*` directly outside this file.

```ts
// shared/config/env.ts
const envSchema = z.object({
  VITE_API_URL: z.string().url(),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']),
});

export const env = envSchema.parse(import.meta.env);
```

Bad

```ts
const apiUrl = import.meta.env.VITE_API_URL; // no validation, no type safety
```

Good

```ts
import { env } from '@shared/config/env';
api.defaults.baseURL = env.VITE_API_URL;
```

If validation fails, the app should fail fast at startup with a clear console error rather than surface a confusing runtime bug later.

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

for DTOs (API payloads, request/response shapes).

interface

for contracts (props, service/class public APIs — anything meant to be extended or implemented).

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

Query Keys

```
query-keys.ts
```

Store

```
auth.store.ts
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

## Testing Requirements

Minimum bar, enforced in PR review:

| Layer | Required? |
|---|---|
| Services | Required |
| Hooks (business logic, not pure passthroughs) | Required |
| Zod schemas (edge cases) | Required |
| Presentational components | Optional |
| Pages | Optional (covered by e2e if present) |

Avoid snapshot-heavy testing; assert on behavior (rendered text, called handlers, resulting state) rather than serialized markup.

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

`console.log` is banned in application code (ESLint rule: `no-console`, error level). Use the shared logger.

```ts
// shared/lib/logger.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

function log(level: LogLevel, message: string, context?: Record<string, unknown>) {
  if (import.meta.env.DEV) {
    console[level === 'debug' ? 'log' : level](`[${level.toUpperCase()}] ${message}`, context ?? '');
    return;
  }
  // production: forward to monitoring (Sentry, LogRocket, etc.)
  monitoringClient.capture(level, message, context);
}

export const logger = {
  debug: (message: string, context?: Record<string, unknown>) => log('debug', message, context),
  info: (message: string, context?: Record<string, unknown>) => log('info', message, context),
  warn: (message: string, context?: Record<string, unknown>) => log('warn', message, context),
  error: (message: string, context?: Record<string, unknown>) => log('error', message, context),
};
```

Usage

```ts
logger.error('Failed to create order', { orderId, error });
```

Rules:

- No `console.*` calls outside `shared/lib/logger.ts` itself.
- Never log tokens, passwords, or full request/response bodies containing PII.

---

# AI Assistant Rules

When generating React code:

1. Follow domain architecture.
2. Never place business logic inside components.
3. Use TanStack Query for server state, with query keys from a domain's `query-keys.ts`.
4. Use React Hook Form with Zod.
5. Keep page components thin.
6. Create reusable hooks.
7. Create reusable services; hooks call services, services call the API layer (see **Application Layers**).
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
19. Separate server state from UI state; use Zustand only for auth/theme/feature flags.
20. Type and map errors before they reach the UI; never leak raw error messages.
21. Use the shared `logger`, never `console.*`.
22. Validate environment variables through `shared/config/env.ts`.
23. Never violate these standards simply because a shorter implementation exists.

---

# Pull Request Checklist

Before merging:

- [ ] ESLint passes
- [ ] Prettier passes
- [ ] TypeScript passes
- [ ] No `any`
- [ ] No duplicate components
- [ ] No business logic in components
- [ ] API calls only through the API layer; orchestration only in services
- [ ] TanStack Query used for server state, with query keys from a domain's `query-keys.ts`
- [ ] Forms use React Hook Form + Zod
- [ ] Errors are typed and mapped to user-facing messages; no raw error leaks to UI
- [ ] Mutations handle `onError`
- [ ] No `console.*`; uses `logger`
- [ ] New env vars added to `env.ts` schema
- [ ] Components remain small and reusable
- [ ] No unnecessary re-renders
- [ ] No dead code
- [ ] No unused imports
- [ ] Tests updated for services/hooks/schemas per Testing Requirements