---
name: money-decimal-handling
description: Use this skill whenever code touches monetary values, prices, balances, fees, or any decimal arithmetic. Critical for this stock trading application where rounding errors cause real bugs. Covers DECIMAL column types, Laravel decimal casts, BCMath for precise arithmetic, and common float pitfalls. Triggers on any code involving balance, price, total, fee, or quantity * price calculations.
---

# Money & Decimal Handling

## The Golden Rule

**Never use float/double for money. Ever.**

## Why Float Fails

JavaScript and PHP both use IEEE 754 floating-point, which cannot represent decimal numbers exactly:

```javascript
0.1 + 0.2 === 0.3  // false
0.1 + 0.2          // 0.30000000000000004

(71500 * 100) === 7150000  // true (small)
(71500.5 * 100) === 7150050  // false (depends on values)
```

```php
var_dump(0.1 + 0.2 === 0.3);  // false
var_dump(0.1 + 0.2);           // float(0.3)
echo number_format(0.1 + 0.2, 20);  // 0.30000000000000004441
```

For a single $0.001 error per transaction, with 1 million transactions you have $1,000 missing. Not acceptable.

## Database: Use DECIMAL

Every money column in migrations:

```php
$table->decimal('balance', 15, 2);        // 15 total digits, 2 after decimal
$table->decimal('price', 15, 2);
$table->decimal('total', 15, 2);
$table->decimal('fee', 15, 2);
$table->decimal('avg_price', 15, 2);
```

**Specifications**:
- `15` = total digits including decimal — supports up to 9,999,999,999,999.99 (about 10 trillion VND, ample)
- `2` = digits after decimal — for VND we only need integer values, but 2 supports cents-equivalent for future use

**Storage**:
- DECIMAL is stored as a string in InnoDB
- Comparisons and arithmetic done in fixed-point inside MySQL
- No precision loss

## Model: Cast to decimal

```php
class User extends Model
{
    protected function casts(): array
    {
        return [
            'balance' => 'decimal:2',
        ];
    }
}

class Stock extends Model
{
    protected function casts(): array
    {
        return [
            'current_price' => 'decimal:2',
            'previous_close' => 'decimal:2',
        ];
    }
}

class Transaction extends Model
{
    protected function casts(): array
    {
        return [
            'price' => 'decimal:2',
            'total' => 'decimal:2',
            'fee' => 'decimal:2',
        ];
    }
}

class Portfolio extends Model
{
    protected function casts(): array
    {
        return [
            'avg_price' => 'decimal:2',
        ];
    }
}
```

**What `decimal:2` does**:
- Reads from database as string `"71500.00"`
- Casts to PHP string when accessed
- Sets to 2 decimal places consistently

## Arithmetic: Use BCMath

PHP's `bcmath` extension provides arbitrary-precision arithmetic on strings:

```php
// Addition
bcadd('71500.50', '1000.25', 2);  // "72500.75"

// Subtraction
bcsub('100000', '71500.50', 2);  // "28499.50"

// Multiplication
bcmul('71500', '100', 2);  // "7150000.00"

// Division
bcdiv('7150000', '100', 2);  // "71500.00"

// Comparison (returns -1, 0, or 1)
bccomp('71500.50', '71500.00', 2);  // 1 (first is greater)

// Modulo (PHP 7.2+)
bcmod('100', '3', 2);  // "1.00"

// Power
bcpow('2', '10', 2);  // "1024.00"
```

**All `bc*` functions require strings** (not floats) for precision.

## Common Operations

### Calculate transaction total

```php
// ❌ DANGEROUS: float
$total = $price * $quantity;
$fee = $total * 0.0015;
$grandTotal = $total + $fee;

// ✅ SAFE: bcmath
$total = bcmul((string) $price, (string) $quantity, 2);
$fee = bcmul($total, '0.0015', 2);
$grandTotal = bcadd($total, $fee, 2);
```

### Check sufficient balance

```php
// ❌ DANGEROUS
if ($user->balance >= $total) { ... }

// ✅ SAFE
if (bccomp($user->balance, $total, 2) >= 0) { ... }
```

### Calculate weighted average price

When buying additional shares of an existing position:

```php
// Formula:
// new_avg = (old_avg * old_qty + buy_price * buy_qty) / (old_qty + buy_qty)

// ❌ DANGEROUS
$newAvgPrice = (
    $portfolio->avg_price * $portfolio->quantity +
    $buyPrice * $buyQuantity
) / ($portfolio->quantity + $buyQuantity);

// ✅ SAFE
$oldCost = bcmul($portfolio->avg_price, (string) $portfolio->quantity, 2);
$newCost = bcmul($buyPrice, (string) $buyQuantity, 2);
$totalCost = bcadd($oldCost, $newCost, 2);
$totalQty = $portfolio->quantity + $buyQuantity;
$newAvgPrice = bcdiv($totalCost, (string) $totalQty, 2);
```

### Calculate profit/loss

```php
// market_value = quantity * current_price
$marketValue = bcmul((string) $portfolio->quantity, $stock->current_price, 2);

// cost_basis = avg_price * quantity
$costBasis = bcmul($portfolio->avg_price, (string) $portfolio->quantity, 2);

// pnl = market_value - cost_basis
$pnl = bcsub($marketValue, $costBasis, 2);

// pnl_percent = (pnl / cost_basis) * 100
$pnlPercent = bcmul(bcdiv($pnl, $costBasis, 6), '100', 2);
```

## Alternative: Integer Cents

For simpler arithmetic, store as integers in the smallest unit:

```php
// Instead of $71,500.50, store 7150050 (cents)
$priceInCents = 7150050;
$quantity = 100;
$totalInCents = $priceInCents * $quantity;  // 715005000
$totalInDong = $totalInCents / 100;  // 7150050.00
```

