# Laravel Coding Standards

This document defines the coding standards for all Laravel projects. Every developer and AI coding assistant (Opencode, Claude Code, Antigravity, ChatGPT, Copilot, Cursor, etc.) must follow these standards consistently.

---

# Architecture Principles

To build scalable, robust, and clean applications, adhere to these fundamental principles:

- **High cohesion:** Group related logic together.
- **Low coupling:** Keep subsystems independent to minimize side-effects.
- **Dependency inversion:** Depend on abstractions, not on concrete implementations.
- **Single responsibility:** A class or method should have one, and only one, reason to change.
- **Explicit dependencies:** Inject dependencies clearly rather than resolving them implicitly.
- **Favor composition over inheritance:** Combine smaller components rather than building deep class hierarchies.

---

# Core Principles

- Follow SOLID principles.
- Follow PSR-12/PER coding standards.
- Prefer readability over clever code.
- Keep methods small and focused.
- Avoid duplicated logic (DRY).
- Never write code that "just works"; write code that is maintainable.
- Every piece of business logic should have a single responsible location.

---

# Control Flow

Prefer early returns over deeply nested conditionals. Keep nesting levels as shallow as possible, aiming to avoid more than 2-3 levels of nesting.

### Bad

```php
if ($user) {
    if ($user->active) {
        // ...
    }
}
```

### Good

```php
if (! $user) {
    return;
}

if (! $user->active) {
    return;
}
```

---

# Architecture Evolution

Choose the simplest architecture that satisfies current requirements. Do not introduce complex structures prematurely.

## Small Projects

Use the standard flat Laravel directory structure.

```
app/
    Http/
    Models/
    Services/
    ...
```

## Medium Projects

Introduce Actions, DTOs, and Enums to keep business logic organized.

```
app/
    Actions/
    DTOs/
    Enums/
    Events/
    Exceptions/
    Helpers/
    Http/
    Jobs/
    Listeners/
    Mail/
    Models/
    Notifications/
    Observers/
    Policies/
    Providers/
    Repositories/
    Rules/
    Services/
    Traits/
    ValueObjects/
```

## Large Projects

Consider domain-based organization to prevent coupling and manage scale.

```
app/
    Domains/
        User/
        Billing/
        Inventory/
        Reporting/
```

Each domain folder owns its own domain-specific layers:
- Actions
- DTOs
- Events
- Jobs
- Models
- Policies
- Services

---

# Dependency Rules

To maintain a clean codebase, adhere strictly to the following directional flow of dependencies.

## Allowed:

```
Controller
    ↓
Service
    ↓
Action
    ↓
Repository (optional layer — see "Repositories")
    ↓
Model
```

**Notes on this chain:**

- The **Repository** layer is optional (see the Repositories section below). When a Repository does not exist for a given Model, Actions (or Services, in small projects — see below) may query the Model/`Model::query()` directly.
- **Whether Services may query Models directly depends on project size** — see "Services and Data Access" below.
- Controllers may only call Services or Actions — never Repositories or Models directly.

## Disallowed:

- Controllers calling repositories or models directly
- Models calling services
- Models dispatching business workflows
- Services depending on controllers
- Circular dependencies between classes
- Mixing patterns within the same entity — e.g. some Services querying `User` directly while others go through `UserAction` classes for the same model. Pick one approach per entity and stay consistent (see "Services and Data Access").

## Services and Data Access

This rule scales with the "Architecture Evolution" tier the project is in:

- **Small projects** (flat structure, no Actions layer): Services may query Models directly. Keep it tight — if a single query is growing into 15+ lines of logic inline in a Service method, that's a signal the project has outgrown this tier; consider introducing an Actions layer for that slice of the app rather than letting the Service sprawl.

- **Medium/large projects** (Actions layer present): once an Actions layer exists, Services must delegate data access to Actions rather than querying Models/Repositories directly. A Service's job is to orchestrate Actions (and other Services), not to touch persistence itself. If a Service needs a query no existing Action provides, add a new Action rather than reaching into the Model from the Service.

Do not mix both approaches for the same entity within one project — if `OrderAction` classes exist, `OrderService` should use them instead of querying `Order` directly, even if it would be more convenient for one specific method.

---

# Framework Independence

Business logic must not depend on the HTTP request lifecycle. Any authenticated user, tenant, locale, timezone, or other request-specific context must be passed explicitly into Services and Actions as method arguments or DTOs, rather than retrieved via Laravel helper functions or facades.

