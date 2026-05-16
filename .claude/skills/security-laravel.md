---
name: security-laravel
description: Use this skill when writing any code that handles user input, authentication, authorization, database queries, file uploads, or external API calls. Covers OWASP Top 10 mitigations specific to Laravel + React stacks. Triggers on tasks involving forms, controllers, middleware, models with $fillable, or anything user-facing.
---

# Security Guidelines for Laravel + Inertia + React

## OWASP Top 10 Mitigations

### 1. Broken Access Control

**Threat**: User accesses resources they shouldn't (e.g., editing another user's data).

**Mitigations**:

- Use middleware for route-level checks
- Use Policies for model-level checks
- NEVER trust IDs from the client; always verify ownership

```php
// BAD: Trusts user-provided ID
public function update(Request $request, $id) {
    $portfolio = Portfolio::find($id);
    $portfolio->update($request->all());  // Anyone can edit anyone's portfolio!
}

// GOOD: Verifies ownership
public function update(Request $request, Portfolio $portfolio) {
    if ($portfolio->user_id !== auth()->id()) {
        abort(403);
    }
    $portfolio->update($request->validated());
}

// BETTER: Use Policy
public function update(UpdatePortfolioRequest $request, Portfolio $portfolio) {
    $this->authorize('update', $portfolio);
    $portfolio->update($request->validated());
}
```

### 2. Cryptographic Failures

**Threat**: Sensitive data exposed (passwords, tokens).

**Mitigations**:

- Use bcrypt/argon2 for passwords (Laravel default)
- Never store plain-text passwords, API keys, or sensitive PII
- Use HTTPS in production (`SESSION_SECURE_COOKIE=true`)
- Don't log sensitive data

```php
// User model
protected $hidden = ['password', 'remember_token'];

// NEVER do this
Log::info('User login', ['email' => $email, 'password' => $password]);
```

### 3. Injection (SQL, Command, etc.)

**Threat**: User input causes unintended database/system commands.

**Mitigations**:

- Use Eloquent or Query Builder (auto-escapes)
- Never concatenate user input into queries
- Validate inputs with Form Requests

```php
// BAD: SQL injection risk
$users = DB::select("SELECT * FROM users WHERE email = '{$email}'");

// GOOD: Parameterized query
$users = DB::select("SELECT * FROM users WHERE email = ?", [$email]);

// BEST: Eloquent
$users = User::where('email', $email)->get();
```

### 4. Insecure Design

**Mitigations**:

- Apply business logic in Services, not Controllers
- Use database transactions for multi-step operations
- Validate state before actions (e.g., check `is_active` before login)

### 5. Security Misconfiguration

**Mitigations**:

- `APP_DEBUG=false` in production
- `APP_ENV=production` in production
- Disable debug pages like Telescope/Debugbar in production
- Set proper file permissions (755 for dirs, 644 for files, 600 for `.env`)
- Don't commit `.env`

### 6. Vulnerable Components

**Mitigations**:

- Run `composer audit` regularly
- Run `npm audit` regularly
- Use Dependabot/Renovate
- Pin major versions in `composer.json` and `package.json`
- Avoid packages with < 100 downloads/week unless absolutely necessary
- Check package's GitHub: when was last commit? open issues? license?

### 7. Identification & Authentication Failures

**Mitigations**:

- Use Laravel Breeze/Fortify (not custom auth)
- Rate limit login attempts
- Require strong passwords (min 8 chars, mixed case, numbers)
- Implement password reset securely (signed URLs)
- Don't reveal whether email exists in login errors

```php
// In RouteServiceProvider or routes file
Route::middleware(['throttle:6,1'])->group(function () {
    Route::post('/login', [LoginController::class, 'store']);
});
```

### 8. Software & Data Integrity Failures

**Mitigations**:

- Verify dependency integrity (use lock files)
- Don't deserialize untrusted data
- Validate uploads (file type, size, content)

```php
// Form Request for file uploads
public function rules(): array
{
    return [
        'logo' => 'nullable|image|mimes:jpeg,png,webp|max:2048',  // 2MB max
    ];
}
```

### 9. Security Logging & Monitoring Failures

**Mitigations**:

- Log authentication attempts (success/failure)
- Log authorization failures
- Log critical business events (e.g., big trades)
- Use Laravel Log channels properly

```php
Log::channel('security')->warning('Failed login attempt', [
    'email' => $email,
    'ip' => $request->ip(),
    'user_agent' => $request->userAgent(),
]);
```

### 10. Server-Side Request Forgery (SSRF)

**Mitigations**:

- Don't fetch user-provided URLs without allowlist
- Validate and parse URLs before making requests

## Project-Specific Security Rules

### Input Validation (MANDATORY for every form/endpoint)

