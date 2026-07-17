# Express.js (TypeScript) Coding Standards

This document defines the coding standards for all Express.js/TypeScript projects. Every developer and AI coding assistant (Claude Code, Opencode, Antigravity, ChatGPT, Copilot, Cursor, etc.) must follow these standards consistently.

Unlike Laravel, Express has no built-in architectural opinion — it is a routing library, not a framework. Because of that, **domain-based architecture is the default for every project, regardless of size.** Starting flat ("controllers/services/utils") is how Express projects historically end up as an unmaintainable pile of loosely related files; starting with domains avoids that entirely and costs nothing extra on day one.

---

# Architecture Principles

- **High cohesion:** Group related logic together, by domain, not by technical layer alone.
- **Low coupling:** Keep domains independent; a domain should not reach into another domain's internals.
- **Dependency inversion:** Depend on interfaces/abstractions, not concrete implementations.
- **Single responsibility:** A class, function, or module should have one reason to change.
- **Explicit dependencies:** Inject dependencies through constructors or factory functions — never resolve them implicitly from a global singleton.
- **Favor composition over inheritance:** Compose small, focused functions/classes rather than building deep class hierarchies.

---

# Core Principles

- Follow SOLID principles.
- Enable TypeScript `strict` mode in every project. No exceptions.
- Prefer readability over clever code.
- Keep functions and methods small and focused.
- Avoid duplicated logic (DRY).
- Never write code that "just works"; write code that is maintainable and typed.
- Every piece of business logic should have a single responsible location — a rule duplicated across a Controller and a background Job is a rule that was put in the wrong place the first time.

---

# Business Rules

Business rules belong in:

- **Actions**
- **Services**
- **Value Objects / Domain Models**
- **Policies** (authorization-adjacent domain rules, e.g. `canOrderBeCancelled()`)

Business rules must never live inside:

- Controllers
- Route files
- Frontend/client code
- Migrations
- Seeders/factories
- Middleware (beyond auth/validation/logging concerns)

If a decision, calculation, or validation rule reflects how the business actually operates — pricing, eligibility, status transitions, permissions, limits — it belongs in one of the locations above, expressed as a named, testable function or method.

---

# Control Flow

Prefer early returns (guard clauses) over deeply nested conditionals. Aim to avoid more than 2–3 levels of nesting.

### Bad

```ts
function process(user?: User) {
  if (user) {
    if (user.isActive) {
      // ...
    }
  }
}
```

### Good

```ts
function process(user?: User) {
  if (!user) return;
  if (!user.isActive) return;

  // ...
}
```

---

# Architecture: Domain-Based by Default

Regardless of project size — a weekend prototype or a multi-team monolith — organize by **domain first, technical layer second.** A domain owns everything it needs to fulfill its responsibility.

```
src/
│
├── app.ts                     # Express app assembly (middleware, routers), no server.listen()
├── server.ts                  # Entry point: creates HTTP server, starts listening
│
├── config/                    # Environment loading & validation (see "Configuration")
├── database/                  # DB connection, migrations, seeders
├── middleware/                # App-wide middleware (auth guard, error handler, request logger)
│
├── shared/
│   ├── errors/                 # Base error classes (AppError, NotFoundError, ValidationError, ...)
│   ├── enums/
│   ├── helpers/
│   ├── logger/
│   ├── dto/                    # Cross-domain DTOs only
│   ├── types/
│   ├── constants/
│   └── utils/
│
├── infrastructure/             # Adapters to the outside world — no business logic
│   ├── cache/                  # Redis client wrapper
│   ├── email/                  # Mail provider client (SES/SendGrid/etc.)
│   ├── queue/                  # BullMQ/SQS client wrapper
│   ├── storage/                # S3/GCS client wrapper
│   └── external/               # Third-party API clients (Stripe, HubSpot, etc.)
│
└── domains/
    ├── auth/
    │   ├── actions/
    │   ├── controllers/
    │   ├── dto/
    │   ├── middleware/          # Domain-specific middleware (e.g. requireVerifiedEmail)
    │   ├── models/
    │   ├── policies/
    │   ├── repositories/
    │   ├── routes/
    │   ├── services/
    │   ├── validators/          # Zod/Joi schemas
    │   ├── interfaces/
    │   ├── types/
    │   └── index.ts             # Public surface of the domain — see "Domain Boundaries"
    │
    ├── users/
    ├── orders/
    └── products/
```

## Domain Boundaries

- Each domain exposes a single public surface through its `index.ts` (routes + any types other domains legitimately need). Everything else inside a domain folder is private to that domain.
- **A domain must not import another domain's `services/`, `repositories/`, `actions/`, or `models/` directly.** If domain A genuinely needs domain B's behavior, it goes through B's exported service interface (from `domains/orders/index.ts`), or the two communicate via a domain event.
- Cross-cutting concerns (auth guard, request ID, logging) live in top-level `middleware/`, not duplicated per domain.
- A domain folder is not required to populate every subfolder — an `orders` domain with no caching need has no `policies/` folder just to match the template. Create subfolders when the domain has content for them (see "No Ceremony" principle below, mirrored from the Laravel `Actions` guidance).

## Scaling Within Domain-Based Architecture

Domain-based is the default *structure*, but the same "don't manufacture abstraction" judgment from the Laravel standard still applies inside each domain:

- **Small domain / early project:** a domain might only need `routes/`, `controllers/`, `services/`, `dto/`, `validators/`. Skip `actions/` and `repositories/` until reuse or complexity justifies them.
- **Growing domain:** introduce `actions/` once a business operation is called from more than one place (a Controller and a Job, for example). Introduce `repositories/` once a Model's queries need abstraction (see "Repositories").
- **Large/critical domain:** may itself be split into sub-modules if it's grown multiple independent responsibilities (e.g. `orders/fulfillment`, `orders/billing`) — a signal the domain should arguably become two domains.

---

# Dependency Rules

## Allowed:

```
Router (HTTP Boundary)
    │
    ▼
Controller (parses req, calls Application Layer, shapes res)
    │
    ▼
Application Layer (Service, Action, or Service delegating to Actions)
    │
    ▼
Data Access Layer (Repository if one exists, otherwise Model/ORM directly)
```

