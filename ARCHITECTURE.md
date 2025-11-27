# Architecture Documentation

## Overview

Technical architecture for Podify-AI, following Domain-Driven Design and SOLID principles with a modular approach.

## Core Principles

- **Domain-Driven Design**: Bounded contexts, domain entities, domain events
- **SOLID**: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion
- **Patterns**: Repository, UseCases, Anti-Corruption Layers, Domain Events

## Request Flow

Request handling follows a clean separation of concerns:

```
HTTP Request
    ↓
FormRequest (validation + authorization)
    ↓
Controller (coordination only - no business logic)
    ↓
UseCase (business workflow)
    ↓
Repository (database abstraction)
    ↓
Entity (domain model)
```

**Example: User filtering**

```php
// 1. Request comes in
GET /api/admin/users?role=creator&per_page=15

// 2. GetUsersRequest validates input and checks authorization
// 3. AdminUserController creates DTO and calls UseCase
$users = $this->getAllUsersUseCase->execute($filters);

// 4. UseCase delegates to repository
$result = $this->userRepository->findWithFilters($filters);

// 5. Repository queries database and returns User entities
// 6. Controller transforms entities to JSON via UserResource
// 7. Response returned: { "success": true, "data": [...] }
```

**Key principles:**
- Controllers don't contain business logic
- UseCases are testable without HTTP layer
- Repositories abstract database implementation
- Entities are framework-agnostic

### What Each Layer Does

**Important:** All layers exist **within each module** at `Modules/{ModuleName}/`. There is no global `app/UseCases/` folder - UseCases live in `Modules/{ModuleName}/app/UseCases/`.

**UI Layer (`Modules/{ModuleName}/UI/`)**
- FormRequests for validation and authorization
- Resources transform entities to JSON
- Controllers coordinate between layers
- Keeps business logic out

**Application Layer (`Modules/{ModuleName}/app/`)**
- UseCases handle business workflows
- DTOs for complex data (mainly Shop module)
- Policies for authorization rules
- Jobs for background tasks

**Domain Layer (`Modules/{ModuleName}/Domain/`)**
- Entities as core business objects
- Repository interfaces (no implementation)
- Value Objects for immutable concepts (Shop module)
- Domain Events for cross-module communication
- Framework-agnostic

**Infrastructure Layer (`Modules/{ModuleName}/Infrastructure/`)**
- Repository implementations using Eloquent
- External service integrations (OpenAI, R2)
- Anti-Corruption Layers for external data
- All database queries live here

### Layer Boundaries (Do's and Don'ts)

**Domain Layer**
- **Do:** Pure business logic, entities, value objects, repository interfaces
- **Do:** Use PHP 8.4 features (readonly, enums, typed properties)
- **Don't:** Import Laravel classes, Eloquent, HTTP, or any framework code
- **Don't:** Know about databases, sessions, requests, or external services

**Application Layer**
- **Do:** Orchestrate business workflows via UseCases
- **Do:** Use repositories (interfaces) and dispatch domain events
- **Do:** Implement policies (DTOs only in Shop module for complex filtering)
- **Don't:** Use Eloquent directly (use repositories)
- **Don't:** Access HTTP request/response (get data from controllers)
- **Don't:** Contain business logic (that belongs in Domain)

**Infrastructure Layer**
- **Do:** Implement repository interfaces with Eloquent
- **Do:** Integrate external services (OpenAI, R2, mail, queues)
- **Do:** Handle all database queries and ORM operations
- **Don't:** Contain business logic (delegate to Domain/Application)
- **Don't:** Be aware of HTTP layer details

**UI Layer**
- **Do:** Handle HTTP concerns (validation, transformation, routing)
- **Do:** Delegate to UseCases for business logic
- **Do:** Transform entities to JSON via Resources
- **Don't:** Contain business logic (controllers stay thin)
- **Don't:** Access Eloquent models directly (use repositories)

## Module Structure