This covers not only `auth()`, but also other request concerns such as:
- Current tenant or company
- Current locale or timezone
- Impersonation context
- Authenticated user
- Request IP or user agent

## Prohibited in Services, Actions, Repositories, Models, Domains, and Value Objects:
- `auth()` and `Auth` facade
- `request()` and `Request` facade
- `session()`
- `cookie()`
- `redirect()`
- `response()`
- `abort()`
- `view()`
- `back()`
- `old()`
- `url()`

These are HTTP/UI concerns.

## Allowed Context Retrieval Exceptions:
You may retrieve HTTP/UI/session context only within specific Laravel infrastructure classes:
- Middleware
- Controllers
- Form Requests
- Policies
- Gates
- Blade Components
- Notifications (sometimes, when building context-dependent messages)
- Listeners (sometimes, when acting as UI/HTTP boundaries)

## Context Ingestion Principle:
Controllers (and other entry-points like console commands or jobs) are responsible for extracting framework-specific data. Services and Actions should receive everything they need through:
- Method parameters
- DTOs
- Constructor injection

They should never retrieve runtime context themselves.

### Bad

```php
class CreateOrderAction
{
    public function handle(array $data)
    {
        $user = auth()->user(); // Bad: Hidden HTTP dependency
        // ...
    }
}
```

### Good

```php
class CreateOrderAction
{
    public function handle(User $user, array $data)
    {
        // ...
    }
}
```

In the Controller:

```php
public function store(StoreOrderRequest $request)
{
    $this->createOrderAction->handle(
        $request->user(),
        $request->validated()
    );
}
```

---

# Actions vs Services

## Actions

Actions represent a single business operation or use case, including data access for that operation.

### Characteristics:
- One responsibility
- One public `handle()` method
- Easily testable
- Reusable
- May query the Model or Repository directly

### Examples:
- `CreateUserAction`
- `ApproveOrderAction`
- `AssignRoleAction`
- `UploadDocumentAction`

### Example:

```php
class CreateUserAction
{
    public function handle(CreateUserData $data): User
    {
        return User::create([...]);
    }
}
```

## Services

Services coordinate multiple actions and orchestrate business workflows. They do not perform data access themselves.

### Characteristics:
- Multiple methods allowed
- Orchestrates business processes
- Coordinates actions, repositories (via actions), jobs, and events
- In projects with an Actions layer: delegates data access to Actions rather than calling `Model::` directly (see "Services and Data Access")
- In small projects with no Actions layer: may query Models directly, kept minimal

### Example:

```php
class UserOnboardingService
{
    public function onboard(CreateUserData $data): User
    {
        $user = $this->createUserAction->handle($data);

        $this->assignDefaultRoleAction->handle($user);

        $this->sendWelcomeEmailAction->handle($user);

        return $user;
    }
}
```

## Choosing Between an Action and a Service

Use an **Action** when:
- The operation performs one business task.
- The operation can be reused independently.
- The class naturally exposes a single `handle()` method.

Use a **Service** when:
- Multiple Actions must be coordinated.
- The workflow spans several business operations.
- The process includes transactions, events, notifications, or external integrations.

### Rule:
- Business logic and data access belong in **Actions**.
- Workflow orchestration belongs in **Services**.

---

# Controllers

Controllers should only:

- Validate request
- Call service/action
- Return response

### Bad

```php
public function store(Request $request)
{
    $user = User::create([...]);

    Mail::to($user)->send(...);

    return response()->json($user);
}
```

### Good

```php
public function store(StoreUserRequest $request)
{
    $user = $this->userService->create($request->validated());

    return UserResource::make($user);
}
```

Controllers should not:

- Build queries
- Perform calculations
- Send emails
- Dispatch business logic
- Contain loops with business rules

---

# Services

Services contain orchestration workflows, not raw data access (see "Actions vs Services").

Each service should have a single responsibility.

### Good

```
UserService
InvoiceService
PaymentService
ForecastCalculationService
```

## Service Size

Keep services under **300 lines**. Treat 300 lines as the point at which you actively review the class for split-worthy responsibilities — don't wait until it's unmanageable. This is the same limit referenced in "Class Size" below; services are not a special case.

---

# Models

Models should represent data.

Allowed:

- Relationships
- Scopes
- Accessors
- Mutators
- Small helper methods

Avoid:

- Huge business logic
- API calls
- Complex calculations

