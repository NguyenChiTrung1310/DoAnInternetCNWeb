# AGENTS.md

> Workflow and conventions for AI coding agents (Claude Code, Cursor, GitHub Copilot Workspace).

## Operating Principles

### 1. Read Before Write

Before writing any code, always read:

- `CLAUDE.md` — project context
- Existing related files in the codebase
- Documentation in `docs/` relevant to the task

### 2. Verify Versions

Before installing dependencies or using framework features, verify:

- Use MCP Context7 if available
- Otherwise web search for "latest stable [library name] 2026"
- Check official changelogs
- Never use beta, alpha, RC versions in production code

### 3. Security by Default

- Treat every input as malicious
- Use framework-provided validation (never roll your own)
- Use parameterized queries (Eloquent does this automatically)
- Hash passwords with bcrypt (not MD5, SHA1, plain text)
- Use CSRF tokens for state-changing operations
- Enable rate limiting on auth endpoints

### 4. Atomic Commits

- One logical change per commit
- Commit message format: `type: description`
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`
- Examples:
  - `feat: add EnsureUserIsAdmin middleware`
  - `docs: add tech-stack documentation`
  - `chore: configure ESLint with strict rules`

### 5. Document Decisions

For non-obvious technical decisions, create an ADR (Architecture Decision Record) in `docs/decisions/`:

```markdown
# ADR-001: Use TailwindCSS v3 instead of v4

## Status
Accepted

## Context
TailwindCSS v4 is in alpha/beta as of 2025. Production use risks instability.

## Decision
Use TailwindCSS v3.x latest stable.

## Consequences
- Stable, well-documented
- Can upgrade to v4 when stable
- Some new v4 features unavailable
```

## Task Execution Workflow

### Before Starting

1. Understand the task — re-read the prompt twice
2. Identify which files need to be created/modified
3. Identify dependencies that need to be installed
4. Check if there are existing patterns to follow

### During Implementation

1. Make the smallest possible change to achieve the goal
2. Write code that follows project conventions (see CLAUDE.md)
3. Add validation, error handling, types
4. Add comments only when necessary (explain WHY, not WHAT)
5. Verify the change works (manual verification)

### After Implementation

1. Run linters/formatters
2. Commit with proper message
3. Update relevant documentation

## Code Style Quick Reference

### PHP

```php
<?php

namespace App\Http\Controllers;

use App\Models\Stock;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

/**
 * Brief description.
 */
class StockController extends Controller
{
    public function __construct(
        private readonly StockService $stockService
    ) {}

    public function index(Request $request): Response
    {
        $stocks = Stock::active()
            ->when($request->search, fn ($q, $s) => $q->search($s))
            ->paginate(20);

        return Inertia::render('Stocks/Index', [
            'stocks' => $stocks,
            'filters' => $request->only('search'),
        ]);
    }
}
```

### TypeScript / React

```tsx
import { Head, useForm } from '@inertiajs/react';
import { type FormEvent } from 'react';
import AppLayout from '@/layouts/app-layout';
import { Button } from '@/components/ui/button';
import type { Stock } from '@/types/models';

interface Props {
  stocks: Stock[];
  filters: { search?: string };
}

export default function Index({ stocks, filters }: Props) {
  const { data, setData, get, processing } = useForm({
    search: filters.search ?? '',
  });

  const handleSearch = (e: FormEvent) => {
    e.preventDefault();
    get(route('stocks.index'), { preserveState: true });
  };

  return (
    <AppLayout>
      <Head title="Danh sách cổ phiếu" />
      {/* ... */}
    </AppLayout>
  );
}
```

## File Creation Patterns

### When asked to create a Model

Always include:

1. Type-hinted PHP class
2. `$fillable` array
3. `$casts` for proper type conversion
4. `$hidden` for sensitive fields (passwords, tokens)
5. Relationships methods with return type hints
6. Query scopes if commonly filtered
7. Accessors for computed fields

### When asked to create a Migration

Always include:

1. Foreign key constraints with proper cascade behavior
2. Indexes on foreign keys and frequently filtered columns
3. Soft deletes if appropriate
4. `created_at` / `updated_at` via `$table->timestamps()`
5. Proper column types (decimal for money, not float)
6. NOT NULL constraints where appropriate
7. Unique constraints where needed
8. Down migration to roll back cleanly

### When asked to create a Controller

Always include:

1. Constructor injection for dependencies
2. Type-hinted parameters and return types
3. Validation via Form Request (separate file)
4. Inertia::render for views
5. Eager loading with `with()` to avoid N+1
6. Authorization checks (middleware or policy)

### When asked to create a React Component

Always include:

1. TypeScript interface for Props
2. Functional component syntax
3. Default props if applicable
4. Proper key prop on lists
5. Accessibility attributes (aria-*, alt, role)
6. Loading and error states for async operations
7. Mobile-responsive Tailwind classes

## Error Handling Patterns

### Backend

```php
try {
    DB::transaction(function () use ($data) {
        // Multi-step operations
    });
} catch (InvalidArgumentException $e) {
    return back()->with('error', $e->getMessage())->withInput();
} catch (Throwable $e) {
    Log::error('Operation failed', [
        'user_id' => auth()->id(),
        'error' => $e->getMessage(),
    ]);
    return back()->with('error', 'Có lỗi xảy ra, vui lòng thử lại');
}
```

### Frontend

```tsx
const { post, processing, errors } = useForm({...});