The Application Layer is not required to be both a Service *and* an Action. It may be:

- a **Service** alone (simple orchestration, no reusable sub-steps),
- an **Action** alone (one Controller calling one business operation), or
- a **Service that delegates to Actions** (coordinating multiple reusable operations).

**Notes on this chain:**

- Controllers must never bypass the Application Layer to touch persistence directly.
- The Repository layer is optional (see "Repositories"). When none exists for a Model, the Application Layer queries the ORM directly (`prisma.order.findMany(...)`, `Order.find(...)`, etc.).
- Controllers may only call the Application Layer — never Repositories or Models directly.

## Disallowed:

- Controllers calling repositories or models/ORM directly.
- Models/entities dispatching business workflows or calling Services.
- Services depending on Controllers or `Request`/`Response` objects.
- Circular imports between domains, or between layers within a domain.

## Services and Data Access

Services should reuse an existing Action for data access when one already exists for that operation — but this is a consistency preference, not a rigid layering rule. A `DashboardService` doing `prisma.user.count()` and `prisma.order.count()` inline is fine; manufacturing `CountUsersAction`/`CountOrdersAction` purely to satisfy a diagram adds ceremony without value.

The judgment call: if an Action for that query already exists, use it. If it doesn't, and creating one wouldn't be reused elsewhere, querying the Model/ORM directly from the Service is acceptable.

---

# Framework Independence

Business logic must not depend on the Express request lifecycle. Any authenticated user, tenant, locale, timezone, or other request-specific context must be passed explicitly into Services and Actions as function arguments or DTOs — never pulled from a global or from `req` deep inside business logic.

## Prohibited in Services, Actions, Repositories, Models, and Value Objects:

- Importing or receiving `Request`/`Response` (`express.Request`, `express.Response`) as a parameter.
- Reading `req.headers`, `req.session`, `req.cookies` from anywhere but a Controller or middleware.
- Calling `res.send()`, `res.json()`, `res.redirect()`, `res.status()` — response shaping is a Controller concern only.
- Reading a global "current user"/"current tenant" singleton (e.g. an `AsyncLocalStorage`-backed global helper used as an implicit context grab) instead of receiving it as a parameter. `AsyncLocalStorage` is acceptable **only** as the mechanism a Controller/middleware uses to *extract* context — Services/Actions still receive that context explicitly as arguments, they do not reach into the store themselves.

## Value Objects Must Be Pure

Value Objects (domain value types) have a stricter bar than Services/Actions: no I/O of any kind. In addition to the above, Value Objects must not:

- Read `process.env` or any config module.
- Call a cache client, storage client, or database client.
- Perform network calls.

A Value Object must be fully constructible and usable from its constructor arguments alone. If a value needs configuration or persisted state to compute, that computation belongs in a Service or Action, which then constructs the Value Object with the result.

## Configuration Should Be Injected at the Boundary

Business logic (Services and Actions) should receive configuration through constructor injection or function arguments — not by calling into the config module directly throughout the codebase.

### Bad

```ts
class InvoiceService {
  calculateLateFee(invoice: Invoice): number {
    const rate = config.get('billing.lateFeeRate'); // hidden dependency on global config
    return invoice.total * rate;
  }
}
```

### Good

```ts
class InvoiceService {
  constructor(private readonly lateFeeRate: number) {}

  calculateLateFee(invoice: Invoice): number {
    return invoice.total * this.lateFeeRate;
  }
}
```

Resolve the concrete value once, at composition time (see "Configuration"), and inject it. This keeps dependencies explicit and makes the class trivial to test with different values, without needing to mock a config module in every test.

## Allowed Context Retrieval Exceptions

You may read request/session context only within:

- Middleware
- Controllers
- Route-level validation (Zod/Joi schema execution against `req.body`/`req.query`)
- Policies (when they need `req.user` to make an authorization decision at the boundary)

## Context Ingestion Principle

Controllers are responsible for extracting framework-specific data. Services and Actions receive everything through function parameters or DTOs.

### Bad

```ts
class CreateOrderAction {
  async handle(req: Request) {
    const user = req.user; // Bad: hidden HTTP dependency
    // ...
  }
}
```

### Good

```ts
class CreateOrderAction {
  async handle(user: User, data: CreateOrderData) {
    // ...
  }
}
```

In the Controller:

```ts
async function store(req: Request, res: Response) {
  const order = await createOrderAction.handle(req.user, req.body as CreateOrderData);
  res.status(201).json(OrderResource.fromEntity(order));
}
```

---

# Actions vs Services

## Actions

Actions represent a single business operation or use case, including the data access needed for that operation.

### Characteristics:
- One responsibility
- One public `handle()`/`execute()` method
- Easily unit-testable
- Reusable across Controllers, Jobs, and Console scripts
- May query the Model/ORM or a Repository directly

### Examples:
- `CreateUserAction`
- `ApproveOrderAction`
- `AssignRoleAction`
- `UploadDocumentAction`

### Example

```ts
export class CreateUserAction {
  constructor(private readonly userRepository: UserRepository) {}

  async handle(data: CreateUserData): Promise<User> {
    return this.userRepository.create(data);
  }
}
```

## Action Dependencies

An Action may depend on other small helper utilities, repositories, or value objects. Avoid Actions calling other Actions to build up larger workflows — that is workflow orchestration and belongs in a Service.

### Avoid

```
CreateUserAction
    ↓ (calls)
AssignRoleAction
    ↓ (calls)
SendWelcomeEmailAction
```

If Actions start chaining into each other, extract that chain into a Service and let each Action stay independently callable.

## Services

Services coordinate multiple Actions and orchestrate business workflows.

### Characteristics:
- Multiple methods allowed
- Orchestrates business processes
- Coordinates Actions, Repositories (via Actions), Queue jobs, and domain events
- Prefers reusing an existing Action for data access over querying the ORM inline, but may query directly for simple, non-reusable cases

### Example

