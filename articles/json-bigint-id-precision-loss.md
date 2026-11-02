[⬅ Back to articles](../README.md)

# The order that belonged to someone else: `JSON.parse` and integers above 2^53

> A real-world case: an internal marketplace back office (Vite + React, a Node API, Postgres behind it). Support opens an order from the "recent orders" list and gets **a different order** — someone else's customer, someone else's amount. The database is fine. The API response on the wire is fine. The bug lives entirely inside the browser, in the half-millisecond it takes `JSON.parse` to read the response.

## Symptom

Two adjacent rows in the table, same ID printed twice:

```text
#   Order ID               Customer        Total
1   1234567890123456800    ACME GmbH       €1,240.00
2   1234567890123456800    Nordwind AG     €89.00
```

Click row 2 — row 1 opens. Select both rows and hit "refund" — the same order is refunded twice. The `key` prop in React prints a duplicate-key warning to the console, and that warning is the only clue anyone notices for weeks.

What makes it nasty:

- **The server is right.** The JSON body literally contains `"id":1234567890123456789` and `"id":1234567890123456788`. Two different values. The browser shows one.
- **It only happens to big IDs.** Old seeded orders with small IDs work forever. New orders from a Snowflake generator break from day one, so QA fixtures never reproduce it.
- **It survives the network tab.** Chrome's preview of the response is also parsed by the same engine, so it shows the rounded number too. The raw response text is the only honest view.
- **Nothing throws.** No exception, no rejected promise, no 500. Just the wrong row.

## Where it starts

The front end does the most ordinary thing in the world:

```ts
type Order = { id: number; customer: string; totalMinor: number };

const orders: Order[] = await fetch("/api/orders").then((r) => r.json());
```

And the ID in that response is:

```json
{ "id": 1234567890123456789, "customer": "ACME GmbH", "totalMinor": 124000 }
```

The type says `number`, TypeScript is happy, the value is a `number`. It's just not *that* number.

## Why the engine does this

First fact about JavaScript numbers: there is only one numeric type, and it is a **binary floating-point double** (IEEE 754 binary64). Every `number` — `1`, `0.1`, `1234567890123456789` — is stored as a sign, an 11-bit exponent and a **53-bit significand**. That gives you exactly 2^53 (9,007,199,254,740,992) distinct integer steps, and only the ones up to

```js
Number.MAX_SAFE_INTEGER; // 9007199254740991  (2^53 - 1)
```

are guaranteed to survive a round trip. "Safe" is the spec's word, not a marketing one.

Second fact: JSON has no such limit. The grammar says a number is a sequence of digits — the parser is what decides how to store it. Every JavaScript engine stores it as a double, so **parsing is a lossy conversion you never asked for**:

| Value | Gap between representable integers |
| :--- | ---: |
| up to 2^53 ≈ 9.0 × 10^15 | 1 |
| 2^53 … 2^54 | 2 |
| 2^54 … 2^55 | 4 |
| **2^60 … 2^61 ≈ 1.15 × 10^18** | **256** |
| 2^62 … 2^63 | 1024 |

The gap doubles at every power of two. A 64-bit database ID commonly lands around `1.5 × 10^18` — the Snowflake range used by Discord, Twitter and a hundred internal services — where the gap is **256**. Which means 256 consecutive IDs all collapse into **one** double. Not "slightly wrong": indistinguishable.

```js
Number("1234567890123456789") === Number("1234567890123456788");
// true — two different orders, one number

Number("1234567890123456789"); // 1234567890123456800
Number.isSafeInteger(1234567890123456789); // false
```

And the same loss happens on the way **out**, because the number was already wrong before it was stringified:

```js
const body = '{"id":1234567890123456789}';
JSON.stringify(JSON.parse(body));
// '{"id":1234567890123456800}'  ← a round trip mutates the ID
```

If that rounded ID is then used in `PATCH /orders/:id`, `DELETE /orders/:id`, a cache key, or a `Map`, you are addressing an ID that does not exist — or, when the gap swallowed a neighbour, an ID that belongs to a different order.

## The sneakiest part: the console agrees with the bug

Print the parsed ID and the exact double next to each other:

```js
const id = JSON.parse('{"id":1234567890123456789}').id;

id;                          // 1234567890123456800  ← shortest string that round-trips
BigInt(id).toString();       // "1234567890123456768" ← what the double actually holds
```

