[⬅ Back to articles](../README.md)


# "1,299" became €1.30: one separator, no locale, and a parser that guessed

> A real-world case: a price-comparison aggregator that ingests merchant feeds (CSV and JSON) from about forty shops across Germany, the US, Switzerland, France and India. A single `parsePrice()` helper turned every price string into cents. It had one heuristic: *"if the string has both `,` and `.`, the rightmost one is the decimal separator."* It worked for a year. Then a merchant started shipping whole-euro prices without cents — and 4,000 products in our catalogue went on sale for about a euro.

## Symptom

The product page said **€1.30**. The merchant's feed said **`1,299`**. The merchant's own site said **€1,299.00**.

The pattern in the bad rows was so regular that it read like a spec:

```text
raw_price   parsed_minor   shown     merchant_site
1,299       130            €1.30     €1,299.00
2,499       250            €2.50     €2,499.00
1.299       130            €1.30     €1,299.00
12,999      1300           €13.00    €12,999.00
```

Four properties, and they give away the root cause:

1. **Only prices of 1,000 and above were wrong.** Nothing under `999` moved. That's a magnitude boundary, not a data-quality issue.
2. **The error was always exactly ÷1000** (then rounded to cents). Not "a bit off" — three orders of magnitude, every time.
3. **Prices with cents were fine.** `1.299,00` and `1,299.00` parsed perfectly. Only strings **without decimals** broke.
4. **Both separators were affected** — comma and dot, in different feeds. So this was not "the German locale is broken". It was "the string never said which format it was in".

Nobody could reproduce it by pasting `1,299` into the console, because `1,299` on its own is not a price. It is a string that *could* be a price in two different conventions.

## When it appears

The helper, in full:

```ts
// price.ts — the heuristic that "handles both formats"
export function parsePrice(raw: string): number {
  const clean = raw.replace(/[^\d.,]/g, ""); // drop €, $, spaces, letters
  const lastComma = clean.lastIndexOf(",");
  const lastDot = clean.lastIndexOf(".");

  // the rightmost separator is the decimal one
  if (lastComma > lastDot) {
    return Math.round(Number(clean.replace(/\./g, "").replace(",", ".")) * 100);
  }
  return Math.round(Number(clean.replace(/,/g, "")) * 100);
}
```

The trigger combination:

1. **The string contains exactly one separator, followed by exactly three digits** — `1,299`, `1.299`, `12,999`.
2. **The parser infers the format from the string itself**, because the feed does not say which one it uses.
3. **The feed omits cents for whole amounts.** Cents are the thing that disambiguates: `1,299.00` has two separators, so the heuristic has something to work with.
4. **The currency has two decimals**, so `1.299` cannot legitimately mean "one and 299 thousandths" — but the parser has no access to that rule at that point.

Take away any one of the four and nothing breaks. That's why it shipped: `1,299.00` is what every fixture in the test suite looked like.

## The mechanics of the bug

### The string does not contain the answer

This is the part that took a while to accept: `1,299` is genuinely ambiguous, and no amount of cleverness resolves it. The same seven characters mean two different things depending on who wrote them:

| Raw string | Read as `en-US` | Read as `de-DE` |
| :--- | ---: | ---: |
| `1,299` | 1,299.00 | 1.299 → 1.30 |
| `1.299` | 1.299 → 1.30 | 1,299.00 |
| `1 299` | 1,299.00 (space is always grouping) | 1,299.00 |
| `1,299.00` | 1,299.00 | 1,299.00 |
| `1.299,00` | 1,299.00 | 1,299.00 |

Rows four and five are the ones the heuristic handled, and they are the only rows where the answer was in the string. The first two are the ones it guessed at: a price without a fractional part carries **no evidence** of its locale — it is a number with a thousands separator, and that is all. So the helper wasn't slightly wrong; it was asked a question the input cannot answer, and answered anyway. Note what the table actually says: under `de-DE`, `1,299` really *is* 1.30. The output was correct for a locale the feed never claimed to use.

### Why three digits is the tell