```ts
export class UserOnboardingService {
  constructor(
    private readonly createUserAction: CreateUserAction,
    private readonly assignDefaultRoleAction: AssignDefaultRoleAction,
    private readonly sendWelcomeEmailAction: SendWelcomeEmailAction,
  ) {}

  async onboard(data: CreateUserData): Promise<User> {
    const user = await this.createUserAction.handle(data);
    await this.assignDefaultRoleAction.handle(user);
    await this.sendWelcomeEmailAction.handle(user);
    return user;
  }
}
```

## Choosing Between an Action and a Service

Introduce Actions when an operation becomes independently reusable — not simply "because that's the standard." A guideline that treats Actions as mandatory produces noise like `GetUserAction`, `FindUserAction`, `DeleteUserAction` for every trivial CRUD verb, reused nowhere. That's ceremony, not architecture.

Use an **Action** when:
- The operation performs one business task.
- It is, or is likely to be, reused independently (Controller, Service, Job, script).
- The class naturally exposes a single `handle()` method.

Use a **Service** when:
- Multiple Actions must be coordinated.
- The workflow spans several business operations.
- The process includes transactions, domain events, notifications, or external integrations.

Use a **Service with no Actions** when:
- The logic is simple orchestration or a single query/mutation with no reusable sub-steps.

---

# Controllers

Controllers should only:

- Parse and coerce the request (with validation already run by middleware/schema)
- Call a Service/Action
- Shape and return the response

### Bad

```ts
async function store(req: Request, res: Response) {
  const user = await prisma.user.create({ data: req.body });
  await sendMail(user.email, 'Welcome!');
  res.json(user);
}
```

### Good

```ts
async function store(req: Request, res: Response, next: NextFunction) {
  try {
    const user = await userService.create(req.body as CreateUserData);
    res.status(201).json(UserResource.fromEntity(user));
  } catch (err) {
    next(err);
  }
}
```

Controllers should not:

- Build ORM queries
- Perform calculations
- Send emails or dispatch external calls
- Contain loops with business rules
- Catch and swallow errors silently (see "Error Handling")

## Async Controllers

Every async Controller/route handler must forward rejected promises to `next()`. Either wrap handlers in a helper (`asyncHandler(fn)`) or use a router library/Express version with native async error propagation. An unhandled rejection in a route handler that isn't forwarded to `next()` will hang the request or crash the process depending on Node version — never rely on this happening "by default."

```ts
const asyncHandler =
  (fn: RequestHandler): RequestHandler =>
  (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
```

---

# Authorization

Authorization belongs in Policies, invoked at the Controller/middleware boundary — not inside Services or Actions.

Services and Actions assume the caller is already authorized. Do not call a Policy, a permission check, or `req.user.can(...)` from inside a Service or Action — that reintroduces an HTTP concern into business logic and duplicates a check that should already have happened at the boundary.

If a Service or Action needs to make an authorization-adjacent *business* decision (e.g. "can this order still be cancelled given its current state"), express it as a domain rule with a descriptive method (`order.canBeCancelled()`), not as a permission check.

```ts
// Middleware / Controller boundary
router.patch('/orders/:id/cancel', authorize(OrderPolicy, 'cancel'), controller.cancel);
```

---

# Services (Size Guidance)

A Service should have a single responsibility. Line count alone is not a reliable signal — a 300-line Service handling one cohesive workflow isn't automatically worse than four tiny Services bouncing calls between each other. When a Service becomes hard to understand, navigate, or test, extract Actions or split into additional Services based on *what* it's doing, not how many lines it takes.

---

# Models / ORM Entities

Models (Prisma schema models, TypeORM entities, Mongoose schemas) should represent data.

Allowed:
- Relationships/associations
- Query scopes/helpers scoped to the model's own concerns
- Computed/derived fields via getters
- Serialization helpers (`toJSON`)

Avoid:
- Business logic
- External API calls
- Complex multi-entity calculations

Complex reporting queries — multi-table joins, aggregations spanning several entities, dashboard-style queries — belong in dedicated Repositories or query objects, not ad-hoc statics bolted onto the Model.

## Mass Assignment

Never spread raw request bodies into a create/update call.

### Bad

```ts
await prisma.user.create({ data: req.body });
```

### Good

```ts
await prisma.user.create({
  data: {
    name: data.name,
    email: data.email,
    passwordHash: hashedPassword,
  },
});
```

Never let user-supplied input set fields that control authorization, ownership, or financial state (`role`, `isAdmin`, `balance`). Set these explicitly in the Action, never from a spread of client input.

## ORM Hooks (Prisma middleware / TypeORM subscribers / Mongoose pre/post hooks)

Avoid placing business workflows inside ORM lifecycle hooks.

Good uses:
- Setting derived/computed fields (slug generation, `updatedAt`)
- Cache invalidation
- Audit logging

Avoid:
- Sending emails
- Payment processing
- External API calls
- Cross-domain workflows (a `Order` hook that touches `Inventory` or `Billing`)

ORM hooks fire implicitly on every save — including from seed scripts and tests — which makes hidden workflows hard to trace and easy to trigger by accident. Prefer explicit Actions, Services, or domain events for anything beyond the "good uses" above.

---

# Soft Deletes & Auditing

## Soft Deletes

Use soft deletes only when business requirements dictate that deleted data must be restorable or retained for audit.

- Be explicit about excluding/including soft-deleted rows in every query — do not rely on an implicit global filter you might forget exists in a raw query or aggregation.
- Unique constraints must account for soft-deleted rows (e.g. a partial unique index that ignores `deletedAt IS NOT NULL`, or application-level validation that explicitly considers/excludes trashed records).

## Global Query Middleware/Scopes

Avoid complex global query middleware (e.g. a Prisma middleware that silently rewrites every query) that implicitly alters results across the application — it complicates unit testing and causes surprising bugs. Prefer explicit, named query helpers that developers call consciously (`OrderRepository.findActive()`).

## Activity Logging & Auditing

- Audit critical mutations, status changes, and authorization changes.
- Use an established audit approach (a dedicated `AuditLog` write inside the same transaction as the mutation) rather than ad-hoc scattered logging.
- Always log full metadata: the actor, the target entity, before/after of key fields, and contextual request metadata (IP, request ID, or CLI/job context).

---

# Validation

Always validate at the boundary using a schema library (Zod or Joi). Never trust `req.body`/`req.query`/`req.params` past the validation middleware.