```
Modules/{ModuleName}/
├── Domain/                          # Pure business logic
│   ├── Entities/                    # Core business objects
│   ├── Repositories/                # Data access interfaces
│   ├── Events/                      # Domain events
│   ├── Enums/                       # Domain enumerations (Auth only)
│   ├── ValueObjects/                # Immutable business concepts (Shop only)
│   ├── Specifications/              # Business rules (Shop only)
│   ├── Services/                    # Domain services
│   └── Exceptions/                  # Domain exceptions
├── Infrastructure/
│   ├── Persistence/Eloquent/
│   │   ├── Models/                  # Eloquent ORM models
│   │   └── Repositories/            # Repository implementations
│   ├── AntiCorruption/              # ACL (Shop & Subscription)
│   ├── Services/                    # External integrations
│   ├── Http/Middleware/             # Security middleware (Auth only)
│   └── Providers/                   # Service providers (Auth only)
├── app/                             # Application layer
│   ├── UseCases/                    # Business workflows
│   ├── Policies/                    # Authorization
│   ├── Actions/                     # Fortify/Jetstream actions (Auth only)
│   ├── DTO/                         # Data Transfer Objects (Shop only)
│   ├── Commands/                    # Artisan commands
│   ├── Jobs/                        # Background jobs
│   └── Services/                    # Application services
├── UI/                              # Presentation layer
│   ├── Http/
│   │   ├── Controllers/             # Request coordination
│   │   ├── Requests/                # FormRequests (validation + auth)
│   │   ├── Resources/               # JSON transformation
│   │   └── Middleware/              # HTTP middleware
│   └── Presenters/                  # View presenters (Auth & Shop)
├── Database/
│   ├── Migrations/                  # Database schema
│   ├── Seeders/                     # Test data
│   └── Factories/                   # Model factories
├── Providers/                       # Module service providers
│   ├── {Module}ServiceProvider.php
│   └── RouteServiceProvider.php
├── routes/                          # Module routes
│   ├── api.php
│   └── web.php
├── config/                          # Module config
└── resources/js/                    # Frontend (Subscription only)
```

## Module Characteristics

### Auth Module (User Authentication & Authorization)
**Complexity:** Simple
**Pattern Level:** Basic DDD

**Implemented:**
- Enums: UserRole for type-safe roles
- UseCases: GetAllUsersUseCase for business workflows
- FormRequests: GetUsersRequest for validation + authorization
- Actions: Fortify/Jetstream integration (CreateNewUser, UpdateUserPassword)
- Presenters: Transform domain data for views

**Not Needed (By Design):**
- ValueObjects: Simple domain, strings sufficient
- Specifications: No complex business rules
- DTOs: Arrays adequate for simple use cases
- Anti-Corruption Layer: Auth is the source of user data, not a consumer

### Shop Module (Product Catalog & Creator Shops)
**Complexity:** Complex
**Pattern Level:** Full DDD

**Implemented:**
- ValueObjects: Price, Dimensions (complex validation)
- Specifications: Business rules for product eligibility
- DTOs: ProductFilters, CategoryFilters (type-safe filtering)
- UseCases: Create/Update/Delete for Product/Category
- FormRequests: Store/UpdateProductRequest, Store/UpdateCategoryRequest
- Anti-Corruption Layer: AuthUserAdapter (translates Auth user data)
- Presenters: Transform domain data for views

**Why More Complex:**
- E-commerce domain requires strict validation
- Complex filtering and search requirements
- Revenue calculations need precision
- Multiple bounded contexts interact

### Subscription Module (Plans & Billing)
**Complexity:** Medium
**Pattern Level:** Standard DDD

**Implemented:**
- UseCases: Subscription workflows
- Anti-Corruption Layer: AuthUserAdapter (translates Auth user data)
- Frontend Components: resources/js/ (module-specific UI)

**Not Needed (By Design):**
- ValueObjects: Simple price model (not e-commerce)
- Specifications: Straightforward subscription rules
- DTOs: Arrays adequate for current needs
- Presenters: JSON Resources sufficient

**Future Considerations:**
- Add DTOs if filtering becomes complex
- Add Specifications for complex plan eligibility rules

## Design Principles

**Complexity Matches Domain:**
Each module uses patterns appropriate for its domain complexity. Simple domains (Auth) avoid unnecessary abstraction, complex domains (Shop) use full DDD patterns.

**Anti-Corruption Layer Pattern:**
- Auth = Source (provides user data)
- Shop = Consumer (adapts Auth user data via AuthUserAdapter)
- Subscription = Consumer (adapts Auth user data via AuthUserAdapter)

This prevents tight coupling and allows Auth to evolve independently.

## ID Strategy: Strings in Domain, Integers in Database

**Why:**
- Future-proof (easy UUID migration)
- JavaScript compatibility (no int64 issues)
- Type-safe (explicit casting at boundaries)