```php
// Form Request example
class PlaceOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user() !== null
            && $this->user()->is_active;
    }

    public function rules(): array
    {
        return [
            'quantity' => [
                'required',
                'integer',
                'min:100',
                'max:1000000',
                'multiple_of:100',  // Vietnamese stock lot size
            ],
        ];
    }

    public function messages(): array
    {
        return [
            'quantity.required' => 'Vui lòng nhập số lượng',
            'quantity.integer' => 'Số lượng phải là số nguyên',
            'quantity.min' => 'Số lượng tối thiểu là 100 cổ phiếu',
            'quantity.multiple_of' => 'Số lượng phải là bội số của 100',
        ];
    }
}
```

### Mass Assignment Protection

```php
// Always define $fillable on every Model
class Stock extends Model
{
    protected $fillable = [
        'symbol',
        'company_name',
        // ... explicitly list fields
    ];

    // NEVER use $guarded = []; it's a security hole
}
```

### Hide Sensitive Fields

```php
class User extends Authenticatable
{
    protected $hidden = [
        'password',
        'remember_token',
    ];
}
```

### Authorization in Controllers

```php
// Pattern: Authorize before action
public function update(Stock $stock, UpdateStockRequest $request)
{
    // Already authorized by middleware (admin only)
    // Already validated by Form Request

    $stock->update($request->validated());

    return back()->with('success', 'Cập nhật thành công');
}
```

### CSRF Protection

Laravel includes CSRF by default. With Inertia, Axios handles this automatically. Just don't disable it:

```php
// In bootstrap/app.php — do NOT add routes to validateCsrfTokens
$middleware->validateCsrfTokens(except: [
    // Only for webhooks, etc. Not for internal forms
]);
```

### Rate Limiting

```php
// routes/auth.php (Breeze)
Route::middleware(['throttle:6,1'])->group(function () {
    Route::post('login', [AuthenticatedSessionController::class, 'store']);
});

// Or in controllers
public function __construct()
{
    $this->middleware('throttle:10,1')->only(['store', 'update']);
}
```

### Password Requirements

```php
// In RegisterRequest or similar
'password' => [
    'required',
    'confirmed',
    Rules\Password::min(8)
        ->letters()
        ->mixedCase()
        ->numbers()
        ->symbols()
        ->uncompromised(),  // Checks haveibeenpwned database
],
```

### XSS Prevention (React)

React auto-escapes content. The ONLY exception is `dangerouslySetInnerHTML`. NEVER use it with user content.

```tsx
// SAFE: React escapes the content
<div>{userInput}</div>

// DANGEROUS: Bypasses escaping
<div dangerouslySetInnerHTML={{ __html: userInput }} />  // XSS risk
```

If you MUST render user HTML (e.g., rich text), sanitize first:

```bash
npm install dompurify
```

```tsx
import DOMPurify from 'dompurify';

<div dangerouslySetInnerHTML={{
  __html: DOMPurify.sanitize(userInput)
}} />
```

### Don't Trust Inertia Props from Client

Even though Inertia "feels" like SPA, server should never trust the client.

```php
// BAD: Trusting balance from frontend
public function buy(Request $request) {
    $balance = $request->input('balance');  // User can fake this!
    // ...
}

// GOOD: Always fetch from DB
public function buy(PlaceOrderRequest $request) {
    $user = $request->user();
    $currentBalance = $user->balance;  // From DB, not request
    // ...
}
```

### Money Calculations

Use Laravel's decimal handling, never floats:

```php
// Model
protected $casts = [
    'balance' => 'decimal:2',
    'price' => 'decimal:2',
];

// Calculations: convert to integer cents for precision if doing math
$cents = (int) ($price * 100);
// Or use bcmath for arbitrary precision:
$result = bcmul($price, $quantity, 2);  // 2 decimal places
```

## Pre-commit Security Checklist

Before every commit, verify:

- [ ] No `.env`, `.env.local` in commit
- [ ] No hardcoded passwords, API keys, tokens
- [ ] No `dd()`, `dump()`, `console.log(sensitiveData)` left in
- [ ] All Form Requests validate inputs
- [ ] All Models have `$fillable` defined
- [ ] All Controllers use Form Requests (not `$request->all()`)
- [ ] No `dangerouslySetInnerHTML` without DOMPurify
- [ ] No raw SQL queries with user input
- [ ] No `$guarded = []` in Models
- [ ] No debug routes exposed

## Production Hardening Checklist

Before deploying:

- [ ] `APP_DEBUG=false`
- [ ] `APP_ENV=production`
- [ ] `SESSION_SECURE_COOKIE=true`
- [ ] HTTPS only (force redirect from HTTP)
- [ ] Run `php artisan config:cache`
- [ ] Run `php artisan route:cache`
- [ ] Run `php artisan view:cache`
- [ ] Database backups configured
- [ ] Error logging configured (Sentry, Bugsnag, or Laravel Telescope private)
- [ ] Rate limiting on all public endpoints
- [ ] CSP headers configured
- [ ] Security headers (X-Frame-Options, X-Content-Type-Options, etc.)