Never call business logic directly on unvalidated input inside a Controller.

### Good

```ts
// domains/users/validators/create-user.schema.ts
export const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  password: z.string().min(8),
});

// Route
router.post('/users', validate(createUserSchema), controller.store);
```

## Organization of Validators

Group validators by domain and resource, mirroring the Form Request organization pattern:

```
domains/users/validators/create-user.schema.ts
domains/users/validators/update-user.schema.ts
```

Never call `.parse()`/`.validate()` inline inside a Service or Action — validation is a boundary concern; by the time a Service receives a typed DTO, it should already be trustworthy.

---

# Database Queries

Never execute queries inside loops.

### Bad

```ts
for (const user of users) {
  user.posts = await prisma.post.findMany({ where: { userId: user.id } });
}
```

### Good

```ts
const users = await prisma.user.findMany({ include: { posts: true } });
```

Always prevent N+1 queries — use `include`/`with`/`populate` (Prisma/TypeORM relations/Mongoose `.populate()`) or a DataLoader for GraphQL-style resolvers.

Prefer existence checks that short-circuit over counting.

### Bad

```ts
(await prisma.user.count({ where: { email } })) > 0;
```

### Good

```ts
(await prisma.user.findFirst({ where: { email }, select: { id: true } })) !== null;
// or, where supported:
await prisma.user.count({ where: { email }, take: 1 }); // still prefer findFirst
```

---

# Collection / Array Standards

Prefer native array methods (`.filter`, `.map`, `.reduce`, `.groupBy` via `Object.groupBy`/lodash) where they improve readability over manual loops.

Do not chain excessively. If a chain becomes hard to read, break it into named intermediate variables or a dedicated function.

### Good

```ts
const activeAdminEmails = users
  .filter((u) => u.isActive && u.role === Role.Admin)
  .map((u) => u.email);
```

---

# ORM-Specific Guidance

Prefer the ORM's query builder/relations over raw SQL.

```ts
prisma.user.findMany({ where: { isActive: true } });
```

instead of

```ts
prisma.$queryRaw`SELECT * FROM users WHERE is_active = true`;
```

unless a raw query is required for performance or a feature the ORM doesn't support — and if so, isolate it in a Repository method with a comment explaining why.

---

# Constructors

Constructors should only assign dependencies.

Avoid:
- Business logic
- Database/network calls
- File operations
- Reading `process.env` directly (see "Configuration")

Constructors must be lightweight and side-effect free — this also matters for testability, since constructing a class for a unit test should never trigger I/O.

---

# Dependency Injection

Always use constructor injection. Never `new` up a dependency inside a class that consumes it — inject it, or inject a factory.

### Bad

```ts
class OrderService {
  private emailClient = new EmailClient(); // hidden concrete dependency
}
```

### Good

```ts
class OrderService {
  constructor(private readonly emailClient: EmailClient) {}
}
```

## DI Container

For anything beyond a small project, use an explicit DI container (Awilix, tsyringe, or InversifyJS) rather than manually wiring every dependency by hand in one giant composition file. Register bindings once, at the composition root (`app.ts` or a dedicated `container.ts`).

```
UserRepositoryInterface  →  PrismaUserRepository
EmailClientInterface     →  SesEmailClient
```

Small projects with few dependencies may wire manually in a composition function instead of pulling in a full container — don't add a DI framework purely for ceremony (see AI Assistant Rule on unjustified abstraction).

Never resolve a dependency from the container inside business logic (`container.resolve(...)` called from inside a Service). Resolution happens once, at the composition root; everything downstream receives what it needs via constructor injection.

## Never Use Singleton Services With Mutable State

A Service instance shared across requests must not hold per-request mutable state as instance fields (e.g. accumulating a "current request's user" on `this`). Node.js processes serve many concurrent requests on one event loop — mutable instance state on a shared singleton is a race condition waiting to happen. Pass all per-request context as function arguments instead.

---

# Repositories

Repositories exist to abstract persistence. Do not create a Repository simply to wrap ORM CRUD 1:1.

Repositories should exist when they provide meaningful abstraction:
- Multiple data sources for the same concept
- Reusable complex queries
- Caching layered on top of reads
- An external system pretending to be a data source

When no Repository exists for a Model, the Application Layer queries the ORM directly. If a Repository is later introduced for a Model, migrate all existing direct-access code paths for that entity to use it — don't leave the same Model queried two different ways across the codebase.

---

# DTOs

Prefer DTOs (typed classes or interfaces) whenever a function receives more than 3–4 related parameters, or the parameters together represent a single business concept.

### Bad

```ts
function createUser(
  name: string,
  email: string,
  phone: string,
  country: string,
  city: string,
): Promise<User> { }
```

### Good

```ts
function createUser(data: CreateUserData): Promise<User> { }
```

### Naming Examples
```
CreateInvoiceData
CreateAppointmentData
UpdateForecastData
```

## Objects vs. DTOs at Layer Boundaries

Avoid passing loose, untyped objects (`Record<string, unknown>`, `any`) between Services and Actions for data that represents a business concept, even for a single parameter. An untyped object has no enforced shape — it only fails at runtime if a key is missing or misspelled, and it defeats the entire purpose of using TypeScript.

### Bad

```ts
function handle(data: Record<string, any>): Promise<Order> { }
```

### Good

```ts
function handle(data: CreateOrderData): Promise<Order> { }
```

DTOs should be plain, serializable shapes — prefer `type` for DTOs (see "Type vs Interface").

---

# Enums and Literal Unions

Use TypeScript enums or literal string unions instead of magic strings — never compare against a raw string scattered across the codebase.

### Bad

```ts
if (status === 'pending') { }
```

### Good — string union (preferred for most cases)

```ts
type OrderStatus = 'pending' | 'approved' | 'rejected';
```

### Good — enum (preferred when the values need to be iterated, or are shared with a database enum type)

```ts
enum OrderStatus {
  Pending = 'pending',
  Approved = 'approved',
  Rejected = 'rejected',
}
```

Prefer string literal unions over numeric enums — numeric enums serialize as numbers with no self-documenting value in logs/JSON and are easy to shift accidentally by inserting a value in the middle.