**Implementation:**
```php
// Domain: Always strings
class Product {
    public function __construct(
        public readonly string $id,
        public readonly string $userId,
    ) {}
}

// Repository: Cast at boundaries
use Shared\Infrastructure\Persistence\IdCaster;

public function findById(string $id): ?Product {
    $model = ProductModel::find(IdCaster::toDatabase($id));  // string → int
    return $model ? $this->toEntity($model) : null;
}

private function toEntity(ProductModel $model): Product {
    return new Product(
        id: IdCaster::toDomain($model->id),        // int → string
        userId: IdCaster::toDomain($model->user_id),
    );
}
```

**IdCaster Methods:**
- `toDatabase(string|int): int` - Convert to DB format
- `toDomain(mixed): string` - Convert to domain format
- `toDomainNullable(mixed): ?string` - Nullable conversion

## Bounded Contexts

### 1. Auth Context
- **Entities**: User, Role, Permission
- **Features**: Session-based auth (Sanctum), role hierarchy, 2FA
- **Events**: `UserCreated`, `UserUpdated`, `UserDeleted`

### 2. Shop Context
- **Entities**: Product, Category, Tag, Image, Shop
- **Features**: Categories with parent-child support (foundation ready, navigation planned), Cloudflare R2 storage, creator shops
- **Events**: `ImageCreated`, `TagCreated`, `ProductCreated/Updated/Deleted`
- **ACL**: AuthUserAdapter for user data

### 3. Subscription Context
- **Entities**: Subscription, Plan, GenerationCredit
- **Features**: Tiered plans, AI credit system, billing cycles
- **Events Consumed**: `UserCreated` (initialize free credits)
- **Events Published**: `SubscriptionCreated`, `SubscriptionUpdated`
- **ACL**: AuthUserAdapter for user data

## Communication Patterns

### 1. Domain Events
Modules communicate via events to avoid tight coupling:

```php
// Publish event
event(new UserCreated(
    userId: (string) $user->id,
    email: $user->email,
    name: $user->name
));

// Listen to event (Subscription module)
class InitializeFreeCreditsForNewUser implements ShouldQueue {
    public function handle(UserCreated $event): void {
        $this->creditService->initializeCredits($event->userId(), 10);
    }
}
```

**Domain Event Structure:**
- **Base class:** `Shared/Domain/Events/DomainEvent.php`
- **Events:** `Modules/{ModuleName}/Domain/Events/` (e.g., `UserCreated.php`, `ProductCreated.php`)
- **Listeners:** `Modules/{ModuleName}/Infrastructure/Listeners/`
- **Registration:** `Modules/{ModuleName}/Providers/EventServiceProvider.php`

All events extend `DomainEvent` (includes `eventId`, `occurredAt`, `toArray()`). Never place events in Laravel's default `app/Events/` - keep them in module Domain layers.

### 2. Anti-Corruption Layer
Translate data between contexts to prevent coupling:

```php
// Shop/Infrastructure/AntiCorruption/AuthUserAdapter.php
class AuthUserAdapter {
    public function toShopUser(UserId $userId): ?ShopUser {
        $authUser = $this->userRepository->findById($userId->value);
        return $authUser ? new ShopUser($userId, $authUser->name()) : null;
    }
}
```

### 3. Repository Pattern

Repositories abstract data access and enforce the following SOLID principles:

**Core Principles:**
1. **Repository loads only the aggregate root** - No eager loading of relations
2. **Domain entities contain only IDs** - Not fully loaded related objects
3. **Presentation layer resolves relations** - When needed, via separate repository calls
4. **Single query per method** - Avoid N+1 problems with batch operations

#### Correct Pattern