const handleSubmit = (e: FormEvent) => {
  e.preventDefault();
  post(route('something'), {
    onSuccess: () => toast.success('Thành công'),
    onError: () => toast.error('Có lỗi xảy ra'),
  });
};

// In JSX:
{errors.fieldName && (
  <p className="text-sm text-red-500">{errors.fieldName}</p>
)}
```

## When You're Unsure

1. **Check documentation first** (Laravel docs, React docs, Inertia docs)
2. **Search for patterns** in the existing codebase
3. **Verify with Context7 MCP** if available
4. **Ask the user** if the question is about business logic
5. **Document the decision** in an ADR after consensus

## Anti-Patterns (Things to Avoid)

### ❌ Pattern: Logic in Controllers

```php
// BAD: Business logic in controller
public function buy(Request $request, Stock $stock) {
    $user = $request->user();
    $quantity = $request->quantity;
    if ($user->balance < $stock->current_price * $quantity) {
        // ...lots of logic
    }
    // ...more logic
}
```

```php
// GOOD: Thin controller, business logic in service
public function buy(PlaceOrderRequest $request, Stock $stock): RedirectResponse {
    try {
        $this->tradingService->buy(
            user: $request->user(),
            stock: $stock,
            quantity: $request->validated('quantity'),
        );
        return back()->with('success', 'Đặt lệnh thành công');
    } catch (InvalidArgumentException $e) {
        return back()->with('error', $e->getMessage());
    }
}
```

### ❌ Pattern: Inline styles instead of Tailwind

```tsx
// BAD
<div style={{ padding: '16px', background: 'white' }}>

// GOOD
<div className="p-4 bg-white">
```

### ❌ Pattern: `any` type

```tsx
// BAD
const data: any = JSON.parse(response);

// GOOD
const data: ApiResponse = JSON.parse(response);
// or
const data: unknown = JSON.parse(response);
if (isApiResponse(data)) { /* type narrowed */ }
```

### ❌ Pattern: Raw SQL

```php
// BAD
$users = DB::select("SELECT * FROM users WHERE email = '{$email}'");

// GOOD
$users = User::where('email', $email)->get();
```

### ❌ Pattern: Direct DOM manipulation

```tsx
// BAD
useEffect(() => {
  document.getElementById('myInput')?.focus();
}, []);

// GOOD
const inputRef = useRef<HTMLInputElement>(null);
useEffect(() => {
  inputRef.current?.focus();
}, []);
```

## Output Format

### After completing a task

Always output:

1. **Summary**: What was done in 1-2 sentences
2. **Files affected**: List of files created/modified
3. **Commands run**: Bash commands executed
4. **Next steps**: What should happen next
5. **Issues encountered**: Any problems or trade-offs

### Example

```markdown
## Task Complete: Add EnsureUserIsAdmin Middleware

### Summary
Created custom middleware to gate admin routes by checking user role.

### Files Affected
- Created: `app/Http/Middleware/EnsureUserIsAdmin.php`
- Modified: `bootstrap/app.php` (registered alias `admin`)
- Modified: `routes/web.php` (applied to admin route group)

### Commands Run
- `php artisan make:middleware EnsureUserIsAdmin`

### Issues / Notes
- Considered using Laravel Policy instead, but middleware is simpler for binary admin/non-admin
- Documented in ADR-004
```

## Special Instructions

### Working with Database

- Always run migrations in development with `php artisan migrate:fresh --seed`
- Always verify seeders work from scratch
- Always include rollback (`down()`) in migrations
- Use transactions for multi-table operations

### Working with Forms

- Server-side validation in Form Request (mandatory)
- Client-side validation is a UX enhancement, not security
- Always handle the loading state (`processing` from `useForm`)
- Always show validation errors next to fields

### Working with Inertia

- Pages are React components in `resources/js/Pages/`
- Use `Inertia::render('PageName', $data)` from controllers
- Use `Link` component for navigation (not `<a>`)
- Use `useForm()` hook for forms (not raw fetch)
- Use flash messages via `back()->with('success', 'message')`

### Working with Authentication

- Use `auth()->user()` or `$request->user()` to get current user
- Check `$user->is_admin` (accessor) for role checks
- Logout: redirect to `/` (not `/login`)
- Login redirect: based on role (`/admin` or `/dashboard`)