A group separator is followed by exactly three digits. A decimal separator is followed by one or two (or three, for KWD/BHD/JOD/TND). The overlap is precisely `\d{3}$`:

```js
"1,299".match(/\d+([.,])\d{3}$/);  // match → could be grouping, could be 3 decimals
"1,29.99".match(/\d+([.,])\d{2}$/); // no match at the tail → the last separator is decimal
"1,29,900".match(/\d+([.,])\d{3}$/); // match, but this is valid en-IN grouping
```

That is the whole rule, and it is a *detector*, not a parser: three digits after the only separator means "I cannot know". `1,29,900` shows why the detector needs the locale too — with `en-IN` declared, that string is perfectly ordinary.

### The guess arrives disguised as a valid number

The heuristic's output went straight into `Number()`, which never complains:

```js
Number("1.299");         // 1.299  → ×100 → 130 cents
Math.round(1.299 * 100); // 130    → €1.30
parseFloat("1,299");     // 1      → ×100 → 100 cents → €1.00
```

Valid, positive, well-formed cents. Nothing downstream can tell them apart from a real €1.30 product — which is why the alert came from merchandising ("a competitor is undercutting us on 4,000 SKUs"), not from the ingestion job.

And it compounded: the "cheapest offer" widget ranks by parsed price, so the broken rows became the **lowest price in every category** and the comparison page promoted them. Wrong money doesn't sit in a database, it gets sorted, ranked and recommended.

The new merchant's export simply trimmed trailing zeros — `1299.00` became `1299`, and their German shop sent `1.299`. A year of cents-heavy feeds had never touched that branch: *one separator, three digits, no decimals*.

## Why the tests were green

- Every fixture had cents: `"1,299.00"`, `"9,99"`, `"€ 1.234,56"`. The ambiguous shape `"1,299"` never appeared once.
- The assertions compared the helper's output with a number produced by the same helper. The test asserted the bug was consistent with itself.
- The only locale in the suite came from the browser: tests ran under `en-US`, so the comma branch was only ever exercised by two-separator strings.
- The data-validation layer checked `price_minor > 0`. `130` passes. It always passes.

A table test for this class of bug is four lines long:

```ts
const CASES = [
  ["1,299.00", "en-US", 129900],
  ["1.299,00", "de-DE", 129900],
  ["1,299",    "en-US", 129900], // ← returned 130
  ["1.299",    "de-DE", 129900], // ← returned 130
  ["1,29,900", "en-IN", 12990000],
] as const;
```

Two of those five fail against the heuristic, and they fail in the same direction — which is exactly the shape of the production incident.

## How we found it

Reproduce, then compare readings:

```js
parsePrice("1,299");  // 130
parsePrice("1,299.00"); // 129900  ← the same price, with cents
```

One line, two answers for one amount. Then the confirmation on real data — the ratio between what we stored and what the merchant's own site showed:

```sql
-- a ÷1000 bug is loud in the ratio, invisible in the error log
SELECT merchant_id,
       count(*) FILTER (WHERE parsed_minor < source_minor / 100) AS suspicious,
       round(percentile_cont(0.5) WITHIN GROUP (
         ORDER BY parsed_minor::numeric / source_minor
       ), 4) AS median_ratio
FROM ingest_prices
WHERE ingested_at > now() - interval '7 days'
GROUP BY merchant_id
ORDER BY 3 ASC;
```

The new merchant's `median_ratio` was `0.0010`, to four decimals, across 4,000 rows. Everyone else's was `1.0000`. Cause confirmed, not guessed.

The control experiment that settled it: re-run the same feed with the separator taken from the merchant's declared locale instead of from the string. Same rows, same numbers, zero outliers.

## The fix

The rule: **a price string is not a price. It is a claim about a number, in a convention that has to come from somewhere else.**

### 1. Move the ambiguity out of the parser and into the feed contract

The durable fix is not a better heuristic — it's not needing one. Ask for integers and a scale:

```json
{
  "sku": "A-1183",
  "currency": "EUR",
  "amount_minor": 129900,
  "scale": 2
}
```

