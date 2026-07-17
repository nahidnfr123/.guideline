# Laravel Coding Standards

This document defines the coding standards for all Laravel projects. Every developer and AI coding assistant (Claude Code, ChatGPT, Copilot, Cursor, etc.) must follow these standards consistently.

---

# Core Principles

- Follow SOLID principles.
- Follow PSR-12 coding standards.
- Prefer readability over clever code.
- Keep methods small and focused.
- Avoid duplicated logic (DRY).
- Prefer composition over inheritance.
- Never write code that "just works"; write code that is maintainable.
- Every piece of business logic should have a single responsible location.

---

# Project Structure

Business logic must never live inside controllers.

```
Controller
    ↓
Service
    ↓
Repository (if needed)
    ↓
Model
```

Use the following structure:

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

---

# Controllers

Controllers should only:

- Validate request
- Call service
- Return response

Bad

```php
public function store(Request $request)
{
    $user = User::create([...]);

    Mail::to($user)->send(...);

    return response()->json($user);
}
```

Good

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

Services contain business logic.

Each service should have a single responsibility.

Good

```
UserService
InvoiceService
PaymentService
ForecastCalculationService
```

Avoid massive services.

If a service exceeds roughly 400–500 lines, consider splitting it.

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

Good

```
StoreInvoiceRequest
UpdateCustomerRequest
```

---

# Database Queries

Never execute queries inside loops.

Bad

```php
foreach ($users as $user) {
    $user->posts;
}
```

Good

```php
User::with('posts')->get();
```

Always prevent N+1 queries.

Always eager load relationships.

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

# Dependency Injection

Always use constructor injection.

Bad

```php
new UserService();
```

Good

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

Good

```
UserRepositoryInterface
UserRepository
```

Register bindings in Service Providers.

---

# Repositories

Use repositories only when they provide value.

Do NOT create repositories for simple CRUD.

Use repositories when:

- multiple data sources
- reusable complex queries
- caching
- external APIs

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

Good

```
UserResource
InvoiceResource
```

---

# DTOs

Use DTOs for large service inputs.

Example

```
CreateInvoiceData
CreateAppointmentData
UpdateForecastData
```

Avoid passing huge arrays.

---

# Enums

Use PHP Enums instead of magic strings.

Bad

```php
$status = "pending";
```

Good

```php
Status::Pending
```

---

# Constants

Never hardcode values.

Bad

```php
if ($role == "admin")
```

Good

```php
Role::ADMIN
```

---

# Naming

Use descriptive names.

Bad

```
$data
$temp
$item
$x
```

Good

```
$customer
$forecast
$invoiceItem
$totalRevenue
```

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

# Comments

Code should be self-explanatory.

Avoid:

```php
// increment counter
$count++;
```

Comment only when explaining:

- why
- business rules
- complex algorithms

---

# Logging

Never use

```php
dd()
dump()
var_dump()
print_r()
```

in committed code.

Use

```php
Log::info();
Log::warning();
Log::error();
```

---

# Error Handling

Never swallow exceptions.

Bad

```php
catch (\Exception $e) {

}
```

Good

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

Use

```php
config(...)
```

Never

```php
env(...)
```

outside configuration files.

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

# Performance

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

Good

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

Example

```php
public function calculate(User $user): Invoice
```

---

# AI Assistant Rules

When generating or modifying Laravel code:

1. Follow SOLID principles.
2. Never place business logic in controllers.
3. Prefer Services over fat Models.
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
14. Preserve existing architecture unless an improvement is justified.
15. Consider performance, scalability, and maintainability before introducing new code.

---

# Pull Request Checklist

Before submitting code, verify:

- [ ] PSR-12 compliant
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
- [ ] Documentation updated if behavior changed