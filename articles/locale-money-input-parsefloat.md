[⬅ Back to articles](../README.md)


# €1.234,56 became €1,23: `parseFloat` and the locale that ate three digits

> A real-world case: a billing back office (Vite + React), desktop. Finance teams create custom invoices: pick a client, type the amount, confirm. The amount field is pre-filled with the client's previous invoice, formatted for their locale — `1.234,56` in Germany, `1,234.56` in the US. Some invoices came out exactly 1000× too small. No error, no warning, no failed request. The first symptom was a revenue report three weeks later.

## Symptom

An invoice for **€1.234,56** was created as **€1,23**. The confirmation screen showed `1,23 €` — and the finance manager who created it signed it off, because on that screen the number looked plausible next to a small line item.

What made it nasty:

- **Nothing failed.** The server received a positive number of cents, validation passed (`amount > 0`), the invoice was issued, the PDF generated, the email sent. Every step reported success.
- **Only some users were affected.** The ones whose locale formats with a decimal comma. Everyone testing in `en-US` saw a perfect flow for months.
- **The error was always downward.** `parseFloat` stops at the first character it doesn't understand, so the amount never got *bigger* — it got truncated. Undercharging doesn't page anyone; it shows up as "revenue is a bit low this quarter".
- **It was invisible in the UI.** The field displayed `1.234,56` the whole time. The number was correct on screen and wrong in the payload — two different values, one input.

## When it appears

The field was "pre-filled with the last invoice", which is where the formatting entered the pipeline:

```tsx
const [amount, setAmount] = useState(
  new Intl.NumberFormat(locale, {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(client.lastInvoice), // "1.234,56"
);

// on submit
const minor = Math.round(parseFloat(amount) * 100);
await api.createInvoice({ clientId, amountMinor: minor });
```

The trigger combination:

1.  **A number is formatted for humans** and then stored back as the input's value;
2.  **That same string is parsed back** with `parseFloat` / `Number`;
3.  **The locale is not `en-US`** — decimal comma, dot or space as the group separator;
4.  Nothing between those steps validates that the parsed value still resembles what the user saw.

## The mechanics of the bug

Three layers, each one individually defensible, stacked into a silent 1000× error.

### 1. `Intl.NumberFormat` output is not machine-readable — by design

```js
new Intl.NumberFormat("de-DE").format(1234.56); // "1.234,56"
new Intl.NumberFormat("en-US").format(1234.56); // "1,234.56"
new Intl.NumberFormat("fr-FR").format(1234.56); // "1 234,56"  ← U+202F narrow no-break space
```

The output is a *presentation* of a number, and it is intentionally lossy: the group separator varies (`.`, `,`, a normal space, a narrow no-break space), the decimal separator varies (`.`, `,`, `٫`), and in some locales the digits themselves are not ASCII:

```js
new Intl.NumberFormat("ar-EG").format(1234.56); // "١٬٢٣٤٫٥٦"
Number("١٢٣");                                   // NaN — JS only parses ASCII digits
```

`formatToParts` exists precisely because the formatted string cannot be reliably taken apart again. If your code does `format()` → store → `parse()`, you have built a lossy round trip and called it a data flow.

### 2. `parseFloat` understands exactly one number format

`parseFloat` is not "parse a number the way a human writes it". It is "read ASCII digits, optionally a `.`, then **stop at the first character you don't recognise**":

```js
parseFloat("1.234,56"); // 1.234   ← stops at the comma, keeps the dot as a decimal point
parseFloat("1,234.56"); // 1       ← stops at the comma immediately
parseFloat("1 234,56"); // 1       ← stops at the space
parseFloat("€1.234,56");// NaN     ← stops at the currency symbol
```

Note the first line: it doesn't even fail. It returns `1.234` — a plausible-looking float that then multiplies by 100 and rounds to `123`. The wrong answer has the same type as the right one.

`Number()` is stricter but equally locale-blind:

```js
Number("1.234,56"); // NaN
Number("1 234,56"); // NaN
Number("");         // 0    ← the silent zero
```

That last one is the second half of the trap: `Number("")` is `0`, not `NaN`. Any code path where the string ends up empty — a cleared field, a rejected value, a sanitised input — produces a valid, positive-looking, zero amount.

### 3. `<input type="number">` silently deletes what it can't parse

The obvious hardening — "let the browser validate it, use `type="number"`" — makes it worse:

```html
<input type="number" value="1.234,56" />
```

The value sanitisation algorithm runs: `1.234,56` is not a valid floating-point number, so the control's `value` becomes **`""`**. In React, the DOM value and the state diverge: the component thinks the field holds `1.234,56`, the DOM holds nothing, and the submit handler reads whichever one it happens to touch. Add `Number("") === 0` and you get an invoice for zero that passes `amount >= 0` validation.

`type="number"` also accepts `1e3` (→ 1000), silently converts `,` to `.` in some browsers and rejects it in others, and gives a validation message that says "Please enter a number" while the user is staring at a perfectly good number.

## Why the tests were green

- Fixtures were `"1234.56"` — the one string that survives `parseFloat` unchanged.
- The CI environment ran with `LANG=en_US`, so `Intl.NumberFormat` never produced a comma.
- Assertions compared `amountMinor` with a hardcoded number computed by the same helper. The test asserted that the bug was consistent with itself.

The cheapest regression test for this class of bug is a table of locales:

```ts
const CASES = [
  ["en-US", 1234.56, "1,234.56"],
  ["de-DE", 1234.56, "1.234,56"],
  ["fr-FR", 1234.56, "1\u202f234,56"],
  ["ar-EG", 1234.56, "١٬٢٣٤٫٥٦"],
] as const;

for (const [locale, value, expected] of CASES) {
  it(`round-trips ${locale}`, () => {
    const formatted = new Intl.NumberFormat(locale, {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    }).format(value);
    expect(formatted).toBe(expected);
    expect(parseLocalizedAmount(formatted, locale)).toBe(value);
  });
}
```

Three of those four lines fail against `parseFloat`. That's the whole bug, caught before it ships.

## How we found it

Reproduce first, then explain:

```js
parseFloat("1.234,56");                        // 1.234
new Intl.NumberFormat("de-DE").format(1234.56); // "1.234,56"
```

Then confirm on the live page: switch the app locale to `de-DE`, type `1.234,56`, submit, and read the payload in the Network tab. `amountMinor: 123`. Cause confirmed, not guessed.

The control experiment that made it undeniable: the same flow with `1,234.56` in `en-US` produced `123456`. Same code, same user, different separator — three orders of magnitude.

## The fix

**Formatted text never goes back into the machine.** Keep three separate things, and never let them merge:

1. **Canonical value** — integer minor units, the source of truth (`123456`).
2. **Draft string** — exactly what the user is typing, unformatted, untouched.
3. **Display string** — produced from the canonical value at render time, never stored.

```tsx
const [amountMinor, setAmountMinor] = useState(client.lastInvoiceMinor); // 123456
const [draft, setDraft] = useState(""); // what the user typed, raw

// on blur: parse the draft explicitly, reject what doesn't parse
const commit = () => {
  const parsed = parseLocalizedAmount(draft, locale);
  if (parsed === null) {
    setError("Enter an amount like 1.234,56");
    return;
  }
  setAmountMinor(Math.round(parsed * 100));
};

// display: always derived, never stored
const display = new Intl.NumberFormat(locale, {
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
}).format(amountMinor / 100);
```

And parse with the locale's own separators, discovered via `formatToParts` instead of guessed:

```ts
function parseLocalizedAmount(input: string, locale: string): number | null {
  const probe = new Intl.NumberFormat(locale).formatToParts(12345.6);
  const group = probe.find((p) => p.type === "group")?.value ?? "";
  const decimal = probe.find((p) => p.type === "decimal")?.value ?? ".";

  // 0. normalise locale digits (ar-EG writes ١٢٣, not 123) to ASCII
  const localeDigits = new Intl.NumberFormat(locale, { useGrouping: false }).format(9876543210);
  const digits = new Map(
    "9876543210".split("").map((ascii, i) => [localeDigits[i], ascii]),
  );
  let cleaned = [...input].map((ch) => digits.get(ch) ?? ch).join("").trim();

  // 1. drop group separators first (they can be "." or "," or a space)
  if (group) cleaned = cleaned.split(group).join("");
  // 2. drop every kind of whitespace (incl. U+00A0 and U+202F)
  cleaned = cleaned.replace(/\s/g, "");
  // 3. normalise the decimal separator to a dot
  if (decimal !== ".") cleaned = cleaned.split(decimal).join(".");

  // 4. reject anything that isn't a plain decimal number — never coerce
  if (!/^-?\d+(\.\d+)?$/.test(cleaned)) return null;
  return Number(cleaned);
}
```

The important part is the last two lines: **`null`, not `0`**. `Number("")` returning `0` is how a validation gap becomes a free invoice.

Two more rules that fintech products learn the hard way:

- **Use `type="text"` with `inputMode="decimal"`.** You get the numeric keyboard without the sanitisation that silently empties the field.
- **Echo the parsed amount back before committing.** A confirm step that renders "You are about to charge **€1.234,56**" from the canonical minor units catches the entire class of bug — the user sees the number the system will use, not the string they typed.

On the server side, add a cheap invariant: an invoice amount more than, say, 10× off from the client's contract or the previous invoice should be rejected or flagged, not silently accepted. Validation that only checks `amount > 0` validates the type, not the intent.

## Neighbouring traps in the same family

- **`toFixed`, `toLocaleString` and `Intl.NumberFormat` are display functions.** Feeding their output into arithmetic is the same bug with a different accent.
- **`Number("") === 0`, `Number(" ") === 0`, `Number("0x10") === 16`, `Number("1e3") === 1000`.** Every implicit conversion is a validation bypass.
- **`parseInt` has no radix by default in old code** and will happily read `"08"` as `8` but `"0x10"` as `16`. `parseInt(x, 10)` or nothing.
- **Locale is a user setting, not a constant.** `navigator.language`, the OS locale, the app's own language switcher and the server's locale can all disagree. Pick one source of truth for money formatting and pass it explicitly.
- **Currency symbols are not separators you can strip safely.** `"1.234,56 €"`, `"€ 1.234,56"`, `"CHF 1'234.56"` — the apostrophe is a group separator in Swiss formatting, and nobody's regex expects it.

---

## Takeaways (checklist)

1.  **`Intl.NumberFormat` output is not parseable.** Group separators, decimal commas, narrow no-break spaces and non-ASCII digits are all intentional; `formatToParts` exists because the string can't be taken apart reliably.
2.  **`parseFloat` stops at the first unknown character and returns a plausible wrong number.** `parseFloat("1.234,56") === 1.234` — not an error, just 1000× too small.
3.  **`Number("") === 0`.** Every empty-string path is a zero-amount path, and zero passes `amount > 0` checks in more codebases than anyone admits.
4.  **`<input type="number">` deletes values it can't parse.** The DOM value becomes `""` while React state keeps the old string — use `type="text"` with `inputMode="decimal"`.
5.  **Keep canonical, draft and display separate.** Minor units in state, raw text in the field, formatted text only at render time — never round-trip formatted output back into the input.
6.  **Reject, don't coerce.** A parser for money returns a number or `null`; it never falls back to `0`.
7.  **Test with a table of locales, not one fixture.** `en-US`, `de-DE`, `fr-FR`, `ar-EG` — if the round trip doesn't hold in all four, the bug is already in the code.


[⬅ Back to articles](../README.md)