```php
// Interface: Modules/Shop/Domain/Repositories/ProductRepository.php
interface ProductRepository {
    public function findById(string $id): ?Product;
    public function findAll(): array;
    public function findAllPaginated(int $perPage = 24, int $page = 1): array;
    public function create(array $data): Product;
    public function update(string $id, array $data): ?Product;
    public function delete(string $id): bool;
}

// Implementation: Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentProductRepository.php
class EloquentProductRepository implements ProductRepository {
    public function findById(string $id): ?Product {
        $model = ProductModel::find(IdCaster::toDatabase($id));
        return $model ? $this->toEntity($model) : null;
    }

    public function findAllPaginated(int $perPage = 24, int $page = 1): array {
        // AVOID: with(['image', 'user']) - Repository shouldn't assume relations are needed
        // CORRECT: Load only aggregate root
        $paginator = ProductModel::orderBy('created_at', 'desc')
            ->paginate($perPage, ['*'], 'page', $page);

        $products = $paginator->getCollection()
            ->map(fn($model) => $this->toEntity($model))
            ->toArray();

        return [
            'data' => $products,
            'pagination' => [
                'current_page' => $paginator->currentPage(),
                'last_page' => $paginator->lastPage(),
                'per_page' => $paginator->perPage(),
                'total' => $paginator->total(),
            ],
        ];
    }

    private function toEntity(ProductModel $model): Product {
        return new Product(
            id: IdCaster::toDomain($model->id),
            name: $model->name,
            userId: IdCaster::toDomain($model->user_id),      // Only ID, not User object
            imageId: IdCaster::toDomainNullable($model->image_id), // Only ID, not Image object
            categoryId: IdCaster::toDomainNullable($model->category_id),
            // ... other scalar fields
        );
    }
}
```

#### Domain Entity (Only IDs, No Relations)

```php
// Modules/Shop/Domain/Entities/Product.php
class Product {
    public function __construct(
        public readonly string $id,
        public readonly string $name,
        public readonly string $userId,        // ID only
        public readonly ?string $imageId,      // ID only
        public readonly ?string $categoryId,   // ID only
        // ... other fields
    ) {}

    // WRONG: public User $user
    // WRONG: public Image $image
}
```

#### Presentation Layer (Batch Loading for Collections)

Batch loading eliminates N+1 queries for paginated results:

```php
// Modules/Shop/UI/Http/Resources/ProductResource.php
class ProductResource extends JsonResource {
    private static ?array $userCache = null;
    private static ?array $imageCache = null;

    public static function collection($resource): AnonymousResourceCollection {
        // Extract unique IDs
        $userIds = [];
        $imageIds = [];
        foreach ($resource as $product) {
            if ($product->userId) $userIds[] = $product->userId;
            if ($product->imageId) $imageIds[] = $product->imageId;
        }

        // Batch load in 2 queries instead of N×2
        if (!empty($userIds)) {
            self::$userCache = app(UserRepository::class)->findByIds(array_unique($userIds));
        }
        if (!empty($imageIds)) {
            self::$imageCache = app(ImageRepository::class)->findByIds(array_unique($imageIds));
        }

        return parent::collection($resource);
    }

    public function toArray($request): array {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'user' => self::$userCache[$this->userId] ?? null,  // From cache
            'image' => self::$imageCache[$this->imageId] ?? null, // From cache
        ];
    }
}
```

**Performance Impact (24 products per page):**
- Without batch loading: 49 queries (1 + 24×2)
- With batch loading: 3 queries (1 + 2)
- **16x improvement**

**Note:** Pagination (default 24/page) keeps batch loading efficient. For larger pages, consider lazy loading images in frontend.

**Why This Is Better:**

1. **Single Responsibility** - Repository only loads aggregate root
2. **Interface Segregation** - Consumers get only what they ask for
3. **Dependency Inversion** - Repository doesn't know about use cases
4. **Performance** - Batch loading in presentation layer (no N+1)
5. **Flexibility** - Different views can load different relations
6. **Testability** - Easy to mock repositories in isolation

**Repository Implementations:**
- `Modules/Auth/Infrastructure/Persistence/Eloquent/Repositories/EloquentUserRepository.php`
- `Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentProductRepository.php`
- `Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentCategoryRepository.php`
- `Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentImageRepository.php`
- `Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentTagRepository.php`
- `Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentShopRepository.php`
- `Modules/Subscription/Infrastructure/Persistence/Eloquent/Repositories/EloquentSubscriptionRepository.php`

## DTOs (Shop Module Only)

Type-safe data containers for complex use cases:

```php
readonly class CreateImageRequest {
    public function __construct(
        public string $name,
        public string $userId,
        public string $status = 'pending',
    ) {}
}
```

**Benefits:** Type safety, IDE support, immutability, self-documenting

## Design Rationale

**Module-Specific Patterns:**
- **DTO Usage**: Auth/Subscription modules use simple data structures; Shop module uses DTOs for complex filtering (ProductFilters/CategoryFilters)
- **ValueObject Usage**: Complex validation patterns implemented where domain complexity requires it (Shop module)
- **Module Structures**: Each module's complexity matches its domain requirements