---

# Constants

Never hardcode values that represent a business rule or a repeated magic value.

### Bad

```ts
if (role === 'admin') { }
```

### Good

```ts
if (role === Role.Admin) { }
```

---

# Naming

Use descriptive names.

### Bad
```ts
const data = ...
const temp = ...
const x = ...
```

### Good
```ts
const customer = ...
const forecast = ...
const invoiceItem = ...
const totalRevenue = ...
```

## Class / File Naming Conventions

- **Actions:** `CreateUserAction`, `UpdateOrderAction` — file: `create-user.action.ts`
- **Services:** `UserService`, `PaymentService` — file: `user.service.ts`
- **Controllers:** `UserController` — file: `user.controller.ts`
- **Repositories:** `UserRepository` — file: `user.repository.ts`
- **DTOs:** `CreateUserData`, `UpdateOrderData` — file: `create-user.dto.ts`
- **Validators:** `createUserSchema` — file: `create-user.schema.ts`
- **Enums:** `UserStatus`, `OrderStatus` — file: `user-status.enum.ts`
- **Policies:** `UserPolicy`, `OrderPolicy` — file: `user.policy.ts`

Use kebab-case for filenames, PascalCase for classes/types, camelCase for functions/variables. Never mix casing conventions within the same category.

## Boolean Methods

Use descriptive prefixes:

### Prefer
- `isActive()`
- `hasPermission()`
- `canEdit()`
- `shouldSync()`

### Avoid
- `check()`
- `verify()`
- `flag()`

---

# Method / Function Size

Line count alone is a poor quality metric. A function should fit comfortably on one screen and express one clear idea. Extract a helper when it improves readability or reveals a reusable concept — not to satisfy an arbitrary line count.

---

# Class / File Size

For Actions, Repositories, and Models/Entities, line count is a useful early-warning signal:

- **over 300 lines** — treat this as the trigger to review the class for split-worthy responsibilities.
- **over 500 lines** — must be split before merging, except where a maintainer explicitly signs off (e.g. large generated types, Prisma client extensions).

For **Services**, follow the responsibility-based guidance above instead of a fixed threshold.

---

# TypeScript Standards

- Enable `strict: true` in `tsconfig.json` for every project — no exceptions, no gradual "we'll add it later."
- **Never use `any`** unless truly unavoidable (e.g. typing a third-party callback with no published types) — and when it is used, isolate it behind a typed wrapper function immediately, with a comment explaining why. Prefer `unknown` over `any` when the type genuinely isn't known, and narrow before use.
- Declare explicit return types on all exported/public functions and class methods. Return type inference is fine for small private/local helpers, but a public API's return type should never be implicit — it's part of the contract and should be visible at the call site without navigating to the implementation.
- Use `readonly` on properties that are not meant to change after construction, and on array/tuple types passed as parameters where mutation is not intended (`readonly string[]`).
- Use typed properties everywhere; avoid implicit `any` from an unannotated field.
- Prefer `unknown` over `any` for values with a genuinely unknown external shape (parsed JSON, third-party responses) and validate/narrow with a schema before use.

## `type` vs `interface`

- Prefer `interface` for object shapes that represent a class contract or that other code might want to `extend`/merge (public API surfaces, plugin contracts).
- Prefer `type` for DTOs, unions, mapped types, and anything that is a fixed data shape not meant to be extended.
- Don't mix both for the same concept in the same domain — pick one and stay consistent within a module.

```ts
// Contract meant to be implemented by multiple classes → interface
export interface EmailClient {
  send(to: string, subject: string, body: string): Promise<void>;
}

// Fixed data shape → type
export type CreateUserData = {
  name: string;
  email: string;
  password: string;
};
```

## Avoid Circular Imports

