---
name: laravel-money
description: Use this skill whenever writing or editing Laravel code that involves monetary values, prices, balances, fees, percentages of money, exchange rates, or any financial calculation. Triggers include any mention of money, currency, price, cost, fee, balance, amount, payment, invoice, transaction, or financial math in a Laravel context.
---

# Laravel Money Handling with Brick

## Core Rule

**NEVER use PHP floats, doubles, or native arithmetic (`+`, `-`, `*`, `/`) for any monetary value.** Floating point arithmetic causes rounding errors that are unacceptable in financial contexts.

```php
// ❌ NEVER DO THIS
$total = $price * $quantity;
$fee = $amount * 0.015;
$balance = $credit - $debit;
```

Use the **Brick** packages instead:

- `brick/math` — arbitrary-precision arithmetic via `BigDecimal`
- `brick/money` — money and currency value objects built on `brick/math`

Prefer `brick/money` when working with known currencies (it handles scale and rounding per currency). Use `brick/math` directly when you need raw precision without currency context (e.g., exchange rates, ratios, intermediate calculations).

## Installation

```bash
# For currency-aware money handling (recommended for most apps)
composer require brick/money

# For raw arbitrary-precision math only
composer require brick/math
```

> `brick/money` depends on `brick/math`, so installing `brick/money` gives you both.

## Choosing Your Approach

### Option A: `brick/money` (Recommended for most apps)

Best when you're working with known currencies. The `Money` class encapsulates both the amount and currency, and automatically applies the correct scale (e.g., 2 decimals for USD, 0 for JPY).

```php
use Brick\Money\Money;

$price = Money::of('29.99', 'USD');
$total = $price->multipliedBy(3);
$discount = $total->minus(Money::of('10.00', 'USD'));

// Currency mismatch throws an exception — prevents bugs
$usd = Money::of('100', 'USD');
$eur = Money::of('50', 'EUR');
// $usd->plus($eur); // ❌ MoneyMismatchException
```

### Option B: `brick/math` BigDecimal

Best for currency-agnostic math, exchange rate calculations, or when you manage scale yourself.

```php
use Brick\Math\BigDecimal;
use Brick\Math\RoundingMode;

$total = BigDecimal::of($price)->multipliedBy($quantity);
$fee = BigDecimal::of($amount)->multipliedBy('0.025')->toScale(2, RoundingMode::HALF_DOWN);
$balance = BigDecimal::of($credit)->minus($debit);
```

## Storing Money in the Database

Choose the strategy that fits your project. **Never use `FLOAT` or `DOUBLE` columns for money.**

### Strategy 1: String columns
Stores the exact decimal representation. Maximum flexibility, no precision loss.
```php
$table->string('amount');
```

### Strategy 2: Decimal columns
Traditional approach. Works well with SQL aggregations.
```php
$table->decimal('amount', 19, 4);
```

### Strategy 3: Integer (smallest unit / cents)
Stores money as cents or the smallest currency unit. Simple, fast, no decimal issues.
```php
$table->bigInteger('amount_cents');
```

## Model Accessors / Mutators

### Using `brick/money` with string storage

```php
use Brick\Money\Money;
use Illuminate\Database\Eloquent\Casts\Attribute;

class Order extends Model
{
    protected function total(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => Money::of($value, $this->currency),
            set: fn (Money|string $value) => $value instanceof Money
                ? $value->getAmount()->__toString()
                : (string) $value,
        );
    }
}
```

### Using `brick/math` with string storage

```php
use Brick\Math\BigDecimal;
use Illuminate\Database\Eloquent\Casts\Attribute;

class Transaction extends Model
{
    protected function amount(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => BigDecimal::of($value),
            set: fn (string $value) => (string) BigDecimal::of($value),
        );
    }
}
```

### Using integer (cents) storage

```php
use Brick\Money\Money;
use Illuminate\Database\Eloquent\Casts\Attribute;

class Payment extends Model
{
    protected function amount(): Attribute
    {
        return Attribute::make(
            get: fn (int $value) => Money::ofMinor($value, $this->currency),
            set: fn (Money $value) => $value->getMinorAmount()->toInt(),
        );
    }
}
```

### Reusable Trait for Multiple Money Fields

```php
// app/Concerns/HasMoneyAttributes.php
namespace App\Concerns;

use Brick\Math\BigDecimal;
use Illuminate\Database\Eloquent\Casts\Attribute;

trait HasMoneyAttributes
{
    protected function moneyAttribute(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => BigDecimal::of($value),
            set: fn (string $value) => (string) BigDecimal::of($value),
        );
    }
}
```

