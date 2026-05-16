# CLAUDE.md

> This file provides essential context for Claude Code when working on this project. It is read automatically at the start of each session.

## Project Identity

- **Name**: Stock Trading Website (Đồ án IE104 - UIT)
- **Type**: Educational project — Internet & Web Technology course
- **Repository**: Git initialized

## What This Project Is

A stock browsing and portfolio management platform where users can:

- Browse listed stocks with real-time-like price displays
- Manage a personal watchlist of stocks they want to track
- View their virtual portfolio with P&L calculations
- View transaction history

Administrators can:

- Manage stock listings (CRUD)
- Manage user accounts (lock/unlock, deposit virtual funds)
- Monitor transactions

**Important**: No live trading — portfolio and transaction data are seeded for demo purposes. Pricing is seeded and updated manually by admins.

## Tech Stack (Locked)

| Layer | Technology | Version |
|-------|-----------|---------|
| Backend Framework | Laravel | 11.x (stable) |
| Language | PHP | 8.2+ |
| Frontend Framework | React | 18.x |
| TypeScript | TypeScript | 5.x strict |
| SPA Bridge | Inertia.js | 2.x |
| Build Tool | Vite | 5.x |
| CSS | TailwindCSS | 3.x (NOT v4) |
| UI Components | shadcn/ui | latest |
| Database | MySQL | 8.x |
| ORM | Eloquent | (Laravel built-in) |
| Charts | Recharts | latest stable |
| Auth Scaffold | Laravel Breeze | latest |
| Icons | Lucide React | latest |
| Notifications | react-hot-toast | latest |
| Date Utility | date-fns | latest |

**Do NOT use**: TailwindCSS v4 (still alpha), Inertia v1 (deprecated), PHP 8.1 or below.

## Architectural Decisions

### Decision 1: Inertia + React over Blade

- **Why**: Team prefers component-based development. Inertia keeps Laravel routing and validation while replacing Blade with React components.
- **Trade-off**: Slightly higher learning curve, but better developer experience and modern UX.

### Decision 2: Session-based Auth (not SPA + Token)

- **Why**: Inertia uses server-rendered approach. Sessions work natively. No need for JWT complexity.
- **Trade-off**: Cannot easily expose API for mobile apps in the future, but acceptable for this scope.

### Decision 3: Custom Middleware over Spatie Permissions

- **Why**: Simple role enum on users table suffices. Adding Spatie adds complexity not needed.
- **Trade-off**: Less flexibility for granular permissions, but appropriate for 2-role system.

### Decision 4: Single Monolith with Route Prefix

- **Why**: One Laravel app handles both user (`/`) and admin (`/admin`) routes. Easier to deploy and maintain.
- **Trade-off**: Cannot independently scale admin vs user, but not needed for this scope.

### Decision 5: Decimal type for money (NOT float)

- **Why**: Floating point arithmetic causes rounding errors with money. Always use `DECIMAL(15,2)`.
- **Critical**: All price/balance/total columns MUST be decimal.

## Domain Concepts

### Entities

- **User**: Person who browses stocks, manages watchlist, views portfolio and history. Has balance.
- **Stock**: A listed security (e.g., VNM, FPT). Has current_price, previous_close.
- **Transaction**: Record of a historical transaction (seeded). Append-only (immutable after creation).
- **Portfolio**: User's current holdings with avg_price. Seeded for demo.
- **Watchlist**: User's personal list of stocks to track. Can be added/removed.
- **PriceHistory**: Daily price snapshots for charting.

### Key Business Rules

1. **Inactive users**: Cannot login (return error message in Vietnamese)
2. **Watchlist idempotent**: Adding a duplicate watchlist entry uses `firstOrCreate` — no error
3. **Watchlist ownership**: Only owner can delete their watchlist entry (403 otherwise)
4. **Admin deposit**: Admin can credit virtual funds to user balance using BCMath
5. **Decimal arithmetic**: All balance/price calculations MUST use BCMath (never float arithmetic)

### Profit & Loss (Portfolio display only)

```
market_value = quantity * current_price
cost_basis   = avg_price * quantity
pnl_amount   = market_value - cost_basis
pnl_percent  = (pnl_amount / cost_basis) * 100
```

## Coding Conventions

### PHP / Laravel

- Follow PSR-12
- Always use type hints and return types
- Prefer constructor injection over `app()` helper
- Use Form Requests for all validation
- Use `DB::transaction()` for multi-step DB operations
- Soft delete sensitive entities (users, stocks)
- Use Eloquent over raw SQL (security + maintainability)

