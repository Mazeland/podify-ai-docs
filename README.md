# Podify-AI

> AI-driven Print-on-Demand platform with automated workflows. Creators can set up automated image generation that creates multiple variations, performs outpainting for different product formats, and publishes to multi-tenant shops—no manual intervention required.

[![Laravel](https://img.shields.io/badge/Laravel-12-FF2D20?style=flat&logo=laravel)](https://laravel.com)
[![Vue.js](https://img.shields.io/badge/Vue.js-3-4FC08D?style=flat&logo=vue.js)](https://vuejs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?style=flat&logo=typescript)](https://www.typescriptlang.org)
[![PHP](https://img.shields.io/badge/PHP-8.4-777BB4?style=flat&logo=php)](https://www.php.net)

## What It Does

A Print-on-Demand marketplace where creators can generate AI artwork and sell it on physical products. Built with a **headless Laravel API** (DDD, SOLID) and a **Vue 3 SPA** frontend, exploring modern architecture patterns with clean separation between business logic and presentation.

**Core Features:**
- Automated AI image generation with smart variations
  - Creator provides base prompt, system generates variations automatically
  - AI prompt enhancement for better results
  - Iterative AI-to-AI feedback: second AI analyzes generated art, describes what it sees, uses that as prompt for next generation
  - AI outpainting: expand images beyond their original borders to fit different product formats (square to landscape, portrait extensions, etc.) with 4K and 8K resolution support
- Multi-tenant shop system with custom branding
- Product catalog with categories and tags
- Subscription plans with generation credits
- Role-based access (User, Creator, Moderator, Admin)

The headless API architecture enables independent deployment, better performance, and clear separation of concerns. See [Tech Stack](#tech-stack) for technical details.

## Quick Start

### Prerequisites
- PHP 8.4+
- Composer
- Node.js 20+
- MySQL 8+

### Installation

```bash
# Clone repository
git clone <repository-url>
cd ai-pod-shop

# Install dependencies
composer install
npm install

# Environment setup
cp .env.example .env
php artisan key:generate

# Database
php artisan migrate
php artisan db:seed

# Start development servers
npm run dev          # Vite (Terminal 1)
php artisan serve    # Laravel (Terminal 2)
```

Visit `http://localhost:8000`

## Architecture

Podify-AI follows **Domain-Driven Design** with a modular architecture:

```
Modules/
├── Auth/          # User authentication & authorization
├── Shop/          # Products, images, categories (DDD reference)
└── Subscription/  # Plans, billing, generation credits
```

Each module follows DDD principles with:
- **Domain Layer** - Entities, Value Objects, Repositories
- **Application Layer** - Use Cases, DTOs
- **Infrastructure Layer** - Eloquent implementations, external services
- **UI Layer** - Controllers, Resources, Vue components

**[See detailed architecture documentation](ARCHITECTURE.md)**

## Tech Stack

### Application Architecture
**Headless API + SPA** - Laravel serves as a headless API backend with no server-rendered views. The Vue 3 frontend is fully decoupled and communicates exclusively via RESTful JSON endpoints. This enables independent deployment, better frontend performance, and clear separation between business logic (backend) and presentation (frontend).

### Backend
- **Framework:** Laravel 12
- **Language:** PHP 8.4
- **Architecture:** DDD, SOLID, Modular (nwidart/laravel-modules)
- **Database:** MySQL 8+ with Eloquent ORM
- **Auth:** Laravel Sanctum (session-based)
- **Storage:** Cloudflare R2 (public/private buckets)
- **Testing:** Pest

### Frontend
- **Framework:** Vue 3 (Composition API)
- **Language:** TypeScript
- **Build Tool:** Vite
- **Styling:** Tailwind CSS
- **Router:** Vue Router (client-side routing)
- **State:** Composables pattern
- **HTTP:** Axios with interceptors

### Development Patterns
- **Error Handling:** `useNotification()` composable (no alerts)
- **API Calls:** `useApi()` composable with automatic CSRF handling
- **Type Safety:** No `as any` - generics and helper functions
- **Request Tracking:** UUID-based request IDs for traceability

**[See development conventions](DEVELOPMENT_GUIDE.md)**

## Project Structure

```
ai-pod-shop/
├── Modules/                    # Domain modules (DDD)
│   ├── Auth/                  # Authentication & users
│   ├── Shop/                  # Products & images (DDD reference)
│   └── Subscription/          # Plans & credits
├── resources/
│   ├── js/                    # Vue 3 SPA
│   │   ├── Components/       # Reusable UI components
│   │   ├── Pages/            # Inertia pages
│   │   └── composables/      # Vue composables
│   └── views/                # Blade templates (minimal)
├── ARCHITECTURE.md           # Technical architecture docs
├── DEVELOPMENT_GUIDE.md      # Development conventions
└── README.md                 # This file
```

## User Roles

| Role | Permissions |
|------|-------------|
| **User** | Browse, purchase, favorites |
| **Creator** | Create shop, generate designs, sell products |
| **Moderator** | Review content, moderate shops |
| **Admin** | Manage all users, shops, and content |

See [DEVELOPMENT_GUIDE.md](DEVELOPMENT_GUIDE.md#user-roles) for technical implementation details (policies, permissions).

## Testing

```bash
# Run all tests
php artisan test

# Run specific module tests
php artisan test --filter Auth
php artisan test --filter Shop
php artisan test --filter Subscription

# With coverage
php artisan test --coverage
```

All tests use:
- **Pest** testing framework
- **RefreshDatabase** trait
- **Factories** for test data
- Feature tests cover Policies and FormRequests

## Security

Session-based auth with CSRF protection, HttpOnly cookies, and input validation via FormRequests and Policies. Rate limiting on AI endpoints, signed URLs for uploads.

## Environment Configuration

Key environment variables:

```env
# Application
APP_ENV=local
APP_URL=http://localhost:8000

# Database
DB_CONNECTION=mysql
DB_DATABASE=ai_pod_shop

# Session (production: secure, httponly)
SESSION_DRIVER=database
SESSION_ENCRYPT=true

# Storage
CLOUDFLARE_R2_BUCKET_PUBLIC=thumbnails
CLOUDFLARE_R2_BUCKET_PRIVATE=hi-res

# AI Services
OPENAI_API_KEY=your-key-here
```

## Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Technical architecture, DDD patterns, module structure
- **[DEVELOPMENT_GUIDE.md](DEVELOPMENT_GUIDE.md)** - Development conventions, coding standards, patterns

## Development Notes

### API Response Structure
All endpoints use consistent JSON structure:
```json
{
  "success": true,
  "data": { ... },
  "message": "Optional message",
  "error": "ERROR_CODE",
  "status": 200
}
```

### Event-Driven Architecture
Modules communicate via domain events:
- `UserCreated`, `UserUpdated`, `UserDeleted` (Auth)
- `ProductCreated`, `ProductUpdated`, `ProductDeleted` (Shop)
- `SubscriptionCreated`, `SubscriptionUpdated` (Subscription)

See [ARCHITECTURE.md](ARCHITECTURE.md) for event patterns.

## Roadmap

**Core Features (Work in Progress):**
- User authentication & authorization
- AI image generation with variations
- Product catalog with categories & tags
- Multi-tenant creator shops
- Subscription plans with credits

**Frontend Improvements (Planned):**
- `useAuth()` composable for global auth state & permissions
- `usePermissions()` composable for role-based UI rendering
- `useShop()` composable for shop context management
- `usePagination()` composable for reusable pagination logic
- Image lazy loading & progressive enhancement
- Skeleton loaders for improved perceived performance
- Optimistic UI updates for better UX

**Backend Improvements (Planned):**
- Queue workers for async job processing
- Redis caching layer for frequently accessed data
- Advanced rate limiting per user role
- Webhook system for external integrations
- Admin analytics dashboard
- Database query optimization & indexing

**Business Features (Planned):**
- Payment integration (Stripe/Mollie)
- Print fulfillment partner integration
- Nested category navigation with breadcrumbs
- Advanced search with filters (price, creator, tags)
- Creator revenue dashboard & payout system
- Clothing & accessories product types
- Order management & tracking
- Customer review system
- Mobile-responsive progressive web app
