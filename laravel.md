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

# Business Rules

Business rules belong in:

- Actions
- Services
- Value Objects
- Policies

Business rules must never live inside:

- Controllers
- Blade templates
- Routes
- JavaScript/frontend code
- Migrations
- Factories/Seeders

If a decision, calculation, or validation rule reflects how the business actually operates — pricing, eligibility, status transitions, permissions, limits — it belongs in one of the four locations above, expressed as a named, testable method. A rule duplicated across a Controller and a Console Command is a rule that was put in the wrong place the first time.

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
Controller (HTTP/CLI Request Boundary)
    │
    ▼ [Rule: Controllers must NEVER bypass the Application Layer to touch Persistence directly]
Application Layer (Service, Action, or Service delegating to Action)
    │
    ▼
Persistence Layer (Repository if one exists, otherwise Model directly)
```

The Application Layer is not required to be both a Service *and* an Action. Depending on the project and the specific operation, it may be:

- a **Service** alone (small projects, or simple orchestration with no reusable sub-steps),
- an **Action** alone (a single Controller calling a single business operation directly), or
- a **Service that delegates to Actions** (medium/large projects coordinating multiple reusable operations).

**Notes on this chain:**

- The **Repository** layer is optional (see the Repositories section below). When a Repository does not exist for a given Model, the Application Layer may query the Model/`Model::query()` directly.
- Controllers may only call the Application Layer (Services/Actions) — never Repositories or Models directly.

## Disallowed:

- Controllers calling repositories or models directly
- Models calling services
- Models dispatching business workflows
- Services depending on controllers
- Circular dependencies between classes

## Services and Data Access

Services should avoid data access when an equivalent Action already exists for that operation — reuse it rather than re-querying inline. But this is a preference for consistency, not a rigid layering rule: a `DashboardService` doing `User::count()` and `Order::count()` inline is fine; creating `CountUsersAction`/`CountOrdersAction` purely to satisfy a layering diagram adds ceremony without value.

The judgment call: if an Action for that query already exists, use it. If it doesn't, and creating one wouldn't be reused elsewhere, querying the Model directly from the Service is acceptable. Prefer consistency within a given entity's codepath over strict adherence to the "ideal" layering.

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

## Value Objects Must Be Pure

Value Objects have a stricter bar than Services/Actions: they must not perform I/O of any kind. In addition to the HTTP/session facades listed above, Value Objects must not use:
- `config()`
- `Cache::`
- `Storage::`
- `DB::`

A Value Object should be fully constructible and usable from its constructor arguments alone, with no side effects and no reads from external state. If a value needs configuration or persisted state to compute, that computation belongs in a Service or Action, which then passes the result into the Value Object.

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

Services coordinate multiple actions and orchestrate business workflows. They should prefer delegating data access to Actions where a suitable one exists, but may query Models directly for simple, non-reusable operations (see "Services and Data Access").

### Characteristics:
- Multiple methods allowed
- Orchestrates business processes
- Coordinates actions, repositories (via actions), jobs, and events
- Prefers reusing an existing Action for data access over querying the Model inline, but may query the Model directly for simple cases with no reuse benefit

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

Not every project or every operation needs an Actions layer. Introduce Actions when a business operation becomes independently reusable — not simply because "that's the standard." A guideline that treats Actions as mandatory tends to produce noise like `GetUserAction`, `FindUserAction`, `DeleteUserAction`, `ArchiveUserAction` for every trivial CRUD verb, none of which are reused anywhere. That's ceremony, not architecture.

Use an **Action** when:
- The operation performs one business task.
- The operation is, or is likely to be, reused independently (called from more than one Controller, Service, Job, or Console Command).
- The class naturally exposes a single `handle()` method.

Use a **Service** when:
- Multiple Actions must be coordinated.
- The workflow spans several business operations.
- The process includes transactions, events, notifications, or external integrations.

Use a **Service with no Actions** when:
- The logic is simple orchestration or a single query/mutation with no independently reusable sub-steps. Don't manufacture an Action just to have one.

### Rule:
- Business logic and data access belong in **Actions** or **Services**, depending on reuse — see above.
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

# Authorization

Authorization belongs in Policies and Gates, invoked at the Controller/Form Request boundary (e.g. `$this->authorize(...)`, `Gate::authorize(...)` in a Controller, or a Form Request's `authorize()` method).

Services and Actions assume the caller is already authorized. Do not call `Gate::allows()`, `Gate::authorize()`, `$user->can(...)`, or a Policy directly from inside a Service or Action — that re-introduces an HTTP/session concern into business logic and duplicates a check that should have already happened at the boundary.

If a Service or Action needs to make an authorization-adjacent business decision (e.g. "can this order still be cancelled given its current state"), express that as a domain rule with a descriptive method (`$order->canBeCancelled()`), not as an authorization check.

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

A Service should have a single responsibility. Line count alone is not a reliable signal — a 320-line Service handling one cohesive workflow isn't automatically worse than four tiny Services bouncing calls between each other.

When a Service becomes difficult to understand, navigate, or test, consider extracting Actions or splitting into additional Services. Large classes are a symptom of mixed responsibilities, not the problem itself — review for *what* the class is doing, not how many lines it takes to do it.

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

# Soft Deletes & Auditing

## Soft Deletes

Use Soft Deletes only when business requirements dictate that deleted data must be restorable or kept for auditing.
- **Global Scopes:** Be aware of default global scopes (e.g., `withoutTrashed()`) when querying relations. Use `withTrashed()` explicitly when you need to load deleted models.
- **Unique Validation:** Standard unique rules will fail if a soft-deleted row exists with the same value. Customize validation rules to ignore/include trashed records appropriately.

## Global Scopes

- Avoid complex global scopes that implicitly alter query outcomes across the application, as they can cause bugs and complicate unit testing.
- Prefer explicit local query scopes (e.g., `scopeActive($query)`) that developers call consciously.

## Activity Logging & Auditing

- Audit critical data mutations, status changes, and user authorization updates.
- Use established packages (e.g., `spatie/laravel-activitylog`) rather than custom DB listeners.
- Always log full metadata: the actor (user), the target model, key changed fields (before/after), and contextual request metadata (like IP, session, or CLI context).

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

Repositories exist to abstract persistence.

Do not create repositories simply to wrap Eloquent CRUD.

Repositories should only exist when they provide meaningful abstraction, such as:

- multiple data sources
- reusable complex queries
- caching
- external systems

When no Repository exists for a Model, the Application Layer (Action or Service) queries the Model directly. If a Repository is later introduced for a Model, migrate all existing direct-Model-access code paths for that Model to use it, rather than mixing both approaches for the same entity.

---

# Request Validation

Always create dedicated FormRequest classes.

Never use:

```php
$request->validate(...)
```

inside controllers.

## Organisation of Form Requests

In larger projects, avoid a flat list of classes in `app/Http/Requests`. Organise requests logically:
- **By Resource/Feature:** Group them into subdirectories under `app/Http/Requests`, e.g.:
  - `app/Http/Requests/User/StoreUserRequest.php`
  - `app/Http/Requests/User/UpdateUserRequest.php`
- **In Domain-Driven Layouts:** Place requests directly inside their corresponding Domain folder, e.g.:
  - `app/Domains/User/Requests/StoreUserRequest.php`

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

## API Error Responses

Avoid returning inconsistent error structures. Use a standardized JSON structure for all API errors.

### Standard Format:
```json
{
    "message": "The given data was invalid.",
    "errors": {
        "email": [
            "The email field is required."
        ]
    }
}
```

Recommend customizing exception rendering in `bootstrap/app.php` (Laravel 11+) or the global Exception Handler to guarantee that all API exceptions (e.g. `ValidationException`, `ModelNotFoundException`, `AuthenticationException`) are converted to this standard response structure automatically.

---

# DTOs

Prefer DTOs whenever a method receives more than 3–4 related parameters, or the parameters together represent a single business concept.

### Bad

```php
public function create(
    string $name,
    string $email,
    string $phone,
    string $country,
    string $city
): User
```

### Good

```php
public function create(CreateUserData $data): User
```

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

For Actions, Repositories, and Models, line count is a useful early-warning signal:

- **under 300 lines** — treat this as the trigger to review the class for split-worthy responsibilities.
- **larger than 500 lines** — must be split before merging, except where a maintainer explicitly signs off (e.g. large generated/DTO classes).

For **Services**, follow the responsibility-based guidance in "Service Size" above instead of a fixed threshold — a large Service that stays cohesive around one workflow is not automatically a problem, but drifting responsibilities are.

In all cases: treat size as a symptom to investigate, not a rule to satisfy by mechanically splitting a class.

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

# Side Effects

Separate calculations from side effects whenever practical.

Pure methods — ones whose purpose is to compute or decide something — should not:

- send emails
- write logs
- dispatch jobs or events
- mutate unrelated models
- perform other I/O

### Bad

```php
public function calculateDiscount(Order $order): float
{
    $discount = $order->total * 0.1;

    Log::info('Discount calculated', ['order_id' => $order->id]); // side effect in a calculation

    $order->update(['discount_applied' => true]); // mutates unrelated state

    return $discount;
}
```

### Good

```php
public function calculateDiscount(Order $order): float
{
    return $order->total * 0.1;
}
```

The caller (an Action or Service orchestrating the workflow) is responsible for logging, dispatching, and persisting — not the calculation itself. This makes pure logic trivially unit-testable without mocks, and makes it obvious, when reading a method, whether calling it twice is safe.

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

## Domain Events

Prefer domain events over tightly coupling unrelated services to one another.

### Bad

```
OrderService
    ↓ (directly calls)