### TypeScript / React

- Strict mode ON (`tsconfig.json`)
- No `any` (use `unknown` if necessary)
- Functional components with hooks
- File naming:
  - Components in `components/`: `kebab-case.tsx`
  - Pages in `Pages/`: `PascalCase.tsx` (Inertia convention)
- Props interfaces named `<ComponentName>Props`
- Use `interface` for object shapes, `type` for unions/intersections
- Default exports for Pages (Inertia requirement), named exports for components

### CSS / Styling

- Tailwind utility-first, no custom CSS files except `app.css`
- shadcn/ui for base components, customize via CSS variables in `globals.css`
- Use `cn()` helper from `lib/utils.ts` for conditional classes
- Mobile-first: design for 360px width first, then add `md:`, `lg:` modifiers
- Dark mode: NOT in scope for this project (defer to future)

### Comments

- Vietnamese for business logic explanations
- English for code structure/architecture notes
- Use JSDoc/PHPDoc for public APIs of classes/functions
- Don't comment what is obvious from the code; comment WHY, not WHAT

## File & Folder Conventions

```
app/Http/Controllers/        # PascalCase, ends with "Controller"
app/Models/                  # PascalCase, singular
app/Http/Requests/           # PascalCase, ends with "Request"
app/Http/Middleware/         # PascalCase
database/migrations/         # YYYY_MM_DD_HHMMSS_snake_case.php
database/seeders/            # PascalCase, ends with "Seeder"
resources/js/Pages/          # PascalCase folders/files (Inertia convention)
resources/js/components/     # kebab-case files
resources/js/layouts/        # kebab-case files
resources/js/lib/            # kebab-case
resources/js/types/          # kebab-case .ts files
resources/js/hooks/          # use-kebab-case.ts
```

## Common Commands

### Development

```bash
# Start dev servers (2 terminals)
php artisan serve            # Terminal 1: http://localhost:8000
pnpm run dev                 # Terminal 2: Vite HMR

# Database
php artisan migrate          # Run new migrations
php artisan migrate:fresh    # Drop all tables, re-run migrations
php artisan migrate:fresh --seed   # Fresh + seed (most common)
php artisan db:seed          # Run seeders only

# Quality
php artisan format           # PHP CS Fixer (if configured)
pnpm run lint                # ESLint
pnpm run format              # Prettier
pnpm run type-check          # TypeScript

# Build
pnpm run build               # Production build
```

## Environment Setup

Required:

- PHP 8.2+
- Composer 2.x
- Node.js 20+ (LTS)
- MySQL 8+
- A local web server (Laragon / XAMPP / built-in `php artisan serve`)

### .env Critical Variables

```
APP_NAME="Stock Trading"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=stock_website
DB_USERNAME=root
DB_PASSWORD=
```

## Security Baseline (Non-negotiable)

1. **Input validation**: All user input goes through Form Requests
2. **CSRF**: Enabled by default (Laravel)
3. **SQL injection**: Use Eloquent, no raw queries
4. **XSS**: React escapes by default, never use `dangerouslySetInnerHTML`
5. **Mass assignment**: Always define `$fillable` on Models
6. **Authentication**: bcrypt for passwords (default)
7. **Authorization**: Middleware checks on every protected route
8. **Rate limiting**: On login, register, password reset routes
9. **Secrets**: Never in code, only in `.env` (gitignored)
10. **Dependencies**: Run `composer audit` and `pnpm audit` regularly

## Performance Targets

- Initial page load: < 2s on 3G
- Time to Interactive: < 3s
- Lighthouse score: > 90 (Performance, Accessibility)
- Database queries: Avoid N+1 (use eager loading with `with()`)

## What NOT to Do

❌ Do not impletement any test (No need to write unit test,...)
❌ Do not introduce new dependencies without documenting in ADR
❌ Do not modify generated Breeze files unnecessarily (only customize what's documented)
❌ Do not use raw SQL queries
❌ Do not commit `.env`, `vendor/`, `node_modules/`, `public/build/`, `storage/logs/`
❌ Do not commit `.claude/`, `AGENTS.md`, `CLAUDE.md`
❌ Do not use deprecated Laravel patterns (e.g., `Route::namespace()`, facade `Input::get()`)
❌ Do not bypass validation for "convenience"
❌ Do not hardcode UI strings — use a constants file for repeated strings
