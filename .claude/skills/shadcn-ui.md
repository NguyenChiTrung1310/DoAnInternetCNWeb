---
name: shadcn-ui
description: Use this skill when building UI components, pages, or layouts that need shadcn/ui components. Covers component selection, customization, theming with CSS variables, dark mode strategy, and common composition patterns. Triggers on tasks involving forms, dialogs, tables, dropdowns, or any visual UI work.
---

# shadcn/ui Best Practices

## What shadcn/ui Is (and Isn't)

**Is**: A collection of accessible, customizable React components built on Radix UI primitives and styled with Tailwind. You install components into YOUR codebase (not a dependency).

**Isn't**: A UI library you import from `node_modules`. You own the code.

## Installation Pattern

```bash
# Initial setup (once per project)
npx shadcn@latest init

# Add components as needed
npx shadcn@latest add button input dialog table card
npx shadcn@latest add form select dropdown-menu

# List all available
npx shadcn@latest add
```

Components install to `resources/js/components/ui/`.

## Required Components

```bash
npx shadcn@latest add button input label dialog dropdown-menu \
  table card select badge avatar separator tabs form \
  toast alert sheet skeleton command popover
```

## Theming

### CSS Variables (in `resources/css/app.css`)

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;

    --primary: 221.2 83.2% 53.3%;        /* Blue, finance-friendly */
    --primary-foreground: 210 40% 98%;

    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;

    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;

    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;

    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;

    --radius: 0.5rem;

    /* Project-specific: stock trading colors */
    --price-up: 142 71% 45%;       /* Green */
    --price-down: 0 84% 60%;       /* Red */
    --price-flat: 215 20% 65%;     /* Gray */
  }
}
```

## Common Patterns

### Form (with React Hook Form + Zod) - DON'T USE THIS

For this project, use **Inertia `useForm`** instead of React Hook Form. shadcn/ui forms are agnostic, just use them with Inertia's hook:

```tsx
import { useForm } from '@inertiajs/react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';

export default function MyForm() {
  const { data, setData, post, processing, errors } = useForm({
    email: '',
    password: '',
  });

  return (
    <form onSubmit={(e) => { e.preventDefault(); post(route('login')); }}>
      <div className="space-y-2">
        <Label htmlFor="email">Email</Label>
        <Input
          id="email"
          type="email"
          value={data.email}
          onChange={(e) => setData('email', e.target.value)}
          disabled={processing}
          aria-invalid={!!errors.email}
        />
        {errors.email && <p className="text-sm text-destructive">{errors.email}</p>}
      </div>

      <Button type="submit" disabled={processing}>
        {processing ? 'Đang đăng nhập...' : 'Đăng nhập'}
      </Button>
    </form>
  );
}
```

### Dialog (Modal)

```tsx
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';