EmailService
    ↓ (directly calls)
SlackService
    ↓ (directly calls)
CRMService
```

### Good

```
OrderCreated event
    ↓
SendOrderConfirmationEmail listener
UpdateCrmRecord listener
NotifySlackChannel listener
```

The triggering Service (`OrderService`) only needs to know that an order was created and dispatch the event — it should not know or care who reacts to it, or how many listeners exist. This keeps unrelated concerns (email, CRM sync, Slack notifications) decoupled and independently testable, and lets new reactions be added without modifying `OrderService`.

Use direct calls (not events) when the caller genuinely needs the result of the operation synchronously, or when there is exactly one tightly-related consequence that will realistically never need to vary independently.

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

- **Database State Management:** Use the `RefreshDatabase` trait (for full DB reset on each run) or `DatabaseTransactions` trait (to roll back changes automatically) to ensure tests run in isolation and maintain a clean state.
- **Database Assertions:** In Feature tests, prefer calling `assertDatabaseHas` and `assertDatabaseMissing` to verify side effects of logic execution, rather than manually checking counts or records.
- **Mocking External Services:** Avoid actual network calls. Mock external integrations using native Laravel fakes (e.g. `Http::fake()`, `Queue::fake()`, `Mail::fake()`, `Event::fake()`, `Storage::fake()`) or bind mock class implementations to interfaces in the service container.
- **Feature tests** exercise the full stack (Controller → Service → Action → Model → DB) through HTTP or console entry points. Use these to verify authorization, validation, and response shape. Use model factories to set up state — do not hand-craft raw DB inserts.
- **Unit tests** target a single Action or Service in isolation. Mock collaborators (other Actions, Services, external clients) using interfaces/constructor injection so the class under test has no hidden dependencies to fake.
- **Do not mock the Model/ORM layer itself** in Unit tests — use an in-memory/test database (SQLite `:memory:` or a dedicated test DB) instead. Mocking Eloquent tends to produce tests that pass against a fake and fail against the real schema.
- **Policies and Gates** should have dedicated authorization tests (allowed/denied cases), separate from Feature tests of the underlying endpoint.
- **Avoid testing framework internals** (e.g. that Laravel's validator rejects a missing required field) — test your own business rules and edge cases.

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
27. **Services should prefer reusing an existing Action for data access over re-querying inline** — but this is a consistency preference, not a rigid ban; simple, non-reusable queries may live directly in a Service (see "Services and Data Access").
28. **Define `$fillable` explicitly on every Model** — never use `$guarded = []`.
29. **Do not introduce an Actions layer, Repository, or other abstraction merely because this document mentions it** — introduce it when the stated justification (reuse, multiple data sources, complex queries, caching, etc.) actually applies. Unjustified abstraction is itself a violation of these standards.
30. **Never generate code that violates this document merely because it is shorter or faster to produce.** Architecture and correctness are preferred over brevity — do not skip Form Requests, Resources, Policies, or DTOs to save a few lines.

---

# Pull Request Checklist

Before submitting code, verify:

- [ ] PSR-12/PER compliant
- [ ] No duplicated code
- [ ] No business logic in controllers, Blade, routes, migrations, or factories (see "Business Rules")
- [ ] Services reuse existing Actions for data access where one already exists; direct Model queries only used where no equivalent Action exists
- [ ] Authorization performed via Policies/Gates at the Controller/Form Request boundary — not inside Services or Actions
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