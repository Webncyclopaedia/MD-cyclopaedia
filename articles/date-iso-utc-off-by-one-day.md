[⬅ Back to articles](../README.md)


# The booking that jumped a day back: `new Date("2026-03-01")` is not the date you think

> A real-world case: a Vite + React booking app. The user picks **March 1** on the calendar. The server stores the clean string `"2026-03-01"`. The confirmation page shows... **February 28**. The database is correct. The network response is correct. Only the rendered page is wrong — and only for some users, in some parts of the world.

## Symptom

A date that was picked once comes back as the day before.

- It works for the developer in Berlin. It breaks for users in New York or Los Angeles.
- The stored data is right: the database says `"2026-03-01"`, the JSON response says `"2026-03-01"`.
- So you check the logs — and the logs look fine too (more on why below, it's the sneakiest part).
- The page itself is the only place where March 1 has quietly become Feb 28.

## Where it starts

The code looks harmless:

```js
const bookedAt = new Date("2026-03-01"); // "turn the string into a Date"

form.setValue("date", bookedAt); // the DatePicker wants a Date object
```

And the display goes back the other way:

```js
date.toLocaleDateString(); // "2/28/2026" ← wait, what?
```

String in, `Date` in the middle, string out. Three steps, and one of them throws the day off by a few hours — enough to cross midnight.

## Why the browser does this

First fact about `Date`: it does **not** hold a date. It holds one number — milliseconds since Jan 1, 1970, UTC. A `Date` always means "an exact moment". To show it as a calendar day, the browser converts that moment into **your local clock time**. Where you live changes the answer.

Second fact: JavaScript parses date strings in two different ways depending on their shape:

| Input string | The browser reads it as |
| :--- | :--- |
| `"2026-03-01"` (only a date) | midnight **UTC** — one fixed point in time |
| `"2026-03-01T00:00:00"` (date + time, no zone) | midnight **your local time** |
| `"2026-03-01T00:00:00Z"` (date + time + `Z`) | midnight UTC — you asked for it |

Here is the trap. `"2026-03-01"` means "midnight UTC of March 1" — but midnight UTC is a **different hour of the day** depending on where you live:

```js
// Same line of code, two different users:

// A user in New York (5 hours behind UTC):
const d = new Date("2026-03-01");
d.toString();        // "Sat Feb 28 2026 19:00:00 GMT-0500"
d.getDate();         // 28 ← the calendar highlights the wrong cell

// A user in Tokyo (9 hours ahead of UTC):
const e = new Date("2026-03-01");
e.toString();        // "Sun Mar 01 2026 09:00:00 GMT+0900"
e.getDate();         // 1 ← everything looks normal
```

The string is the same. The `Date` is the same. But midnight UTC falls on **Feb 28, 7 pm** in New York and on **March 1, 9 am** in Tokyo. Only people behind UTC get bitten by this direction of the bug — which is exactly why it survived code review: the reviewer lived in Europe.

## The other direction of the same bug

One important correction to how I first explained this: the bug is **not only about parsing, and not only about users west of UTC**. The same midnight problem works in reverse — on the way **out** — for users east of UTC.

Imagine Tokyo again. A date picker builds the date from parts, in local time:

```js
// Tokyo user picks March 1:
const d = new Date(2026, 2, 1); // March 1, 00:00 Tokyo time

d.toISOString().slice(0, 10);   // "2026-02-28" ← one day back, right at save
```

Midnight **Tokyo** time of March 1 is still "Feb 28, 3 pm" in UTC. `toISOString()` always prints UTC, so the string saved to the server is `"2026-02-28"`. After a reload, the user sees the day before.

So the full picture:

| User lives in | Parsing date-only strings... | Saving local-midnight Dates with `toISOString()`... |
| :--- | :--- | :--- |
| West of UTC (New York) | breaks: the day slides back **when displaying** | looks fine |
| East of UTC (Tokyo) | looks fine | breaks: the day slides back **when saving** |
| Near UTC (London, winter) | looks fine | looks fine |

Either way, the day that gets eaten is always the one at the UTC boundary. My earlier "it only shifts backward, and only for Americans" was half the story: Americans lose the day on **read**, everyone east loses it on **write**. Both patches below are needed.

## The sneakiest part: logs that look correct

While debugging, two lines next to each other can both "look right":

```js
const d = new Date("2026-03-01");

d.toISOString();        // "2026-03-01T00:00:00.000Z" ← the original string, back again
d.toLocaleDateString(); // "2/28/2026"                ← the user's local day
```

`toISOString()` prints UTC — it hands back the exact string you started with. Every "local" getter (`getDate`, `getMonth`, `toLocaleDateString`) reads your own clock. So the first log says "all good" and the second one shows the bug — and you spend an hour doubting the second one.

## A quick test that proves it

Run this in the browser console of the affected user (or set your machine's clock for the test):

```js
function diagnose(input) {
  const d = new Date(input);
  console.table({
    "string in": input,
    "instant (UTC)": d.toISOString(),
    "day the user sees": d.toLocaleDateString(),
    // A Date's own offset, in minutes. Positive = you are BEHIND UTC
    // (New York: 300, London winter: 0, Tokyo: -540). The sign is the
    // opposite of what most people expect — that tripped me up too.
    "offset (min)": d.getTimezoneOffset(),
  });
}

diagnose("2026-03-01");
```

If **"instant (UTC)"** says one day and **"day the user sees"** says another — that's this bug, confirmed. If the offset is positive (Americas), the break shows on display; if negative (Asia, Oceania), watch out for the save path instead.

## The fix

### 1. Best fix: stop turning calendar dates into `Date` objects

A booking night, an invoice date, a birthday — these are **not moments in time**. They are just day labels. Keep them as strings everywhere:

```ts
type CalendarDate = string; // always "2026-03-01", always zero-padded, always 10 chars

function isCalendarDate(s: string): boolean {
  return /^\d{4}-\d{2}-\d{2}$/.test(s);
}

function isPast(day: CalendarDate, today: CalendarDate): boolean {
  // Safe to compare with < : both strings are equal-length and zero-padded,
  // so plain string comparison = date comparison. No timezone anywhere.
  return day < today;
}
```

Pass `"2026-03-01"` to the date picker via its string format option (most libraries have one — look for `valueFormat` or `dateFormat`). Zero `Date` objects involved, zero ways to drift. This one change removes the entire bug class.

### 2. If some library insists on a `Date`: build it from parts

Never parse a date-only **string**. Read the numbers out of it and build the `Date` yourself, in local time:

```js
// WRONG: parsed as midnight UTC, shifts under the user's feet
const d = new Date("2026-03-01");

// RIGHT: the string's own numbers, local noon
function parseCalendarDate(s) {
  const [y, m, day] = s.split("-").map(Number);
  return new Date(y, m - 1, day, 12, 0, 0, 0); // months count from 0!
}
```

Why **noon**? Because noon sits safely in the middle of the day. Even if this `Date` later shifts by a few hours through a serialization or a display call, 12:00 never crosses midnight, so the day we render stays the day we meant.

### 3. When saving: keep the string, or print UTC on purpose

The Tokyo bug came from `.toISOString()` on a local-midnight `Date`. Don't do that — send the string you already have. If you truly have a `Date` and need a string, format it in UTC **explicitly**, so every user gets the same answer:

```js
new Intl.DateTimeFormat("en-US", {
  timeZone: "UTC",     // read back exactly the moment the Date represents
  dateStyle: "long",
}).format(d);           // "March 1, 2026" — same in New York, Tokyo, anywhere
```

Mentioned as a correction: the rule is not "always avoid UTC". The rule is **one timezone, chosen by you**, from input to output. The bug appears exactly where the code switches timezones without saying so — parse = one switch, `toISOString` = the other.

### 4. The proper long-term fix: `Temporal.PlainDate`

JavaScript now has a type that really means "a calendar day, no clock attached":

```js
const day = Temporal.PlainDate.from("2026-03-01");

day.toString();       // "2026-03-01" — no drift, by design
day.equals(otherDay); // honest comparison
```

`PlainDate` has no time and no zone inside, so there is nothing to shift. Once support (or the polyfill) is acceptable for your project, every "date" field in the app should become a `PlainDate`.

---

## Takeaways (checklist)

1.  **`Date` is a moment, not a date.** Under the hood it's one number: milliseconds since 1970 UTC. Shows as a "day" only after a local-time conversion — and that conversion depends on where the user lives.
2.  **Parsing depends on the string's shape:** `"2026-03-01"` → midnight UTC; `"2026-03-01T00:00:00"` (no zone) → midnight local. Adding and removing the time part changes the timezone the browser assumes.
3.  **The bug has two exits — my first write-up missed one:** west of UTC it bites on **display** (parse date-only → local day slides back); east of UTC it bites on **save** (local-midnight `Date` + `toISOString()` → stored day slides back). Near UTC you see nothing, which is how it ships.
4.  **`toISOString()` "lies" while debugging:** it re-prints the original string and the log looks fine, while local getters on the next line show the shifted day. Don't trust the first good-looking log.
5.  **Keep calendar dates as strings through the whole pipeline.** Validate the `YYYY-MM-DD` shape and compare strings directly — but only because they're equal-length and zero-padded.
6.  **When a `Date` is forced on you, build it from parts at local noon** — `new Date(y, m - 1, d, 12)`. Noon can't slide across midnight; months count from 0.
7.  **Choose one timezone and keep it explicit end to end** (format with `timeZone: "UTC"` when re-showing a date-only value). The bug lives wherever a conversion happens silently. Or better: use `Temporal.PlainDate`, which has no zone to switch at all.


[⬅ Back to articles](../README.md)