## Complete Example: Creating a Product

**Request Flow:**
```
POST /api/products → StoreProductRequest (validation) → ProductController → CreateProductUseCase → ProductRepository → Database
```

**1. FormRequest validates and authorizes**
```php
// Modules/Shop/UI/Http/Requests/StoreProductRequest.php
public function authorize(): bool { return $this->user() !== null; }
public function rules(): array { return ['name' => 'required|string|max:255', ...]; }
```

**2. Controller delegates to UseCase**
```php
// Modules/Shop/UI/Http/Controllers/ProductController.php
public function store(StoreProductRequest $request, CreateProductUseCase $createProduct) {
    $product = $createProduct->execute($request->validatedWithUserId());
    return ApiResponseBuilder::make()->withData('product', new ProductResource($product))->build();
}
```

**3. UseCase handles business logic and dispatches event**
```php
// Modules/Shop/app/UseCases/CreateProductUseCase.php
public function execute(array $data): Product {
    $product = $this->productRepository->create($data);
    event(new ProductCreated($product->id, $product->name, $data['slug'], $product->userId));
    return $product;
}
```

**4. Repository (Infrastructure) uses IdCaster for int↔string conversion**
```php
// Modules/Shop/Infrastructure/Persistence/Eloquent/Repositories/EloquentProductRepository.php
public function create(array $data): Product {
    $model = ProductModel::create($data);
    return new Product(
        id: IdCaster::toDomain($model->id),  // int → string
        userId: IdCaster::toDomain($model->user_id),
        ...
    );
}
```

**Key Points:**
- FormRequest handles validation and authorization before controller runs
- Controller has zero business logic
- UseCase contains business workflow and dispatches domain events
- Repository casts int (DB) ↔ string (domain) at boundary
- Domain entity is readonly and framework-agnostic

## Non-Goals

Things I deliberately avoided:

- **Pure textbook DDD** - Not every module needs the same level of abstraction. Auth is simpler than Shop, and that's fine.
- **Microservices** - Modules are separate conceptually, not physically. Premature splitting would add complexity without benefit.
- **Every entity as ValueObject** - Only used where validation/immutability genuinely helps (like Price, Color in Shop).
- **Repository for everything** - Simple lookups sometimes just need Eloquent. Repositories are for complex queries and abstraction.
- **100% framework-agnostic domain** - PHP has great features. Using readonly, enums, and typed properties makes sense even if it couples to PHP 8.4.

## What I Learned

Building this taught me practical lessons about implementing SOLID and DDD principles in a real-world application:

- **Designed custom DDD structure within Laravel modules** - nwidart/laravel-modules provides basic module scaffolding, but I learned to structure Domain, Application, Infrastructure, and UI layers according to DDD principles. This required understanding bounded contexts and designing the folder hierarchy from scratch.
- **Applied DDD pragmatically per domain** - Implemented basic structure for Auth, but added ValueObjects and Specifications for Shop's complex business logic. Learned that forcing patterns everywhere creates unnecessary complexity; let domain needs guide architectural decisions.
- **Implemented string IDs at boundaries** - Designed IdCaster pattern to handle JavaScript's safe integer limit (2^53). This boundary strategy keeps domain entities clean while preventing int64 overflow issues that are nearly impossible to debug in production.
- **Solved middleware ordering issues** - Debugged double middleware registration in Laravel's module lifecycle. Documented the solution because these edge cases are easy to forget and painful to rediscover.
- **Refactored to repository pattern for performance** - Implemented batch loading that reduced 49 queries to 3 for 24 products per page (16x improvement). Clean abstractions made this optimization a 30-minute refactor instead of a multi-day rewrite.
- **Enforced TypeScript without escape hatches** - Eliminated all `as any` casts by designing generic types and helper functions upfront. This discipline prevents type safety from breaking at runtime during refactors.
- **Built event-driven architecture** - Designed cross-module communication via domain events. Subscription module reacts to Auth events without coupling, enabling independent refactoring of each bounded context.

## Summary

This architecture implements Domain-Driven Design with clean architecture principles, providing clear separation between domain logic, business workflows, and infrastructure concerns. The modular structure enables independent development of bounded contexts while maintaining consistency through shared patterns and domain events.

### Middleware Configuration (Critical)

**Incident Report: Double Middleware Authentication Bug**