**Pros**:
- All math is integer (no precision issues)
- Fast comparisons

**Cons**:
- Need to convert when displaying
- Doesn't translate well to multiple decimal places

For this project (VND only, integer amounts in practice), the **decimal column + bcmath** approach is more straightforward and matches database semantics.

## Frontend (TypeScript)

### Receive money as strings

Laravel's decimal cast returns strings via Inertia. TypeScript types:

```typescript
export interface User {
  id: number;
  balance: number;  // After parsing
  // ...
}

export interface Stock {
  id: number;
  current_price: number;
  previous_close: number;
  // ...
}
```

When Laravel sends `"71500.00"` (string), parse on the client:

```typescript
const balance = parseFloat(user.balance);
```

Or better: define types as `string`:

```typescript
export interface Stock {
  current_price: string;  // "71500.00" as received
}
```

And format directly:

```typescript
formatCurrency(stock.current_price);  // Handles both string and number
```

### Formatting

```typescript
export function formatCurrency(amount: number | string): string {
  const value = typeof amount === 'string' ? parseFloat(amount) : amount;
  if (isNaN(value)) return '0 ₫';
  return new Intl.NumberFormat('vi-VN', {
    style: 'currency',
    currency: 'VND',
    maximumFractionDigits: 0,  // VND has no decimals
  }).format(value);
}

formatCurrency(71500);        // "71.500 ₫"
formatCurrency('71500.00');   // "71.500 ₫"
formatCurrency(0);            // "0 ₫"
```

### Avoid Frontend Calculations

Don't compute totals in TypeScript when the server does it:

```typescript
// ❌ Display calculation
const total = quantity * stock.current_price;
const fee = total * 0.0015;
const grandTotal = total + fee;
// (might be off-by-a-few VND from server)

// ✅ Send to server, server returns canonical total
post(route('trading.buy'), { quantity }, {
  onSuccess: () => toast.success('Đặt lệnh thành công'),
});
```

If you must show preview in the UI:

```typescript
// Preview total (acceptable for display only)
const previewTotal = Math.round(quantity * parseFloat(stock.current_price));
```

Use `Math.round()` to avoid floating-point display artifacts.

## Vietnamese Stock Specifics

### Lot Size

Vietnamese stock market trades in lots of 100 shares:

```php
// Form validation
'quantity' => ['required', 'integer', 'min:100', 'multiple_of:100'],
```

### Transaction Fee

Standard rate: 0.15% of transaction value.

```php
const TRANSACTION_FEE_RATE = '0.0015';

$fee = bcmul($total, self::TRANSACTION_FEE_RATE, 2);
```

### Currency Display

- Use VND (Vietnamese Dong)
- No decimal places (1 VND is the smallest unit in practice)
- Comma as thousand separator: `71.500 ₫` or `71,500₫`

## Common Bugs

### Bug 1: Off-by-cent total

```php
// User reports: "I bought 100 shares at 71,500 VND but total shows 7,150,000.5"

// Cause: float arithmetic
$total = $stock->current_price * $quantity;  // 71500.0 * 100 = 7150000.0 (but stored as float)
// When cast to decimal: maybe shows 7150000.00 or 7150000.01

// Fix: bcmath
$total = bcmul($stock->current_price, (string) $quantity, 2);
```

### Bug 2: Balance check passes when it shouldn't

```php
// User has 99,999.99 in balance, wants to buy at 100,000.00 total
if ($user->balance >= $total) { ... }
// Might pass due to float weirdness

// Fix
if (bccomp($user->balance, $total, 2) >= 0) { ... }
```

### Bug 3: Frontend total != backend total

```typescript
// Frontend shows: 7,150,000.50 ₫
// User clicks Buy
// Backend deducts: 7,150,000.51 ₫
// User confused

// Fix: Always show server-calculated values, not client-calculated
```

### Bug 4: Sum doesn't equal total

```php
$transactions = $user->transactions()->where('type', 'buy')->get();
$sum = $transactions->sum('total');
// Sum might differ from manual calculation due to float casting in Collection::sum()

// Fix: Use database aggregate
$sum = $user->transactions()
    ->where('type', 'buy')
    ->sum('total');  // Computed in MySQL, returns string-safe decimal
```

## Quick Reference

| Operation | Float (WRONG) | BCMath (RIGHT) |
|-----------|---------------|----------------|
| `$a + $b` | `$a + $b` | `bcadd($a, $b, 2)` |
| `$a - $b` | `$a - $b` | `bcsub($a, $b, 2)` |
| `$a * $b` | `$a * $b` | `bcmul($a, $b, 2)` |
| `$a / $b` | `$a / $b` | `bcdiv($a, $b, 2)` |
| `$a > $b` | `$a > $b` | `bccomp($a, $b, 2) > 0` |
| `$a == $b` | `$a == $b` | `bccomp($a, $b, 2) === 0` |
| `$a >= $b` | `$a >= $b` | `bccomp($a, $b, 2) >= 0` |

## Mental Checklist Before Writing Money Code

1. Is this a money/price/balance/fee value?
2. If yes:
   - Database column: `DECIMAL(15, 2)` ✓
   - Model cast: `'decimal:2'` ✓
   - PHP arithmetic: `bc*` functions ✓
   - Comparison: `bccomp()` ✓
   - Display: `formatCurrency()` helper ✓
3. If no (e.g., quantity, count):
   - Database column: `INTEGER`
   - PHP type: `int`
   - Normal arithmetic OK

## Final Reminder

This is a stock trading application. **Every** money bug is a real bug that breaks user trust. Take the extra 5 seconds to use `bcmul()` instead of `*`. It's worth it.