## Mass Assignment

Always define `$fillable` explicitly on every Model. Do not use `$guarded = []`.

### Bad

```php
class User extends Model
{
    protected $guarded = [];
}
```

### Good

```php
class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];
}
```

Never mass-assign fields that control authorization, ownership, or financial state (e.g. `role`, `is_admin`, `balance`) through user-supplied input. Set these explicitly in the Action, not via `create($request->all())`.

---

# Validation

Always use Form Requests.

Never validate directly inside controllers.

### Good

```
StoreInvoiceRequest
UpdateCustomerRequest
```

---

# Database Queries

Never execute queries inside loops.

### Bad

```php
foreach ($users as $user) {
    $user->posts;
}
```

### Good

```php
User::with('posts')->get();
```

Always prevent N+1 queries.

Always eager load relationships.

---

# Collection Standards

Prefer Collection methods where readability improves.

### Good

```php
collect($users)
    ->filter(...)
    ->map(...)
    ->groupBy(...);
```

Do not chain collections excessively. If the chain becomes difficult to read, refactor into variables or dedicated methods.

---

# Eloquent

Prefer Eloquent relationships.

Avoid raw SQL unless performance requires it.

Prefer:

```php
User::query()
```

instead of

```php
DB::table(...)
```

unless necessary.

---

# Constructors

Constructors should only assign dependencies.

Avoid:
- Business logic
- Database queries
- API calls
- File operations

Constructors must be lightweight and completely side-effect free.

---

# Dependency Injection

Always use constructor injection.

### Bad

```php
new UserService();
```

### Good

```php
public function __construct(
    private UserService $service
) {}
```

Never resolve classes manually unless absolutely necessary.

Avoid

```php
app(UserService::class)
```

---

# Service Container

Bind interfaces to implementations.

### Good

```
UserRepositoryInterface
UserRepository
```

Register bindings in Service Providers.

---

# Repositories

Do not introduce repositories by default. Repositories are justified only when they provide meaningful abstraction, such as:

- Multiple persistence implementations
- External data sources
- Reusable complex queries
- Caching layers

Simple CRUD applications generally do not benefit from repositories.

**When a Repository does not exist for a given Model, Actions query the Model directly** (`Model::query()`), and the dependency chain effectively becomes `Controller → Service → Action → Model`. Do not introduce a Repository purely to satisfy the layering diagram — only add one when one of the justifications above applies. If a Repository is later introduced for a Model, all Actions touching that Model should be migrated to use it, rather than mixing direct Model access and Repository access for the same entity.

---

# Request Validation

Always create dedicated FormRequest classes.

Never use:

```php
$request->validate(...)
```

inside controllers.

---

# API Responses

Always use API Resources.

Never return Eloquent models directly.

### Good

```
UserResource
InvoiceResource
```

---

# API Standards

- Use API Resources.
- Return consistent response structures.
- Use proper HTTP status codes.
- Paginate collections.
- Avoid exposing internal attributes.
- Version public APIs when necessary.

## API Versioning

Version any API consumed by external clients or a separately-deployed frontend/mobile app. Use URL-based versioning (`/api/v1/...`) as the default approach; only deviate (header-based versioning, etc.) with explicit justification. Group versioned routes and controllers under a `V1`/`V2` namespace so breaking changes to `V2` don't touch `V1` code.

Internal-only APIs consumed exclusively by a monolith's own frontend do not require versioning.

---

# DTOs

Use DTOs for large service inputs.

### Example

```
CreateInvoiceData
CreateAppointmentData
UpdateForecastData
```

Avoid passing huge arrays.

---

# Enums

Use PHP Enums instead of magic strings.

### Bad

```php
$status = "pending";
```

### Good

```php
Status::Pending
```

---

# Constants

Never hardcode values.

### Bad

```php
if ($role == "admin")
```

### Good

```php
Role::ADMIN
```

---

# Naming

Use descriptive names.

### Bad

```
$data
$temp
$item
$x
```

### Good

```
$customer
$forecast
$invoiceItem
$totalRevenue
```

## Class Naming Conventions

Always align class names with their roles:

- **Actions:** `CreateUserAction`, `UpdateOrderAction`
- **Services:** `UserService`, `PaymentService`
- **Requests:** `StoreUserRequest`, `UpdateUserRequest`
- **Resources:** `UserResource`, `OrderResource`
- **Policies:** `UserPolicy`, `OrderPolicy`
- **DTOs:** `CreateUserData`, `UpdateOrderData`
- **Enums:** `UserStatus`, `OrderStatus`

