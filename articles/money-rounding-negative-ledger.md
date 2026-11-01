[⬅ Back to articles](../README.md)


# The cent that never came back: `Math.round` is not symmetric, and the ledger knows

> A real-world case: an internal payments back office (Vite + React on the front, a Node service and a Postgres ledger behind it). Merchants can issue full and partial refunds; the UI computes the refund amount and posts it. After two quarters the daily reconciliation report started showing a discrepancy against the PSP. Small — a few cents — but it only ever went **one way**: our ledger always claimed slightly more money than the provider did. And it grew.

## Symptom

The reconciliation job compares, per currency and per day, the sum of movements in our ledger with the sum reported by the payment provider. Normally the delta is `0.00`. Then:

```text
2025-11-03  EUR  ledger=48,120.55  psp=48,120.54  delta=+0.01
2025-11-04  EUR  ledger=51,003.10  psp=51,002.98  delta=+0.12
2025-11-05  EUR  ledger=39,884.02  psp=39,883.61  delta=+0.41
```

Three properties of that table are the entire diagnosis:

1.  **The delta is always positive.** Our books always hold *more* than the provider's. A race condition or a lost message would produce both signs. A systematic bias means a systematic calculation.
2.  **It scales with refund volume.** Days with more partial refunds drift more. Days with only captures are clean.
3.  **It's always a whole number of cents.** Nothing is "off by a float"; something is being *rounded*, and rounded in one direction.

Nobody could reproduce it by hand. Every individual refund looked correct in the UI: `12.50` in, `12.50` shown, `12.50` in the confirmation.

## When it appears

One helper, used by both directions of money movement:

```ts
// money.ts — the only place that touches cents
export const toMinor = (amount: number) => Math.round(amount * 100);
export const fromMinor = (minor: number) => (minor / 100).toFixed(2);
```

Charges call `toMinor(price)`. Refunds call `toMinor(-refundAmount)` — because the ledger stores refunds as negative movements. That single minus sign is the bug.

The trigger combination:

1.  **Money crosses the float boundary** — `amount * 100` on a decimal value;
2.  **The value lands exactly on a half cent** — `x.xx5`, which is common for percentage-based partial refunds (50% of `0.01`, 33% of `0.15`, VAT splits);
3.  **The sign is negative** for refunds;
4.  **`Math.round` is used as the rounding function.**

## The mechanics of the bug

Two independent traps stack on top of each other. Either one alone is survivable; together they produce a silent, one-directional leak.

### Trap 1: `Math.round` breaks ties toward `+∞`, not away from zero

This is not a browser quirk — it's specified, identical in every engine:

```js
Math.round( 1.5); //  2
Math.round(-1.5); // -1   ← not -2
Math.round( 0.5); //  1
Math.round(-0.5); // -0   ← not -1
Math.round( 2.5); //  3
Math.round(-2.5); // -2   ← not -3
```

For positive numbers "round half up" looks like "round away from zero". For negative numbers it's the opposite: the half always moves **toward zero**, i.e. toward the customer's detriment — but only in the refund direction. So the same helper is asymmetric:

| Operation | Exact value | `Math.round` result | Money moved |
| :--- | ---: | ---: | ---: |
| Capture | `+0.005` | `+1` cent | we take 1 cent |
| Refund | `-0.005` | `-0` cents | we return 0 cents |

Charge and refund of the same amount do **not** cancel. Every half-cent refund leaves one cent in our ledger that the provider has already paid out. That's the positive delta in the report, and its growth rate is exactly "number of half-cent refunds".

### Trap 2: `amount * 100` is not a multiplication, it's a rounding decision

`1.005` cannot be represented in binary floating point. It is stored as `1.00499999999999989341858963598497211933135986328125`. Multiply by 100 and you get `100.49999999999999`, so even a "correct" round-half-up gives `100` — one cent short of the intended `1.01`:

```js
1.005 * 100;              // 100.49999999999999
Math.round(1.005 * 100);  // 100   → 1.00, not 1.01
(1.005).toFixed(2);       // "1.00"  → and toFixed is formatting, not rounding
(2.675).toFixed(2);       // "2.67"  → the classic
```

`toFixed` is the third leg of the trap: it *formats* a binary float to N decimals, and the value it formats is already wrong. It never "fixes" the number, it just hides the error at two decimals — which is why the UI looked perfect while the ledger drifted. Display and computation were both fed from the same corrupted float, so they agreed with each other and disagreed with reality.

### Why the tests were green

- Every fixture used **positive** amounts, so the `Math.round` asymmetry never fired.
- Assertions used `toBeCloseTo(..., 2)`, which is exactly the tolerance that hides a one-cent error.
- The refund tests compared the UI text with the posted payload — both produced by the same buggy helper. The test asserted that the bug was consistent with itself.

## How we found it

Three console lines closed a two-week investigation:

```js
Math.round(-0.5);        // -0  → refund of half a cent moves nothing
Math.round(0.5);         //  1  → capture of half a cent moves one cent
Math.round(1.005 * 100); // 100 → 1.005 is not 1.005
```

Then the confirmation on real data — group the day's movements by direction and compare the count of half-cent values with the delta:

```sql
-- the delta should equal the number of half-cent refunds
SELECT count(*) FILTER (WHERE amount_minor % 10 = 5 AND amount_minor < 0) AS half_cent_refunds,
       sum(amount_minor) AS ledger_delta
FROM movements
WHERE booked_at::date = '2025-11-05';
```

It matched to the cent. The control experiment: switch the refund path to the same rounding used by the provider (half away from zero, computed in minor units) and replay the day — delta `0.00`. Cause confirmed, not guessed.