`amount_minor: 129900` cannot be misread. Every feed that can only produce human-formatted strings gets a **per-merchant config** (`locale`, or explicitly `decimal_separator` / `group_separator`), stored next to the credentials, because it is a property of the sender, not of the message:

```ts
type FeedConfig = {
  locale: string;   // "de-DE", "en-US", "en-IN", "fr-CH"
  currency: string; // "EUR"
  decimalCents: boolean; // do their amounts include a fractional part?
};
```

### 2. Parse with the declared locale, never the declared string

Separators come from `Intl`, so they are right for `fr-CH` (apostrophe), `fr-FR` (narrow no-break space) and `en-IN` (two-digit groups) without a table of your own:

```ts
function separators(locale: string) {
  const parts = new Intl.NumberFormat(locale).formatToParts(1234567.8);
  return {
    group: parts.find((p) => p.type === "group")?.value ?? "",
    decimal: parts.find((p) => p.type === "decimal")?.value ?? ".",
  };
}
```

### 3. Return "ambiguous" instead of picking an answer

The parser's job is to produce a number **or a refusal**. Making the refusal an explicit state is what stops the next heuristic from being written by someone in a hurry:

```ts
const SCALE: Record<string, number> = { EUR: 2, USD: 2, CHF: 2, GBP: 2, JPY: 0, KWD: 3 };

type Price =
  | { ok: true; minor: number }
  | { ok: false; reason: "ambiguous"; readings: number[] }
  | { ok: false; reason: "unparseable" };

export function parsePrice(raw: string, locale: string, currency: string): Price {
  const { group, decimal } = separators(locale);
  const scale = SCALE[currency];

  // 1. strip symbols, ISO codes, apostrophes and every kind of space
  let s = raw.replace(/[^\d.,\s'-]/g, "").replace(/[\s']/g, "");

  // 2. Swiss-style "no cents": 1'299.–  and  1.299,-
  s = s.replace(/[.,][–-]$/, `${decimal}${"0".repeat(scale)}`);

  // 3. the config says one thing, the string says another: three digits that
  //    are not the declared group separator, in a currency with two decimals
  const tail = s.match(/\d+([.,])\d{3}$/);
  if (tail && tail[1] !== group && scale !== 3) {
    return {
      ok: false,
      reason: "ambiguous",
      readings: [asGrouped(s, scale), asDecimal(s, scale)], // 129900 | 130
    };
  }

  // 4. now the reading is determined by the declared locale
  if (group) s = s.split(group).join("");
  const [whole, frac = ""] = s.split(decimal);
  if (!/^\d*$/.test(whole) || !/^\d*$/.test(frac)) return { ok: false, reason: "unparseable" };

  const digits = (frac.replace(/\D/g, "") + "0".repeat(scale)).slice(0, scale);
  const minor = Number(whole || "0") * 10 ** scale + Number(digits);
  return { ok: true, minor };
}
```

Two details worth keeping: the ambiguity check fires **before** the group separator is stripped (after stripping there is nothing left to inspect), and the `readings` array goes straight into the quarantine record so a human sees *both* candidate prices instead of a mystery. The two helpers it calls exist only for that branch, so they live underneath:

```ts
// both candidate readings, so a human can decide — integers only, no floats
const asGrouped = (s: string, scale: number) => Number(s.replace(/\D/g, "")) * 10 ** scale;

const asDecimal = (s: string, scale: number) => {
  const [whole, frac = ""] = s.split(/[.,]/);
  const padded = (frac + "0".repeat(scale + 1)).slice(0, scale + 1); // one spare digit to round with
  const base = Number(whole) * 10 ** scale + Number(padded.slice(0, scale));
  return base + (Number(padded[scale]) >= 5 ? 1 : 0);
};

asGrouped("1.299", 2); // 129900
asDecimal("1.299", 2); // 130
```