**Problem:**
- Users receiving `401 Unauthenticated` errors despite valid session cookies
- Intermittent failures on protected routes
- Session appears valid in browser dev tools but backend rejects requests

**Root Cause:**
- `bootstrap/app.php` automatically adds `EnsureFrontendRequestsAreStateful` to ALL `api/*` routes
- Manually adding `stateful` middleware in route definitions creates **double middleware stack**
- Second middleware instance overwrites first session, causing authentication failure

**Solution:**
Never manually add `stateful` middleware to API routes - it's already applied globally.

**Correct Route Definition:**
```php
// CORRECT - stateful already applied via api middleware group
Route::middleware(['auth:web', 'admin'])->prefix('admin')->group(function () {
    Route::get('/users', [AdminUserController::class, 'index']);
});

// WRONG - duplicates EnsureFrontendRequestsAreStateful
Route::middleware(['stateful', 'auth:web', 'admin'])->prefix('admin')->group(function () {
    Route::get('/users', [AdminUserController::class, 'index']);
});
```

**Middleware Stack Order (bootstrap/app.php):**
```php
configureApiMiddleware($middleware) {
    $middleware->api(prepend: [
        HandleCors::class,
        EnsureFrontendRequestsAreStateful::class,  // Applied to ALL api/* routes
    ]);
}
```

**Session Cookie Configuration (Same-Origin via Vite Proxy):**

The application uses a Vite proxy to ensure all API requests are same-origin, eliminating cross-origin cookie issues:

**Development Setup:**
1. **Vite serves at:** `https://podify-ai.test:5174`
2. **API requests go to:** `https://podify-ai.test:5174/api/*` (same-origin)
3. **Vite proxies to:** `https://podify-ai.test/api/*` (Laravel backend)
4. **Result:** Browser sees same-origin, cookies work seamlessly

**vite.config.js Configuration:**
```javascript
export default defineConfig({
    server: {
        host: 'podify-ai.test',
        port: 5174,
        proxy: {
            '/api': {
                target: 'https://podify-ai.test',
                changeOrigin: false,  // Keep origin header
                secure: false,        // Allow self-signed certs
                ws: true,
            },
        },
    },
});
```

**Axios Configuration (useApi.ts):**
```typescript
const api = axios.create({
    baseURL: '',  // Empty = relative URLs, same-origin requests
    withCredentials: true,
    headers: {
        'X-Requested-With': 'XMLHttpRequest',
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    },
});
```

**Session Cookie Settings (.env):**
- `SESSION_SAME_SITE=lax` - Safe for same-origin (better CSRF protection than 'none')
- `SESSION_SECURE=true` - HTTPS only
- `SESSION_DOMAIN=null` - Works for all Herd domains
- `SESSION_HTTP_ONLY=true` - JavaScript cannot read cookie
- `SESSION_PARTITIONED_COOKIE` - Not needed with same-origin

**Custom Sanctum Middleware:**
Created `app/Http/Middleware/EnsureFrontendRequestsAreStateful.php` to override Sanctum's hardcoded SameSite=lax:
```php
protected function configureSecureCookieSessions(): void
{
    config([
        'session.http_only' => true,
        'session.same_site' => config('session.same_site', 'lax'),
    ]);
}
```

**Debugging Session Issues:**
```bash
# Check Sanctum stateful domains
php artisan tinker --execute="print_r(config('sanctum.stateful'));"

# Verify session config
php artisan config:show session | grep -E "(secure|same_site|domain)"

# Test same-origin cookie flow (browser console)
await fetch('/api/auth/user', {credentials: 'include'}).then(r => r.json());
```

**Same-Origin Authentication Implementation (2025-01-22):**
1. Global `stateful` middleware applied in `bootstrap/app.php`
2. Axios configured with relative URLs (`baseURL: ''`) for same-origin requests
3. Vite proxy handles `/api` and `/sanctum` routes in development
4. Session-based auth with proper SameSite=lax configuration
5. Custom middleware respects session configuration
6. Seamless same-origin authentication in both dev and production

### Key Takeaways

1. **Continuous Improvement**: Feature [roadmap](README.md#roadmap) documented and prioritized for future development
2. **Test Coverage**: Shop module has 25/25 tests passing with comprehensive coverage of UseCases, Policies, and FormRequests
3. **Middleware Order Matters**: Duplicate middleware causes subtle session bugs - follow documented patterns
