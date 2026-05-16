---
name: inertia-react
description: Use this skill when creating or modifying Inertia.js pages, components, or forms in React + TypeScript. Covers page conventions, useForm hook, navigation with Link, flash messages, shared data, and avoiding common pitfalls like direct fetch calls or wrong Page imports.
---

# Inertia.js + React Best Practices

## Page Component Structure

Every page in `resources/js/Pages/` follows this pattern:

```tsx
import { Head } from '@inertiajs/react';
import AppLayout from '@/layouts/app-layout';
import type { Stock } from '@/types/models';
import type { PageProps } from '@/types/inertia';

interface Props extends PageProps {
  stocks: Stock[];
  // other server-passed props
}

export default function Index({ stocks, auth }: Props) {
  return (
    <AppLayout>
      <Head title="Page Title" />
      {/* content */}
    </AppLayout>
  );
}
```

### Critical Rules

- **Default export only** for Page components (Inertia requirement)
- **Named exports** for non-Page components
- Always use `<Head>` for the page title (SEO + UX)
- Pages should be thin: extract logic into hooks/components

## Navigation

### Use `Link` for internal navigation

```tsx
import { Link } from '@inertiajs/react';

// GOOD: Inertia handles the request, no full page reload
<Link href={route('stocks.show', stock.symbol)}>
  View Details
</Link>

// BAD: Causes full page reload
<a href="/stocks/VNM">View Details</a>
```

### Programmatic navigation

```tsx
import { router } from '@inertiajs/react';

router.visit(route('stocks.index'));
router.visit(route('stocks.index'), { method: 'get', preserveScroll: true });
router.delete(route('stocks.destroy', id), {
  onSuccess: () => toast.success('Đã xóa'),
});
```

## Forms with useForm

### Standard Form Pattern

```tsx
import { useForm } from '@inertiajs/react';
import type { FormEvent } from 'react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export default function Create() {
  const { data, setData, post, processing, errors, reset } = useForm({
    symbol: '',
    company_name: '',
    current_price: 0,
  });

  const handleSubmit = (e: FormEvent) => {
    e.preventDefault();
    post(route('admin.stocks.store'), {
      onSuccess: () => reset(),
      onError: (errs) => console.error(errs),
    });
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <Label htmlFor="symbol">Mã CK</Label>
        <Input
          id="symbol"
          value={data.symbol}
          onChange={(e) => setData('symbol', e.target.value)}
          disabled={processing}
        />
        {errors.symbol && (
          <p className="mt-1 text-sm text-red-500">{errors.symbol}</p>
        )}
      </div>

      <Button type="submit" disabled={processing}>
        {processing ? 'Đang lưu...' : 'Lưu'}
      </Button>
    </form>
  );
}
```

### Form Options

```tsx
post(url, {
  preserveScroll: true,           // Don't reset scroll on success
  preserveState: true,            // Keep form state (useful for filters)
  onStart: () => {},              // Before request
  onSuccess: () => {},            // After 200
  onError: (errors) => {},        // After 422
  onFinish: () => {},             // After response (success or error)
});
```

### Available Methods

- `get(url, options?)` — GET request
- `post(url, options?)` — POST
- `put(url, options?)` — PUT
- `patch(url, options?)` — PATCH
- `delete(url, options?)` — DELETE
- `submit(method, url, options?)` — Custom

## Flash Messages

### Backend (Controller)

```php
return back()->with('success', 'Tạo thành công');
return back()->with('error', 'Có lỗi xảy ra');
return redirect()->route('stocks.index')->with('success', 'Đã lưu');
```

### Frontend (consume flash via shared data)

Setup in `HandleInertiaRequests.php` middleware:

```php
'flash' => [
    'success' => fn () => $request->session()->get('success'),
    'error' => fn () => $request->session()->get('error'),
],
```

In React (typically in a layout):

```tsx
import { usePage } from '@inertiajs/react';
import { useEffect } from 'react';
import { toast } from 'react-hot-toast';

export default function AppLayout({ children }: { children: React.ReactNode }) {
  const { flash } = usePage<PageProps>().props;

  useEffect(() => {
    if (flash.success) toast.success(flash.success);
    if (flash.error) toast.error(flash.error);
  }, [flash]);

  return <>{children}</>;
}
```