## Boolean Methods

Use descriptive prefixes for boolean methods to return clearly identifiable true/false states.

### Prefer:
- `isActive()`
- `hasPermission()`
- `canEdit()`
- `shouldSync()`

### Avoid:
- `check()`
- `verify()`
- `flag()`

---

# Method Size

Aim for:

- 10–30 lines

If a method exceeds ~50 lines, consider refactoring.

---

# Class Size

Prefer:

- under 300 lines — treat this as the trigger to review the class for split-worthy responsibilities.

Refactor mandatorily:

- classes larger than 500 lines must be split before merging, except where a maintainer explicitly signs off (e.g. large generated/DTO classes).

This threshold applies uniformly to Services, Actions, Repositories, and Models — there is no separate limit per class type.

---

# PHP Standards

Always:
- declare strict types
- use final classes when extension is unnecessary
- use readonly where applicable
- prefer enums over constants
- use typed properties
- declare return types

---

# Comments

Code should be self-explanatory. Avoid obvious code documentation.

### Bad:

```php
// increment counter
$count++;
```

Comment only when explaining:

- why
- business rules
- complex algorithms

---

# Documentation

Document:
- Complex algorithms
- Public APIs
- Reusable services

Do not document obvious code.

---

# Logging

Never use `dd()`, `dump()`, `var_dump()`, or `print_r()` in committed code.

Use structured logging:

```php
Log::info('User created', [
    'user_id' => $user->id,
    'email' => $user->email,
]);
```

Avoid unstructured logs like:

```php
Log::info($user);
```

Supported levels:

```php
Log::info();
Log::warning();
Log::error();
```

---

# Error Handling

Never swallow exceptions.

### Bad

```php
catch (\Exception $e) {

}
```

### Good

```php
catch (\Throwable $e) {
    Log::error($e);

    throw $e;
}
```

Use custom exceptions where appropriate.

---

# Transactions

Wrap multi-step database operations.

```php
DB::transaction(function () {

});
```

---

# Queues

Queue work that involves any of the following:

- external API calls (payment gateways, third-party sync)
- sending emails or notifications
- imports/exports
- calculations expected to take longer than ~1 second synchronously
- any operation that would otherwise block the HTTP response for the user

Do not queue trivial in-memory operations or single fast (< ~100ms) DB writes — the overhead of dispatching and processing a job exceeds the cost of doing it inline. When in doubt, measure before deciding.

---

# Events

Use events when multiple listeners need to react.

Avoid events for simple sequential code.

---

# Configuration

Never hardcode configuration.

Use:

```php
config(...)
```

Never:

```php
env(...)
```

outside configuration files.

Never hardcode:
- URLs
- API endpoints
- Feature flags
- Timeouts
- Retry counts
- Cache durations

Place them in configuration files.

---

# Caching

Cache:

- expensive queries
- dashboard statistics
- configuration
- external API results

Use cache tags where supported.

---

# Testing

Prefer:

- Feature tests
- Unit tests

Every new business feature should include tests where practical.

## Testing Conventions