```php
use App\Concerns\HasMoneyAttributes;

class Transaction extends Model
{
    use HasMoneyAttributes;

    protected function amount(): Attribute
    {
        return $this->moneyAttribute();
    }

    protected function fee(): Attribute
    {
        return $this->moneyAttribute();
    }
}
```

### Handling Nullable Money Columns

When a money column is nullable, use `?string` to avoid `TypeError` on null values:

```php
use Brick\Math\BigDecimal;
use Illuminate\Database\Eloquent\Casts\Attribute;

class Invoice extends Model
{
    protected function discount(): Attribute
    {
        return Attribute::make(
            get: fn (?string $value) => $value !== null ? BigDecimal::of($value) : null,
            set: fn (?string $value) => $value !== null ? (string) BigDecimal::of($value) : null,
        );
    }
}
```

For the reusable trait, add a nullable variant:

```php
trait HasMoneyAttributes
{
    protected function moneyAttribute(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => BigDecimal::of($value),
            set: fn (string $value) => (string) BigDecimal::of($value),
        );
    }

    protected function nullableMoneyAttribute(): Attribute
    {
        return Attribute::make(
            get: fn (?string $value) => $value !== null ? BigDecimal::of($value) : null,
            set: fn (?string $value) => $value !== null ? (string) BigDecimal::of($value) : null,
        );
    }
}
```

## Common Operations

### With `brick/money`

```php
use Brick\Money\Money;

// Basic arithmetic
$subtotal = Money::of('49.99', 'USD');
$tax = $subtotal->multipliedBy('0.08', RoundingMode::HALF_DOWN);
$total = $subtotal->plus($tax);

// Splitting (e.g., dividing a bill)
$perPerson = $total->dividedBy(3, RoundingMode::HALF_DOWN);

// Allocating (distributes remainder across parts — no money lost)
[$share1, $share2, $share3] = $total->split(3);

// Comparison
if ($balance->isGreaterThanOrEqualTo(Money::of('100', 'USD'))) {
    // sufficient funds
}

// Sum a collection
$total = $payments->reduce(
    fn (Money $carry, $payment) => $carry->plus($payment->amount),
    Money::of(0, 'USD')
);
```

### With `brick/math`

```php
use Brick\Math\BigDecimal;
use Brick\Math\RoundingMode;

// Addition
$total = BigDecimal::of($a)->plus(BigDecimal::of($b));

// Subtraction
$remaining = BigDecimal::of($balance)->minus(BigDecimal::of($withdrawal));

// Multiplication (e.g., quantity × unit price)
$lineTotal = BigDecimal::of($unitPrice)->multipliedBy($quantity);

// Division (always specify scale and rounding)
$split = BigDecimal::of($total)->dividedBy(3, 2, RoundingMode::HALF_DOWN);

// Percentage / fee calculation
$fee = BigDecimal::of($amount)->multipliedBy('0.025')->toScale(2, RoundingMode::HALF_DOWN);

// Comparison
if ($balance->isGreaterThanOrEqualTo($withdrawalAmount)) {
    // allow withdrawal
}

// Sum a collection
$total = $transactions->reduce(
    fn (BigDecimal $carry, $tx) => $carry->plus(BigDecimal::of($tx->amount)),
    BigDecimal::zero()
);
```

## Rounding Rules

- Default to `RoundingMode::HALF_DOWN` for general monetary rounding (rounds toward zero at the exact midpoint).
- Always specify scale (decimal places) explicitly when using `BigDecimal`.
- `brick/money` handles scale automatically based on currency (e.g., 2 for USD, 0 for JPY, 3 for KWD).

```php
// BigDecimal: always specify scale
$rounded = BigDecimal::of($value)->toScale(2, RoundingMode::HALF_DOWN);

// Money: scale is automatic
$price = Money::of('19.999', 'USD', roundingMode: RoundingMode::HALF_DOWN); // → $20.00
```

## Formatting for Display

```php
// brick/money — use the built-in formatter
use Brick\Money\Money;

$price = Money::of('1234.50', 'USD');
echo $price; // "USD 1234.50"

// For locale-aware formatting with brick/money
$formatted = (new \Brick\Money\Formatter\IntlMoneyFormatter(
    new \NumberFormatter('en_US', \NumberFormatter::CURRENCY)
))->format($price); // "$1,234.50"

// BigDecimal — convert to string, format at the display layer only
$displayValue = $amount->toScale(2, RoundingMode::HALF_DOWN)->__toString();
$formatted = Number::currency((float) $displayValue, 'USD');
```