The check deliberately fires only when the separator disagrees with the declared group separator. `1,299` under `en-US` is not ambiguous — the config says the comma groups, so it reads as 1,299.00. `1.299` under `en-US` **is** ambiguous, because a dot in front of three digits is either a 3-decimal amount (impossible for a 2-decimal currency) or a European thousand separator the config never mentioned. And `KWD` is exempt: there, three decimals are legal.

### 4. Quarantine, don't fail, and don't guess

An ambiguous row is not a reason to fail the whole ingestion, and it is definitely not a reason to take the cheaper reading. It goes to a holding table with the raw string and both readings, and merchandising resolves it by opening the merchant's page:

```ts
if (!parsed.ok) {
  await quarantine.insert({
    merchantId,
    rawPrice: row.price,
    reason: parsed.reason,
    readings: parsed.reason === "ambiguous" ? parsed.readings : [],
  });
  return; // the SKU keeps its last known good price
}
```

That last comment is the important behaviour: a product with a stale price is a mild problem. A product listed at €1.30 is a financial and reputational one, and the ranking widget will amplify it.

### 5. Assert the round trip at ingestion

The cheap net that catches everything the detector misses is a **digit fingerprint**: format the parsed cents back, strip all non-digits, compare with the digits of the source string.

```ts
const digits = (s: string) => s.replace(/\D/g, "");

// display only: Intl formats, it never computes
const formatMinor = (minor: number, currency: string, locale: string) =>
  new Intl.NumberFormat(locale, { style: "currency", currency })
    .format(minor / 10 ** SCALE[currency]);

const back = formatMinor(parsed.minor, currency, locale); // "1.299,00"
if (digits(back) !== digits(raw)) throw new Error(`round trip failed: ${raw} → ${back}`);
```

With the old helper, `1,299` parsed to `130` and formatted back as `1.30`: digits `130` against `1299`. The gate would have stopped the very first bad row. Its false positives (Swiss `1'299.–`, whose source carries no cents at all) are not noise — they are the list of formats your parser does not actually support yet, which is exactly what you want to learn on day one rather than on the day merchandising calls.

## Neighbouring traps in the same family

- **`1299` with no separator at all is also ambiguous** — €12.99 in a cents-feed, €1,299 in a units-feed. Only the feed contract can tell you which; a parser that "just multiplies by 100" picks the wrong one half the time.
- **Some feeds use `,` for the group separator in the JSON and `.` in the CSV.** Same merchant, two exports, two conventions. Per-endpoint config, not per-merchant.
- **`1,29,900` breaks every "three digits means grouping" assumption.** `en-IN` groups by two after the first three digits; a regex written for European formats mangles it.
- **Currency scales are not always two.** `JPY` has 0, `KWD`/`BHD`/`JOD`/`TND` have 3 — so `1.299` is a perfectly valid KWD amount and the ambiguity branch above must not fire for it.

---

## Takeaways (checklist)

1.  **`1,299` is not decidable from the string.** Comma-grouping and decimal-comma produce different readings and both are legal; only the sender's convention can settle it.
2.  **Three digits after the only separator is the ambiguity signature.** Detect it (`\d+[.,]\d{3}$`), don't resolve it.
3.  **Never infer the locale from the string.** It is a property of the feed, configured per merchant or per endpoint — not of the browser, and not of `navigator.language`.
4.  **A parser returns a number or a refusal.** Add `{ ok: false, reason: "ambiguous", readings }` to the type so the next developer can't quietly add a `?? 0`.
5.  **Quarantine ambiguous rows and keep the last known good price.** A stale price is an annoyance; a €1.30 price is a loss.
6.  **Round-trip every parsed amount and compare digit fingerprints.** `1,299 → 1.30 → 130 ≠ 1299` catches the whole class, including the formats you haven't thought about yet.
7.  **The durable fix is integers plus a scale in the feed.** `amount_minor: 129900` has no locale, no separator and no ambiguity — ask for it before you write a cleverer regex.
8.  **Check the distribution, not the error log.** A ÷1000 bug logs nothing; it shows up as `median_ratio = 0.0010` for one merchant next to `1.0000` for everyone else.


[⬅ Back to articles](../README.md)