## Shared Data (Auth User, etc.)

### Backend

In `app/Http/Middleware/HandleInertiaRequests.php`:

```php
public function share(Request $request): array
{
    return [
        ...parent::share($request),
        'auth' => [
            'user' => fn () => $request->user()?->only(['id', 'name', 'email', 'role']),
        ],
        'flash' => [
            'success' => fn () => $request->session()->get('success'),
            'error' => fn () => $request->session()->get('error'),
        ],
    ];
}
```

### Frontend

```tsx
import { usePage } from '@inertiajs/react';
import type { PageProps } from '@/types/inertia';

export default function MyComponent() {
  const { auth } = usePage<PageProps>().props;
  return <div>Hello, {auth.user.name}</div>;
}
```

## Common Pitfalls

### ❌ Using fetch/axios for app navigation

```tsx
// BAD: bypasses Inertia, breaks SPA experience
fetch('/api/stocks').then(res => res.json());
```

```tsx
// GOOD: use Inertia router
router.reload({ only: ['stocks'] });
```

### ❌ Forgetting to use named routes

```tsx
// BAD: hardcoded paths
<Link href="/admin/stocks/123/edit">Edit</Link>
```

```tsx
// GOOD: named route helper (requires ziggy-js)
<Link href={route('admin.stocks.edit', stock.id)}>Edit</Link>
```

### ❌ Missing TypeScript Props interface

```tsx
// BAD
export default function Index(props) { ... }
```

```tsx
// GOOD
interface Props extends PageProps {
  stocks: Stock[];
}
export default function Index({ stocks }: Props) { ... }
```

### ❌ Reset form on every keystroke

```tsx
// BAD: causes infinite re-renders
useEffect(() => {
  reset();
}, [data]);
```

### ❌ Multiple `<Head>` tags

```tsx
// BAD
<Head title="A" />
<Head title="B" />  // overwrites A
```

## Page Reload Behavior

### Full reload (default)

```tsx
router.visit('/stocks');  // Loads fresh data
```

### Partial reload (only specific props)

```tsx
router.reload({ only: ['stocks'] });
```

### Preserve state (for filter/search forms)

```tsx
router.get('/stocks', { search: 'VNM' }, {
  preserveState: true,
  preserveScroll: true,
});
```

## Modal/Dialog Pattern

Inertia doesn't have built-in modal routes. Use one of:

### Pattern 1: State-based modal (simple)

```tsx
const [open, setOpen] = useState(false);

<Dialog open={open} onOpenChange={setOpen}>
  <DialogContent>
    {/* Edit form */}
  </DialogContent>
</Dialog>
```

### Pattern 2: URL-based modal (for shareable links)

```tsx
const { props } = usePage<{ editing?: Stock }>();

<Dialog open={!!props.editing} onOpenChange={(open) => {
  if (!open) router.get(route('stocks.index'));
}}>
  ...
</Dialog>
```

## Loading States

### Pending state on navigation

```tsx
import { router } from '@inertiajs/react';
import { useState, useEffect } from 'react';

const [loading, setLoading] = useState(false);

useEffect(() => {
  const start = () => setLoading(true);
  const finish = () => setLoading(false);
  router.on('start', start);
  router.on('finish', finish);
  return () => {
    // Cleanup
  };
}, []);
```

### Better: Use form's processing state

```tsx
const { processing } = useForm({...});
<Button disabled={processing}>
  {processing ? 'Đang xử lý...' : 'Submit'}
</Button>
```

## File Naming Convention

```
resources/js/Pages/
  ├── Welcome.tsx                 # PascalCase
  ├── Dashboard.tsx
  ├── Stocks/
  │   ├── Index.tsx               # Folder + PascalCase file
  │   └── Show.tsx
  └── Admin/
      └── Stocks/
          ├── Index.tsx
          ├── Create.tsx
          └── Edit.tsx
```

Inertia maps these as:

```php
Inertia::render('Welcome');
Inertia::render('Stocks/Index');
Inertia::render('Admin/Stocks/Create');
```