## Validation Rules

When validating monetary input in form requests:
```php
public function rules(): array
{
    return [
        'amount' => ['required', 'numeric', 'min:0.01'],
        'currency' => ['required', 'string', 'size:3'],
    ];
}
```
Then immediately convert to a Money/BigDecimal object in the controller or service layer:
```php
// With brick/money
$amount = Money::of($request->validated('amount'), $request->validated('currency'));

// With brick/math
$amount = BigDecimal::of($request->validated('amount'));
```

## JSON / API Responses

Always serialize monetary values as **strings** in JSON to preserve precision:
```php
// With brick/money
return response()->json([
    'total'    => $total->getAmount()->__toString(),
    'currency' => $total->getCurrency()->getCurrencyCode(),
]);

// With brick/math
return response()->json([
    'balance' => $balance->toScale(2, RoundingMode::HALF_DOWN)->__toString(),
    'fee'     => $fee->toScale(2, RoundingMode::HALF_DOWN)->__toString(),
]);
```

## Testing

### Asserting Money Values with PHPUnit

```php
use Brick\Math\BigDecimal;
use Brick\Money\Money;

// Assert BigDecimal values
$result = $service->calculateFee(BigDecimal::of('1000'));
$this->assertTrue(
    $result->isEqualTo(BigDecimal::of('25.00')),
    "Expected fee of 25.00, got {$result}"
);

// Assert Money values
$total = $service->calculateTotal($order);
$this->assertTrue(
    $total->isEqualTo(Money::of('149.99', 'USD')),
    "Expected $149.99, got {$total}"
);

// Assert comparisons
$this->assertTrue($balance->isGreaterThan(BigDecimal::zero()));
$this->assertTrue($fee->isLessThanOrEqualTo(BigDecimal::of('100')));
```

### Custom Assertion Trait

```php
// tests/Concerns/MoneyAssertions.php
namespace Tests\Concerns;

use Brick\Math\BigDecimal;
use Brick\Money\Money;

trait MoneyAssertions
{
    protected function assertMoneyEquals(string $expected, Money $actual, string $currency = 'USD'): void
    {
        $this->assertTrue(
            $actual->isEqualTo(Money::of($expected, $currency)),
            "Failed asserting that {$actual} equals {$expected} {$currency}"
        );
    }

    protected function assertBigDecimalEquals(string $expected, BigDecimal $actual): void
    {
        $this->assertTrue(
            $actual->isEqualTo(BigDecimal::of($expected)),
            "Failed asserting that {$actual} equals {$expected}"
        );
    }
}
```

### Factory Patterns

```php
class TransactionFactory extends Factory
{
    public function definition(): array
    {
        return [
            'amount' => (string) BigDecimal::of($this->faker->randomFloat(2, 1, 10000)),
            'currency' => $this->faker->randomElement(['USD', 'EUR', 'GBP']),
        ];
    }
}
```

## Crypto & High-Precision Amounts (Optional)

When working with cryptocurrency or other high-precision values, use `BigDecimal` with higher scale:

```php
// Crypto: use 8+ decimal places depending on the asset
$btcAmount = BigDecimal::of($value)->toScale(8, RoundingMode::HALF_DOWN);
$ethAmount = BigDecimal::of($value)->toScale(18, RoundingMode::DOWN);

// Exchange rate calculations: use BigDecimal, not Money
$rate = BigDecimal::of('42350.75');
$btcValue = BigDecimal::of($usdAmount)->dividedBy($rate, 8, RoundingMode::HALF_DOWN);

// Store crypto amounts as strings — decimal columns lack sufficient precision
$table->string('crypto_amount');
```

## What To Watch For

1. **Never pass Money/BigDecimal directly to Eloquent `create()`/`update()`.** Cast to string first: `(string) $amount` or `$money->getAmount()->__toString()`.
2. **Never compare money with `==` or `>` on raw values.** Use `isEqualTo()`, `isGreaterThan()`, etc.
3. **Never do math in Blade templates.** All calculations happen in services/models — Blade only displays.
4. **Aggregations in MySQL** (`SUM`, `AVG`) return floats. Wrap the result in `BigDecimal::of()` or `Money::of()` immediately.
5. **Be explicit about scale** on every `BigDecimal` division and `toScale()` call. Omitting it can cause `RoundingNecessaryException`.
6. **Never mix currencies.** `brick/money` enforces this with exceptions. With `BigDecimal`, you must enforce it yourself.
7. **Use `split()` not `dividedBy()`** when distributing money among parties — `split()` handles remainder distribution so no cents are lost.