- **Feature tests** exercise the full stack (Controller → Service → Action → Model → DB) through HTTP or console entry points. Use these to verify authorization, validation, and response shape. Use model factories to set up state — do not hand-craft raw DB inserts.
- **Unit tests** target a single Action or Service in isolation. Mock collaborators (other Actions, Services, external clients) using interfaces/constructor injection so the class under test has no hidden dependencies to fake.
- Do not mock the Model/ORM layer itself in Unit tests — use an in-memory/test database (SQLite `:memory:` or a dedicated test DB) instead. Mocking Eloquent tends to produce tests that pass against a fake and fail against the real schema.
- Policies and Gates should have dedicated authorization tests (allowed/denied cases), separate from Feature tests of the underlying endpoint.
- Avoid testing framework internals (e.g. that Laravel's validator rejects a missing required field) — test your own business rules and edge cases.

---

# Code Quality Tooling

Every project should include automated code quality tools.

## Required

- **Laravel Pint** (Code formatting)
- **Larastan (PHPStan)** (Static analysis)
- **PHPUnit or Pest** (Testing frameworks)

## Optional

- Rector
- Laravel IDE Helper
- Infection PHP

## Commands

Format code:
```bash
./vendor/bin/pint
```

Static analysis:
```bash
./vendor/bin/phpstan analyse
```

Run tests:
```bash
php artisan test
```

## CI Requirements

Pull requests should fail when:
- Pint fails to format code
- PHPStan reports compilation or type errors
- Tests fail

---

# Security

Always:

- authorize actions
- validate input
- escape output
- use Policies/Gates
- hash passwords
- never trust client input
- define `$fillable` explicitly on every Model (see "Models → Mass Assignment")

---

# File Uploads

Always validate:

- MIME type
- size
- dimensions (images)

Store using Storage.

Never trust original filenames.

---

# Performance & Query Standards

## Query Performance Standards

Prefer:

```php
User::query()->exists();
```

instead of:

```php
User::query()->count() > 0;
```

Avoid:

```php
Model::all();
```

for large datasets.

Use:

- `paginate()`
- `chunk()`
- `chunkById()`
- `cursor()`
- `lazy()`

when processing large amounts of data.

Always review queries for N+1 issues.

## General Performance Rules

Always:

- eager load
- paginate large datasets
- chunk large jobs
- lazy collections when appropriate
- queue heavy work

Measure performance before optimizing.

---

# Migrations

Each migration should do one thing.

Always:

- add indexes
- add foreign keys
- make columns nullable only when necessary
- implement a working `down()` method that reverses the `up()` — do not leave it empty unless the change is genuinely irreversible (in which case, comment why)

Never modify old migrations after deployment.

Create new migrations instead.

---

# Routes

Keep routes clean.

### Good

```php
Route::apiResource('users', UserController::class);
```

Group routes logically.

Use middleware appropriately.

---

# Blade

Keep business logic out of Blade templates.

Avoid nested conditionals and database queries in views.

---

# Code Style

Always:

- declare strict types where feasible
- use typed properties
- use return types
- use constructor property promotion
- use readonly properties when appropriate
- use nullable types explicitly

### Example

```php
public function calculate(User $user): Invoice
```

---

# AI Assistant Rules & Guidelines

When generating or modifying Laravel code:

1. Follow SOLID principles.
2. Never place business logic in controllers.
3. Prefer Services/Actions over fat Models.
4. Use Form Requests for validation.
5. Use Resources for API responses.
6. Prevent N+1 queries.
7. Use dependency injection.
8. Use transactions for multi-step writes.
9. Prefer Enums over magic strings.
10. Never use `env()` outside config files.
11. Never leave debug statements (`dd`, `dump`, etc.).
12. Write clean, readable, production-quality code.
13. Refactor duplicated logic instead of copying code.
14. Follow the existing architecture and conventions.
15. Reuse existing classes before creating new ones.
16. Do not introduce new patterns unless justified.
17. Prefer consistency over personal preference.
18. Keep changes minimal and focused.
19. Preserve backward compatibility unless instructed otherwise.
20. Refactor when complexity becomes excessive.
21. Consider performance, scalability, and maintainability before introducing new code.
22. **Prefer refactoring over rewriting** when modifying existing code.
23. **Preserve existing public APIs** unless instructed otherwise.
24. **Do not rename files or classes** unnecessarily.
25. **Match the existing project's coding style**.
26. **Avoid introducing additional dependencies** without justification.
27. **In projects with an Actions layer, Services must not query Models/Repositories directly** — delegate data access to an Action. In small projects with no Actions layer, Services may query Models directly. Never mix both approaches for the same entity.
28. **Define `$fillable` explicitly on every Model** — never use `$guarded = []`.

---

# Pull Request Checklist

Before submitting code, verify:

- [ ] PSR-12/PER compliant
- [ ] No duplicated code
- [ ] No business logic in controllers
- [ ] No data access in Services where an Actions layer exists (delegated to Actions); direct Model access only used in small projects without one
- [ ] Validation via Form Requests
- [ ] Uses Resources for API responses
- [ ] No N+1 queries
- [ ] Proper authorization implemented
- [ ] Transactions where required
- [ ] Logging instead of debug statements
- [ ] Tests updated or added where applicable
- [ ] No unused imports or dead code
- [ ] Clear variable and method names
- [ ] Documentation updated if behavior/architecture changed
- [ ] No commented-out code
- [ ] No TODO left behind
- [ ] No debug logging
- [ ] New configuration documented
- [ ] Model `$fillable` reviewed for new/changed columns
- [ ] Migration `down()` method verified