## The fix

The real fix is not a better `Math.round`. It's a rule: **money never exists as a float.**

### 1. Integers in minor units, end to end

Store, transport and compute in integer minor units (`bigint` or `number` below `2^53`). The database column is `BIGINT`, the API field is `amount_minor`, the React state is a number of cents. No `* 100` anywhere in the business logic.

```ts
type Minor = number; // 1250 === €12.50

const capture = (amount: Minor) => amount;
const refund = (amount: Minor) => -amount; // no rounding: it's already exact
```

### 2. Convert at the edges, with an explicit scale

The scale is a property of the currency, not a constant. `* 100` is wrong for JPY (0 decimals) and for KWD/BHD/JOD/TND (3 decimals):

```ts
const CURRENCY_SCALE: Record<string, number> = {
  EUR: 2, USD: 2, GBP: 2,
  JPY: 0, KRW: 0,
  KWD: 3, BHD: 3, JOD: 3, TND: 3,
};
```

Parse the string form, never the float form:

```ts
// "1 234,56" style input → 123456 minor units, no float in between
function parseAmount(input: string, currency: string): number {
  const scale = CURRENCY_SCALE[currency];
  const clean = input.replace(/[\s_]/g, "").replace(",", ".");
  const negative = clean.startsWith("-");
  const [whole, frac = ""] = clean.replace(/^[+-]/, "").split(".");
  const digits = (frac + "0".repeat(scale)).slice(0, scale);
  const minor = Number(whole || "0") * 10 ** scale + Number(digits || "0");
  return negative ? -minor : minor;
}
```

### 3. If a float ever has to be rounded, make the mode explicit and symmetric

Sometimes a float arrives from outside — a percentage, a tax rate, a provider API. Then the rounding function must say what it does, and it must behave identically for both signs:

```ts
// half away from zero: symmetric for + and -
// (the common convention for money; check what your provider uses)
function roundHalfAwayFromZero(value: number): number {
  const abs = Math.abs(value);
  return Math.sign(value) * Math.round(abs);
}

roundHalfAwayFromZero(0.5);  //  1
roundHalfAwayFromZero(-0.5); // -1  ← symmetric, unlike Math.round
```

But be honest with yourself: this helper still receives a float, so `1.005` is still `1.00499…`. For anything that must be exact, round a **decimal** — `decimal.js`, `big.js`, or integer arithmetic on the already-parsed minor units — not a `number`. A rounding helper is a patch over the float boundary, not a replacement for removing it.

### 4. Round once, and record the remainder

When one amount is split across several line items — installments, split payments, fee allocation — rounding each part independently loses or invents cents. Allocate in minor units and give the remainder to the last (or largest) part:

```ts
// split 100.00 across 3 parts without losing a cent
function allocate(totalMinor: number, weights: number[]): number[] {
  const sum = weights.reduce((a, b) => a + b, 0);
  const parts = weights.map((w) => Math.floor((totalMinor * w) / sum));
  let rest = totalMinor - parts.reduce((a, b) => a + b, 0);
  for (let i = 0; rest > 0; i = (i + 1) % parts.length) {
    parts[i] += 1;
    rest -= 1;
  }
  return parts;
}

allocate(10000, [1, 1, 1]); // [3334, 3333, 3333] — sums to 10000, always
```

The invariant to assert in tests: `parts.reduce((a, b) => a + b) === total`, for every input. That single property test would have caught the original bug class in a minute.

## Neighbouring traps in the same ledger

- **`Intl.NumberFormat` is for display, never for math.** Its rounding mode is configurable in recent engines and defaults differ from your provider's. Format the final minor units, don't parse formatted output back.
- **Idempotency keys must be generated per operation, not per attempt.** A key created inside the retry loop produces a second charge when the first one times out. Generate it once, before the first attempt, and persist it with the intent.
- **`JSON.parse` turns amounts into floats.** If the API sends `"1234567890123456789"` as a JSON number, it is already wrong before your code runs. Transport money as a string or as minor units, and use `bigint` beyond `2^53`.
- **Timezones date your money.** `new Date("2025-11-03")` is UTC midnight; `.toLocaleDateString()` in a negative-offset timezone renders "Nov 2". Settlement dates and report cut-offs must be plain dates (`YYYY-MM-DD` strings), never `Date` objects converted for display.

---

## Takeaways (checklist)

1.  **`Math.round` breaks ties toward `+∞`.** `Math.round(-1.5) === -1`, `Math.round(0.5) === 1`. Used on signed amounts, it is asymmetric: charges round away from zero, refunds round toward it.
2.  **A one-directional reconciliation delta is a rounding bug, not a race.** Lost messages and duplicate writes produce both signs; systematic bias produces one.
3.  **`amount * 100` and `toFixed` both operate on a value that is already wrong.** `1.005 * 100 === 100.49999999999999`, `(1.005).toFixed(2) === "1.00"`.
4.  **Money is an integer in minor units — in the DB, the API and the UI state.** Convert only at the edges, and never let a float into the ledger.
5.  **The scale is per currency:** 0 for JPY/KRW, 2 for EUR/USD, 3 for KWD/BHD/JOD/TND. `* 100` is a hardcoded assumption, not a rule.
6.  **If rounding is unavoidable, name the mode and make it symmetric** — and test it with negative values. Every money helper needs a `-x.xx5` test case.
7.  **Round once, allocate with a remainder, and assert the sum.** `parts.sum === total` as a property test is the cheapest insurance a ledger can buy.


[⬅ Back to articles](../README.md)