Both lines describe the *same* double, and neither shows the ID you sent. Chrome DevTools and Node's `console.log` print the shortest decimal string that re-parses to that double — a convenience that hides the exact stored value. So the log "looks plausible": nineteen digits, starts correctly, no error. You compare it with the API response, see `...789` vs `...800`, and conclude "the server sent something else".

The JSON preview in DevTools is worse: it re-serializes the parsed object, so *both* sides of your comparison have already passed through the same rounding. The raw text (`view-source`, `curl`, "Copy response body") is the only artifact that still knows the truth.

## How we found it

Three console lines ended a two-week hunt:

```js
const raw = '{"id":1234567890123456789}';

JSON.parse(raw).id;              // 1234567890123456800
BigInt(JSON.parse(raw).id);      // 1234567890123456768n
JSON.stringify(JSON.parse(raw));  // '{"id":1234567890123456800}'
```

Then the confirmation on real data: pull the two suspected orders straight from Postgres and compare their IDs.

```sql
SELECT id, customer FROM orders WHERE id IN (1234567890123456789, 1234567890123456788);
-- 1234567890123456789 | ACME GmbH
-- 1234567890123456788 | Nordwind AG   ← two apart in Postgres, identical in the browser
```

The database had two rows. The UI rendered one number twice. Cause confirmed, not guessed.

And a quick reproduction anyone can run: fetch the same list twice, once through `r.json()` and once through `r.text()`.

```js
const raw = await fetch("/api/orders").then((r) => r.text());
const parsed = JSON.parse(raw);

console.log(raw.includes("1234567890123456789")); // true  — the wire is honest
console.log(parsed[0].id === parsed[1].id);       // true  — the browser is not
```

## The fix

The real fix is not a better parser. It's a rule: **an identifier is a string, not a number.** The digits are data, not arithmetic.

### 1. Make the API contract say `string`

IDs cross the wire as JSON strings, always:

```json
{ "id": "1234567890123456789", "customer": "ACME GmbH" }
```

```yaml
# OpenAPI
id:
  type: string
  pattern: "^[0-9]+$"
```

This is not exotic. Protobuf's JSON mapping already serializes `int64` as a string. GraphQL's `ID` scalar is serialized as `String` — precisely because a JavaScript client sits on the other end. Twitter's API shipped `id_str` next to `id` for exactly this reason in 2010. If you own both ends, just do it from the start; if you don't, that `*_str` field is your escape hatch.

### 2. Keep the string all the way through the UI

No conversion at any layer — not for a `Map` key, not for a React `key`, not for a URL, not for `localStorage`:

```tsx
type OrderId = string; // see step 3 — the point is to never convert it back to number

function OrderRow({ order }: { order: Order }) {
  // fine: strings work as keys, Map keys, query keys, URL segments
  return <Link to={`/orders/${encodeURIComponent(order.id)}`}>{order.customer}</Link>;
}
```

The tempting one-liner `Number(order.id)` or `parseInt(order.id, 10)` is the *entire* bug reintroduced by hand. Treat it like `eval`.

### 3. Put the type system on guard duty

A plain `string` loses the distinction the moment someone adds `id: string | number`. Brand it:

```ts
declare const brand: unique symbol;

export type OrderId = string & { readonly [brand]: "OrderId" };

export function asOrderId(value: string): OrderId {
  if (!/^[0-9]{1,19}$/.test(value)) throw new Error(`Not an order id: ${value}`);
  return value as OrderId;
}
```

Now `Number(id)` still compiles (JavaScript allows it), but the return type is plain `number` and can't be passed where an `OrderId` is expected — so the conversion has to be written down, reviewed, and justified. One ESLint rule (`no-restricted-syntax` on `Number(<OrderId>)`) closes the rest.

### 4. If the server sends numbers anyway, don't parse them with `JSON.parse`

Once `JSON.parse` has run, the digits are gone — a `reviver` receives the already-rounded `number`, so it can *detect* the loss but never recover it:

```js
JSON.parse(raw, (key, value) => {
  if (typeof value === "number" && !Number.isSafeInteger(value)) {
    console.warn(`Unsafe integer in "${key}" — precision already lost`, value);
  }
  return value;
});
```

The only real options are outside the built-in parser:

- **Ask for strings** (option 1) — always the cheapest.
- **Use a lossless parser** — `lossless-json` or `json-bigint`, which keep big integers as `BigInt` or as the original text so you can decide per field.
- **Normalize at the edge** — a thin backend/BFF layer that reads BIGINT as text and emits `"id": "..."`. Node's `pg` already returns `int8` columns as strings by default for this exact reason; `JSON.stringify`ing them as-is is correct, and mapping them through `Number` is how teams undo that protection.

### 5. Keep "safe" numbers safe — and check them at the boundary

Not every number should be a string: counts, percentages, durations and money in minor units are genuinely arithmetic and fit under 2^53 (9 quadrillion cents is 90 trillion euros). Assert that where data enters the app:

```ts
function assertSafeInteger(value: unknown, field: string): number {
  if (typeof value !== "number" || !Number.isSafeInteger(value)) {
    throw new TypeError(`${field} is not a safe integer: ${String(value)}`);
  }
  return value;
}

assertSafeInteger(1_240_00, "totalMinor");          // ok
assertSafeInteger(1234567890123456789, "orderId");  // throws — loudly, at the edge
```

That last line is the test that was missing: a fixture with a real production-shaped ID would have failed instantly, instead of passing with `id: 1`.

### 6. Compare and sort IDs as big integers, not as text

String IDs are opaque. `"9" > "10"` is `true` lexicographically, so "newest first" sorted by ID silently breaks as soon as IDs grow a digit — unless they are fixed-width or compared numerically:

```ts
const byId = (a: string, b: string) => (BigInt(a) < BigInt(b) ? -1 : BigInt(a) > BigInt(b) ? 1 : 0);
```

Better yet, sort by a real `created_at` and stop deriving order from IDs at all.

## Neighbouring traps in the same codebase

- **`localStorage` and `structuredClone` preserve the damage, not the truth.** If an ID ever became a `number`, storing it keeps the rounded value; no amount of later stringifying brings the digits back.
- **`ObjectId` from MongoDB is fine; `BIGSERIAL` / `BIGINT` / Snowflake are not.** Check the column type, not the framework. Prisma hands you `bigint` for `BigInt` columns — `JSON.stringify` then throws on it, which is an annoying but honest failure; convert with `.toString()`, never `Number()`.
- **Cache keys, `Map` keys and React `key`s inherit the collision.** Two orders that collapsed into one double are one entry in your normalized store, one row in the virtualized list, one node reused by React.
- **The same thing eats 64-bit hashes and nanosecond timestamps.** Any value from an external system in `1e16`–`1e19` deserves a `BigInt` or a string, even if it looks like "just a number".
- **`Date.now()` and `performance.now()` are safe** (milliseconds are ~1.7 × 10^12). Microsecond and nanosecond timestamps are not: `1.7 × 10^18` nanoseconds is past the safe range and will drift by hundreds of nanoseconds when parsed.

---

## Takeaways (checklist)

1.  **JavaScript has one number type: a 53-bit-significand double.** Integers above `Number.MAX_SAFE_INTEGER` (2^53 − 1) are not all representable, and `JSON.parse` silently rounds them.
2.  **The gap grows with the value.** At Snowflake scale (~1.5 × 10^18) it is 256, so 256 consecutive IDs collapse into a single `number`. Different orders become the same key.
3.  **A JSON round trip mutates the ID.** `JSON.stringify(JSON.parse('{"id":1234567890123456789}'))` returns `...800`; the value was already lost on the way in.
4.  **The console shows the rounded value too** — and so does DevTools' JSON preview, because both re-serialize the parsed object. Only the raw response text (`text()`, `curl`) is trustworthy.
5.  **IDs are strings.** Declare them as `string` in the API schema (or `id_str` alongside a legacy numeric field), keep them as strings in state, keys, URLs and storage, and brand the type so `Number(id)` has to be written deliberately.
6.  **After `JSON.parse` the digits are unrecoverable.** A reviver can only warn; the fix is either strings on the wire or a lossless parser (`lossless-json`, `json-bigint`).
7.  **Reserve real numbers for arithmetic, and assert `Number.isSafeInteger` at every boundary.** A test fixture with a production-shaped 19-digit ID would have caught this before it shipped - and would have caught the duplicate React key on the spot.

[⬅ Back to articles](../README.md)
