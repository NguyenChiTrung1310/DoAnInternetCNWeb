---
name: laravel-migration
description: Use this skill when creating or modifying Laravel database migrations. Covers naming conventions, column types (especially money/decimal), foreign keys, indexes, soft deletes, and rollback safety. Triggers on tasks involving Schema::create, Schema::table, $table->id(), creating new tables, or modifying existing schema.
---

# Laravel Migration Best Practices

## Always Use These Conventions

### File Naming

- Format: `YYYY_MM_DD_HHMMSS_action_table_name.php`
- Action verbs: `create`, `add`, `remove`, `rename`, `modify`
- Examples:
  - `2025_05_15_120000_create_stocks_table.php`
  - `2025_05_15_130000_add_balance_to_users_table.php`
  - `2025_05_15_140000_modify_price_in_stocks_table.php`

### Class Structure

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('table_name', function (Blueprint $table) {
            // columns
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('table_name');
    }
};
```

## Column Type Reference

### IDs and Foreign Keys

```php
$table->id();                                    // Primary key (bigint unsigned auto-inc)
$table->foreignId('user_id')->constrained();     // FK to users.id, NOT NULL
$table->foreignId('user_id')->nullable()->constrained();  // Nullable FK
$table->foreignId('user_id')->constrained()->cascadeOnDelete();  // Delete cascade
$table->foreignId('user_id')->constrained()->restrictOnDelete(); // Block delete if used
```

### Strings

```php
$table->string('name');                          // VARCHAR(255)
$table->string('symbol', 10);                    // VARCHAR(10)
$table->text('description');                     // TEXT
$table->longText('content');                     // LONGTEXT
```

### Numbers

```php
// MONEY: ALWAYS DECIMAL, NEVER FLOAT
$table->decimal('balance', 15, 2);               // 15 digits total, 2 after decimal point
$table->decimal('price', 15, 2)->default(0);     // With default

// INTEGERS
$table->integer('quantity');                     // INT
$table->bigInteger('volume');                    // BIGINT
$table->unsignedInteger('age');                  // UNSIGNED INT
$table->tinyInteger('rating');                   // TINYINT
```

### Booleans, Enums, JSON

```php
$table->boolean('is_active')->default(true);
$table->enum('role', ['user', 'admin'])->default('user');
$table->enum('type', ['buy', 'sell']);
$table->json('metadata')->nullable();
```

### Dates

```php
$table->timestamp('executed_at')->nullable();
$table->date('birthday');
$table->dateTime('event_at');
$table->timestamps();                            // created_at + updated_at
$table->softDeletes();                           // deleted_at
```

## Index Strategy

### Always Index

1. Foreign keys (Laravel does this automatically when using `constrained()`)
2. Columns used in WHERE clauses frequently
3. Columns used in ORDER BY
4. Unique constraints

### Examples

```php
// Single column index
$table->index('email');

// Unique constraint
$table->string('email')->unique();
$table->string('symbol', 10)->unique();

// Composite index (for compound queries)
$table->index(['user_id', 'created_at']);       // Useful for "user's transactions over time"
$table->index(['stock_id', 'date']);            // Useful for "stock price on date"
```

## Common Mistakes to Avoid

### ❌ Float for money

```php
$table->float('balance');  // NEVER. Causes rounding errors.
```

### ✅ Decimal for money

```php
$table->decimal('balance', 15, 2);
```

### ❌ Missing foreign key constraint

```php
$table->unsignedBigInteger('user_id');  // No constraint
```

### ✅ Proper foreign key

```php
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
```

### ❌ No down() method

```php
public function down(): void
{
    // empty
}
```

### ✅ Proper rollback

```php
public function down(): void
{
    Schema::dropIfExists('table_name');
}
```

### ❌ Hardcoded enum values without comment

```php
$table->enum('status', ['p', 'c', 'x']);  // What do these mean?
```

### ✅ Self-documenting enums

```php
$table->enum('status', ['pending', 'completed', 'cancelled'])->default('pending');
```

## Foreign Key Cascade Strategy

| Behavior | When to Use | Example |
|----------|-------------|---------|
| `cascadeOnDelete()` | Child data is meaningless without parent | User → their Tokens, their Sessions |
| `restrictOnDelete()` | Parent shouldn't be deletable while children exist | Stock → Transactions (don't delete stock with history) |
| `nullOnDelete()` | Child can outlive parent (with reference cleared) | Post → Author (post survives if author deleted) |
| `noActionOnDelete()` | Manual handling | Rare; use case dependent |

## Migration Order

When migrations depend on each other:

1. `users` migration first (Laravel default)
2. Then independent tables (`stocks`)
3. Then tables with FKs (`transactions`, `portfolios`, `price_histories`)

Use timestamps in filenames to control order.

## Modifying Existing Migrations

### Never edit a migration that has been run on production

Always create a NEW migration:

```php
// 2025_05_15_120000_add_logo_url_to_stocks_table.php
public function up(): void
{
    Schema::table('stocks', function (Blueprint $table) {
        $table->string('logo_url')->nullable()->after('description');
    });
}

public function down(): void
{
    Schema::table('stocks', function (Blueprint $table) {
        $table->dropColumn('logo_url');
    });
}
```

## Testing Migrations

```bash
# Test full migration cycle works
php artisan migrate:fresh --seed

# Test rollback works
php artisan migrate:rollback

# Test rollback of specific number of migrations
php artisan migrate:rollback --step=3
```

If `migrate:fresh --seed` fails from a clean database, the migration is broken.

## Special: Money Columns Recap

This project is a stock trading system. **EVERY** money-related column MUST be:

```php
$table->decimal('column_name', 15, 2);
```

Columns this applies to:

- `users.balance`
- `stocks.current_price`
- `stocks.previous_close`
- `transactions.price`
- `transactions.total`
- `transactions.fee`
- `portfolios.avg_price`
- `price_histories.price`

NEVER float, NEVER int (cents), always DECIMAL.