A domain must not have two modules that import each other, directly or transitively. If module A needs something from module B and B needs something from A, extract the shared piece into `shared/` or restructure so the dependency points one direction only. Use a lint rule (`eslint-plugin-import`'s `no-cycle`) to catch this in CI rather than relying on developers noticing.

## Barrel Exports

Use an `index.ts` barrel export at the top of each domain to define its public surface (see "Domain Boundaries"). Avoid deep barrel chains inside a domain purely for convenience (`actions/index.ts` re-exporting everything) — they slow down IDE navigation and increase the odds of a circular import. One barrel per domain boundary is enough; internal files should be imported directly by relative path.

## Path Aliases

Configure path aliases (`@domains/*`, `@shared/*`, `@infrastructure/*`) in `tsconfig.json` and the bundler/test runner config so imports don't degrade into `../../../../shared/utils`. Keep the alias list small and mapped 1:1 to the top-level folders in this document's structure — don't create an alias per individual subfolder.

## Avoid Static Helper Classes

Prefer an injectable service (even a small one) over a class of static methods when the logic has any dependency at all (config, a client, another service) — static methods can't be mocked via DI and quietly become a hidden global dependency. A true, dependency-free pure function (e.g. `formatCurrency`) is fine as a plain exported function; it doesn't need to be a class at all.

---

# API Responses / Resources

Always transform entities through a Resource/serializer before returning them. Never return an ORM entity directly.

### Good

```ts
export class UserResource {
  static fromEntity(user: User) {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
      // passwordHash intentionally omitted
    };
  }
}
```

---

# API Standards

- Use Resources/serializers for every response.
- Return consistent response structures.
- Use proper HTTP status codes.
- Paginate collection endpoints (cursor-based pagination for large/real-time datasets; offset-based is acceptable for small, stable admin lists).
- Never expose internal fields (password hashes, internal IDs meant to stay opaque, raw stack traces).
- Version any API consumed by external clients or a separately-deployed frontend/mobile app.

## API Versioning

Use URL-based versioning (`/api/v1/...`) as the default. Group versioned routers under a `v1`/`v2` namespace so breaking changes to `v2` don't touch `v1` code paths. Internal-only APIs consumed exclusively by a monolith's own first-party frontend do not require versioning.

## API Error Responses

Use a standardized JSON error structure for every error response, produced by the global error-handling middleware — never let individual routes format their own ad-hoc error shape.

### Standard Format

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "email": ["The email field is required."]
  }
}
```

---

# Error Handling

Never swallow errors.

### Bad

```ts
try {
  await service.process(order);
} catch (err) {
  // nothing
}
```

### Good

```ts
try {
  await service.process(order);
} catch (err) {
  logger.error('Failed to process order', { error: err, orderId: order.id });
  throw err;
}
```

Context is everything — a bare `logger.error(err)` gives a stack trace with no idea which order, user, or request triggered it. Always log the error alongside identifiers needed to reproduce or locate the affected record.

## Custom Error Classes

Define a small hierarchy of typed application errors (`AppError`, `NotFoundError`, `ValidationError`, `UnauthorizedError`, `ConflictError`) in `shared/errors/`, each carrying an HTTP status code and a machine-readable code. Throw these from Services/Actions instead of throwing raw strings or generic `Error`.

```ts
export class NotFoundError extends AppError {
  readonly statusCode = 404;
  constructor(entity: string, id: string) {
    super(`${entity} with id ${id} not found`);
  }
}
```

## Global Error Middleware

Register exactly one global error-handling middleware, last in the middleware chain, that:

- Maps known `AppError` subclasses to their status code and the standard error JSON shape.
- Maps validation library errors (Zod/Joi) to the standard `errors` field shape.
- Logs unexpected (non-`AppError`) errors at `error` level with full context, and returns a generic `500` body — never leak a raw stack trace or internal error message to the client in production.

```ts
app.use((err: unknown, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({ message: err.message, errors: err.details });
  }
  logger.error('Unhandled error', { error: err, requestId: req.id });
  res.status(500).json({ message: 'Internal server error' });
});
```

## Async Error Handling

Every route handler, middleware, and job processor that returns a Promise must have its rejection routed somewhere it will be handled — either via `asyncHandler` wrapping (see "Controllers") or a router that natively forwards async errors. Never rely on an unhandled rejection surfacing as a visible crash; add a process-level `unhandledRejection` handler as a last-resort safety net that logs and (in most setups) triggers a graceful shutdown, not as the primary error-handling strategy.

---

# Side Effects

Separate calculations from side effects whenever practical. Pure functions — ones whose purpose is to compute or decide something — should not send emails, write logs, dispatch jobs/events, mutate unrelated entities, or perform other I/O.

### Bad

```ts
function calculateDiscount(order: Order): number {
  const discount = order.total * 0.1;
  logger.info('Discount calculated', { orderId: order.id }); // side effect in a calculation
  order.discountApplied = true; // mutates unrelated state
  return discount;
}
```

### Good

```ts
function calculateDiscount(order: Order): number {
  return order.total * 0.1;
}
```

The caller (an Action or Service orchestrating the workflow) is responsible for logging, dispatching, and persisting. This makes pure logic trivially unit-testable without mocks, and makes it obvious whether calling a function twice is safe.

---

# Transactions

Wrap multi-step database writes in a transaction.

```ts
await prisma.$transaction(async (tx) => {
  await tx.order.update({ where: { id: orderId }, data: { status: 'paid' } });
  await tx.inventory.decrement({ where: { productId }, data: { quantity: qty } });
});
```

Avoid performing external API calls (Stripe, HubSpot, third-party sync) inside a database transaction whenever possible. Commit the transaction first, then dispatch a job/event that triggers the external call. Holding a database connection/lock while waiting on a third-party network response is a common source of contention.

## Transaction Ownership

Only the outermost orchestration layer (typically a Service) should open a transaction. Actions should assume they already execute inside the correct transactional context (by receiving the transaction client — e.g. Prisma's `tx` — as a parameter) and should not open their own transaction unless genuinely called standalone, outside any Service.

Avoid nested transactions unless isolation is intentionally required. If an Action needs transactional safety alone *and* needs to compose inside a Service-level transaction, make the transaction boundary the caller's responsibility (pass the client in) rather than baking it into the Action.

---

# Queues

Queue work that involves:

- External API calls (payment gateways, third-party sync)
- Sending emails or notifications
- Imports/exports
- Calculations expected to take longer than ~1 second synchronously
- Any operation that would otherwise block the HTTP response

Do not queue trivial in-memory operations or a single fast (< ~100ms) DB write — the overhead of enqueueing and processing a job exceeds the cost of doing it inline. When in doubt, measure.

Use an established queue library (BullMQ backed by Redis, or SQS for cloud-native setups) rather than a hand-rolled polling mechanism.

---

# Jobs / Workers

A Job/worker processor should coordinate an existing Service or Action, not duplicate business logic inline.

### Bad

```ts
export async function processInvoiceJob(job: Job) {
  // validates, queries, calculates, updates — all written inline,
  // only reachable via the queue
}
```

### Good

```ts
export function processInvoiceJob(invoiceService: InvoiceService) {
  return async (job: Job<{ invoiceId: string }>) => {
    await invoiceService.process(job.data.invoiceId);
  };
}
```

Keeping the logic in the Service means it stays reusable and testable outside the queue (called synchronously from a script or another workflow) instead of being locked inside code only reachable by dispatching a real or faked job.

---

# Events

Use events (an in-process `EventEmitter`, or a domain-event dispatcher) when multiple listeners need to react to something that happened. Avoid events as a substitute for a normal function call — if there is exactly one thing that needs to happen next, and it will always need to happen, just call it directly.

## Domain Events

Prefer domain events over tightly coupling unrelated services to one another.

### Bad

```
OrderService → EmailService → SlackService → CrmService
```

### Good

```
OrderCreated event
    ↓
SendOrderConfirmationEmail listener
UpdateCrmRecord listener
NotifySlackChannel listener
```

`OrderService` only needs to know that an order was created and emit the event — it should not know or care who reacts to it. This keeps unrelated concerns decoupled and independently testable, and lets new reactions be added without modifying `OrderService`.

Use a direct call (not an event) when the caller genuinely needs the result synchronously, or when there is exactly one tightly-related consequence that will realistically never need to vary independently.

---

# Configuration

Never hardcode configuration values. Never read `process.env` outside the configuration layer.

## Centralize and Validate Configuration Loading

Load and validate all environment variables **once**, at startup, through a single typed config module — using a schema (Zod) so the process fails fast at boot with a clear error if a required variable is missing or malformed, rather than failing obscurely deep in a Service at request time.

```ts
// config/env.ts
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  PORT: z.coerce.number().default(3000),
  LATE_FEE_RATE: z.coerce.number(),
});