export function DeleteConfirmDialog({ onConfirm }: { onConfirm: () => void }) {
  return (
    <Dialog>
      <DialogTrigger asChild>
        <Button variant="destructive" size="sm">Xóa</Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Xác nhận xóa</DialogTitle>
          <DialogDescription>
            Hành động này không thể hoàn tác. Bạn có chắc chắn?
          </DialogDescription>
        </DialogHeader>
        <DialogFooter>
          <Button variant="outline">Hủy</Button>
          <Button variant="destructive" onClick={onConfirm}>Xóa</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

### Table

```tsx
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';

<Table>
  <TableHeader>
    <TableRow>
      <TableHead>Mã CK</TableHead>
      <TableHead>Tên công ty</TableHead>
      <TableHead className="text-right">Giá hiện tại</TableHead>
    </TableRow>
  </TableHeader>
  <TableBody>
    {stocks.map((stock) => (
      <TableRow key={stock.id}>
        <TableCell className="font-medium">{stock.symbol}</TableCell>
        <TableCell>{stock.company_name}</TableCell>
        <TableCell className="text-right">
          {formatCurrency(stock.current_price)}
        </TableCell>
      </TableRow>
    ))}
  </TableBody>
</Table>
```

### Dropdown Menu

```tsx
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';

<DropdownMenu>
  <DropdownMenuTrigger asChild>
    <Button variant="ghost">{user.name}</Button>
  </DropdownMenuTrigger>
  <DropdownMenuContent align="end">
    <DropdownMenuLabel>Tài khoản</DropdownMenuLabel>
    <DropdownMenuSeparator />
    <DropdownMenuItem asChild>
      <Link href={route('profile.edit')}>Hồ sơ</Link>
    </DropdownMenuItem>
    <DropdownMenuItem asChild>
      <Link href={route('logout')} method="post" as="button">
        Đăng xuất
      </Link>
    </DropdownMenuItem>
  </DropdownMenuContent>
</DropdownMenu>
```

## Composition Patterns

### Custom Button Variants

To add custom variants beyond default:

```tsx
// components/ui/button.tsx (extend the existing one)
const buttonVariants = cva(
  // base classes
  {
    variants: {
      variant: {
        default: '...',
        // Add:
        success: 'bg-green-500 text-white hover:bg-green-600',
        warning: 'bg-yellow-500 text-white hover:bg-yellow-600',
      },
    },
  }
);
```

### Reusable Form Field

```tsx
// components/shared/form-field.tsx
interface FormFieldProps {
  label: string;
  error?: string;
  required?: boolean;
  children: React.ReactNode;
}

export function FormField({ label, error, required, children }: FormFieldProps) {
  return (
    <div className="space-y-2">
      <Label>
        {label}
        {required && <span className="text-destructive ml-1">*</span>}
      </Label>
      {children}
      {error && <p className="text-sm text-destructive">{error}</p>}
    </div>
  );
}
```

## Common Mistakes

### ❌ Adding components you might use

```bash
# BAD: bloat
npx shadcn@latest add accordion alert-dialog aspect-ratio badge breadcrumb \
  calendar carousel chart checkbox collapsible command context-menu \
  drawer dropdown-menu hover-card menubar navigation-menu pagination \
  popover progress radio-group resizable scroll-area select separator \
  sheet skeleton slider sonner switch table tabs textarea toast \
  toggle toggle-group tooltip typography
```

```bash
# GOOD: install only what you need now
npx shadcn@latest add button input dialog
# Add more later as needed
```

### ❌ Modifying base components extensively

shadcn/ui components are designed to be customized but in a controlled way. If you find yourself rewriting a component, you probably want to compose, not modify.

### ❌ Mixing styling systems

```tsx
// BAD: Tailwind + inline styles
<Button style={{ marginTop: '10px' }} className="bg-blue-500">

// GOOD: Tailwind only
<Button className="mt-2.5 bg-blue-500">
```

### ❌ Not using `cn()` utility

```tsx
// BAD: Manual conditional
<div className={`p-4 ${isActive ? 'bg-blue-500' : 'bg-gray-100'}`}>

// GOOD: cn() handles conflicts
import { cn } from '@/lib/utils';
<div className={cn('p-4', isActive ? 'bg-blue-500' : 'bg-gray-100')}>
```

## Accessibility Quick Wins

- Always use semantic HTML (`<button>` not `<div onClick>`)
- Add `aria-label` to icon-only buttons
- Provide `alt` text for images
- Ensure focus is visible (`focus-visible:` modifiers)
- Color isn't the only indicator (add text/icons alongside)

```tsx
// GOOD
<Button variant="ghost" size="icon" aria-label="Xóa">
  <Trash2 className="h-4 w-4" />
</Button>
```

## Responsive Design

Mobile-first approach:

```tsx
// Base = mobile, md/lg = larger screens
<div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
  {/* Cards */}
</div>
```

Container width:

```tsx
<div className="container mx-auto px-4 py-6">
  {/* content */}
</div>
```

Breakpoints (Tailwind defaults):

- `sm`: 640px
- `md`: 768px
- `lg`: 1024px
- `xl`: 1280px
- `2xl`: 1536px
