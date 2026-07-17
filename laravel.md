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
Repository
    ↓
Model
```

## Disallowed:

- Controllers calling repositories directly
- Models calling services
- Models dispatching business workflows
- Services depending on controllers
- Circular dependencies between classes

---

# Actions vs Services

## Actions

Actions represent a single business operation or use case.

### Characteristics:
- One responsibility
- One public `handle()` method
- Easily testable
- Reusable

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

Services coordinate multiple actions and orchestrate business workflows.

### Characteristics:
- Multiple methods allowed
- Orchestrates business processes
- Coordinates actions, repositories, jobs, and events

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
- Business logic belongs in **Actions**.
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

Services contain business logic and orchestration workflows.

Each service should have a single responsibility.

### Good

```
UserService
InvoiceService
PaymentService
ForecastCalculationService
```

Large services often indicate multiple responsibilities. Review any service exceeding ~300 lines. Refactor before it becomes difficult to understand.

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

- under 300 lines

Review classes larger than 500 lines.

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

Queue:

- emails
- notifications
- imports
- exports
- large calculations
- external API synchronization

Never queue tiny operations.

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

---

# Pull Request Checklist

Before submitting code, verify:

- [ ] PSR-12/PER compliant
- [ ] No duplicated code
- [ ] No business logic in controllers
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