export const env = envSchema.parse(process.env);
```

`process.env` should be read in exactly this one file. Every other module imports the parsed, typed `env` object (or receives the specific value it needs via constructor injection) — never `process.env.SOMETHING` scattered throughout Services, Actions, or Controllers.

Never hardcode:
- URLs / API endpoints
- Feature flags
- Timeouts
- Retry counts
- Cache durations

Place them in `config/`, sourced from validated environment variables or a config file.

---

# Caching

Cache:
- Expensive queries
- Dashboard/aggregate statistics
- Computed configuration
- External API results with a sensible TTL

Access the cache only through the `infrastructure/cache` wrapper — never instantiate a Redis client ad hoc inside a Service.

---

# Testing

Prefer:
- **Integration tests** — exercise the full stack (Router → Controller → Service → Action → DB) via `supertest` against a real or in-memory/test database. Use these to verify authorization, validation, and response shape.
- **Unit tests** — target a single Action or Service in isolation, with collaborators mocked via constructor injection so the class under test has no hidden dependencies.

## Testing Conventions

- **Database state management:** Use a dedicated test database, reset between test files/suites (via transactions rolled back per test, or a full reset script) so tests never depend on execution order.
- **Assertions:** Prefer asserting against the actual persisted row (`findUnique`) or the HTTP response body over checking side effects indirectly.
- **Mocking external services:** Never make real network calls in tests. Use `nock`/`msw` to intercept HTTP, or fake queue/email clients bound via the DI container specifically for the test environment.
- **Do not mock the ORM itself** in unit tests where avoidable — prefer a real test database (SQLite/Postgres in a test container) so tests fail against real query/schema mismatches instead of passing against a mock that's silently drifted from reality. Mock the ORM only at the Repository boundary when testing a Service/Action in isolation from persistence entirely.
- **Policies** should have dedicated authorization tests (allowed/denied cases) separate from integration tests of the underlying endpoint.
- **Avoid testing framework internals** (e.g. that Zod rejects a missing required field) — test your own business rules and edge cases.

Every new business feature should include tests where practical.

---

# Code Quality Tooling

## Required
- **ESLint** (with `@typescript-eslint`, `eslint-plugin-import` including `no-cycle`)
- **Prettier** (formatting)
- **Husky + lint-staged** (pre-commit: lint, format, type-check the staged files)
- **Jest or Vitest** (testing)
- **TypeScript compiler (`tsc --noEmit`)** as a dedicated CI type-check step, separate from the build

## Optional
- Rector-equivalent: `ts-morph`-based codemods for larger refactors
- `madge` for visualizing/verifying no circular dependencies
- Stryker for mutation testing

## Commands

```bash
npm run lint          # eslint .
npm run format        # prettier --write .
npm run typecheck     # tsc --noEmit
npm test              # jest / vitest
```

## CI Requirements

Pull requests should fail when:
- ESLint reports errors
- Prettier finds unformatted files
- `tsc --noEmit` reports type errors
- Tests fail
- `madge --circular` (or equivalent) finds a circular dependency

---

# Security

Always:
- Authorize actions via Policies at the boundary
- Validate all input via schemas (Zod/Joi)
- Escape/sanitize any output rendered as HTML (if server-rendering views)
- Hash passwords with a slow, salted algorithm (bcrypt/argon2) — never a fast general-purpose hash
- Never trust client input, including headers and query params
- Set standard security headers via `helmet`
- Apply rate limiting on authentication and other sensitive endpoints
- Keep secrets out of source control; load them only through the validated config layer

---

# File Uploads

Always validate:
- MIME type (based on file content/magic bytes, not just the client-supplied `Content-Type` header or extension)
- Size
- Dimensions (for images, when relevant)

Store uploads via the `infrastructure/storage` wrapper (S3/GCS), never directly on the app server's local disk in production.

Never trust the original filename — generate a new storage key/UUID and keep the original name only as metadata.

---

# Performance & Query Standards

See "Database Queries" for the existence-check rule.

Avoid unbounded reads:

```ts
await prisma.user.findMany(); // no limit, on a large table
```

Use:
- `paginate` (cursor-based for large/live tables)
- Batch processing (`findMany` with `take`/`cursor` loops) for large migrations/backfills
- Streaming for very large exports

Always review new queries for N+1 issues before merging (`include`/`with`/`populate`, or a DataLoader).

## General Performance Rules

Always:
- Eager-load relations needed for the response
- Paginate large datasets
- Batch large background jobs
- Queue heavy or slow work
- Measure before optimizing — do not guess at a performance fix without a profile or benchmark

---

# Migrations

Each migration should do one thing.

Always:
- Add indexes for columns used in `WHERE`/`JOIN`/sort
- Add foreign keys with an explicit `onDelete` behavior
- Make columns nullable only when necessary
- Provide a working rollback (a real `down` migration, or the ORM's equivalent) that reverses the `up` — do not leave it a no-op unless the change is genuinely irreversible, in which case comment why

Never modify a migration that has already been deployed. Create a new migration instead.

---

# Routes

Keep route files clean — a route file should read as a table of contents, not contain logic.

### Good

```ts
router.get('/orders', controller.index);
router.get('/orders/:id', controller.show);
router.post('/orders', validate(createOrderSchema), controller.store);
```

Group routes by domain (each domain owns its own `routes/` folder, mounted once in `app.ts`). Apply middleware (auth, validation, rate limiting) at the router level rather than repeating it per handler.

---

# Middleware

Middleware should be small, single-purpose, and side-effect-focused (auth check, request logging, validation, rate limiting). Middleware is one of the allowed places to read request/session context (see "Framework Independence"), but it should not itself contain business decision logic beyond "is this request allowed to proceed" — a business rule about *whether* something is allowed still belongs in a Policy, called from the middleware.

```ts
export function authorize(policy: Policy, action: string): RequestHandler {
  return (req, res, next) => {
    if (!policy[action](req.user, req.resource)) {
      return next(new UnauthorizedError());
    }
    next();
  };
}
```

---

# Console / CLI Scripts

CLI scripts (seed scripts, one-off maintenance scripts, `ts-node` scripts) are application entry points, parallel to Controllers.

Scripts should:
- Parse arguments/options
- Call Services or Actions
- Print output

Scripts should not contain business logic — a script that validates, queries, calculates, and mutates inline is not reusable from anywhere except that script, and is difficult to test without invoking it directly.

---

# Notifications

Notification classes/functions (email, SMS, push, Slack) should only format and deliver messages.

Determine *who* should be notified and *under what conditions* in the calling Service or Action, before dispatching. A Notification should not query the database to decide whether it's relevant, or branch on business state to decide what to say — it receives exactly what it needs to render the message.

---

# Code Style

Always:
- Enable `strict` mode
- Use typed function parameters and return types on exported functions
- Use constructor property shorthand (`constructor(private readonly x: X) {}`)
- Use `readonly` properties where appropriate
- Use explicit nullable types (`string | null`) rather than relying on implicit `undefined`

### Example

```ts
calculate(user: User): Invoice { ... }
```

---

# Comments & Documentation

Code should be self-explanatory. Avoid obvious comments.

### Bad
```ts
// increment counter
count++;
```

Comment only to explain *why*, a business rule, or a non-obvious algorithm.

Document:
- Complex algorithms
- Public API contracts (exported interfaces other domains/consumers rely on)
- Reusable Services

Do not document obvious code. Do not leave commented-out code or stray `TODO`s in merged code — file a ticket instead.

---

# Logging

Never leave `console.log` in committed code. Use a structured logger (Pino or Winston).

```ts
logger.info('User created', { userId: user.id, email: user.email });
```

Avoid unstructured logs:

```ts
logger.info(user); // no context, not queryable
```

Supported levels: `debug`, `info`, `warn`, `error`. Attach a request ID to every log line within a request's lifecycle so logs can be correlated across a single request.

---

# AI Assistant Rules & Guidelines

When generating or modifying Express/TypeScript code:

1. Follow SOLID principles.
2. Never place business logic in Controllers, routes, or middleware beyond auth/validation/logging.
3. Prefer Services/Actions over fat Controllers or fat Models.
4. Validate every input at the boundary with a Zod/Joi schema — never `req.body` unchecked.
5. Use Resources/serializers for API responses — never return an ORM entity directly.
6. Prevent N+1 queries; eager-load relations.
7. Use constructor-based dependency injection.
8. Use transactions for multi-step writes.
9. Prefer enums/literal unions over magic strings.
10. Never read `process.env` outside the config layer.
11. Never leave `console.log`/debug statements in committed code.
12. Write clean, strictly-typed, production-quality code — no implicit `any`.
13. Refactor duplicated logic instead of copying code.
14. Follow the existing domain-based architecture and conventions of the project.
15. Reuse existing classes/modules before creating new ones.
16. Do not introduce new patterns (a new state library, a new validation library, a new DI container) unless justified.
17. Prefer consistency with the existing codebase over personal preference.
18. Keep changes minimal and focused on the requested task.
19. Preserve backward compatibility unless instructed otherwise.
20. Refactor when complexity becomes excessive — but don't refactor unrelated code while completing an unrelated task.
21. Consider performance, scalability, and maintainability before introducing new code.
22. **Prefer refactoring over rewriting** when modifying existing code.
23. **Preserve existing public APIs** (exported functions, route contracts) unless instructed otherwise.
24. **Do not rename files, classes, or exported symbols** unnecessarily.
25. **Match the existing project's coding style**, even where it differs slightly from this document, unless asked to correct it.
26. **Avoid introducing additional dependencies** without justification.
27. **Services should prefer reusing an existing Action for data access** over re-querying inline — a consistency preference, not a rigid ban (see "Services and Data Access").
28. **Never mass-assign request bodies directly into a `create`/`update` call** — map fields explicitly.
29. **Do not introduce an Actions layer, Repository, DI container, or new domain subfolder merely because this document mentions it** — introduce it when the stated justification (reuse, multiple data sources, complex queries, growing responsibility) actually applies. Unjustified abstraction is itself a violation of these standards.
30. **Never generate code that violates this document merely because it is shorter or faster to produce.** Do not skip schema validation, Resources, Policies, or DTOs to save a few lines.
31. **When modifying existing code, preserve existing public behavior unless the requested change explicitly requires altering it.** Refactoring must not introduce behavioral changes — moving code between layers, extracting functions, or renaming internals should produce identical outputs for identical inputs.
32. **Never introduce `any` to make a type error disappear** — fix the underlying type, narrow with a type guard, or as a last resort use `unknown` with explicit validation.
33. **Never create a circular import between domains** to satisfy a quick fix — route the dependency through `shared/` or a domain event instead.

---

# Pull Request Checklist

Before submitting code, verify:

- [ ] ESLint and Prettier pass with no errors
- [ ] `tsc --noEmit` passes with no type errors, no new `any`
- [ ] No duplicated code
- [ ] No business logic in Controllers, routes, middleware, migrations, or seed scripts
- [ ] Services reuse existing Actions for data access where one already exists; direct ORM queries only used where no equivalent Action exists
- [ ] Authorization performed via Policies at the Controller/middleware boundary — not inside Services or Actions
- [ ] Input validated via a Zod/Joi schema at the boundary
- [ ] Uses a Resource/serializer for every API response
- [ ] No N+1 queries
- [ ] Transactions used where required, and not opened at more than one layer
- [ ] Structured logging in place of `console.log`
- [ ] Tests updated or added where applicable
- [ ] No unused imports or dead code
- [ ] No new circular imports (`madge --circular` clean)
- [ ] Clear variable, function, and file names following this document's conventions
- [ ] Documentation updated if behavior/architecture changed
- [ ] No commented-out code
- [ ] No leftover `TODO`
- [ ] No debug logging
- [ ] New configuration added to the validated config schema and documented
- [ ] Migration rollback (`down`) verified
- [ ] No breaking database changes without a migration strategy (backfilling, dual-write period, or a documented rollout plan for column drops/renames)