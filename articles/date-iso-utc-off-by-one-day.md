[⬅ Back to articles](../README.md)


# The booking that jumped a day back: `new Date("2026-03-01")` is not the date you think

> A real-world case: Vite + React booking app. The user picks **March 1** on the calendar, the API stores the clean string `"2026-03-01"`, the confirmation screen renders it back — and highlights **February 28**. The server says the record is correct. The local test suite passes. The bug appears only for users in the Americas and only on the "view" pages, never on the "edit" pages.

## Symptom

A date that was chosen once comes back as the previous day. The off-by-one rides on the timezone:

- In Berlin everything is fine. In New York, Los Angeles, São Paulo — the day before.
- It's **directional**: nobody ever sees "March 2" instead of "March 1", only "Feb 28". The drift is always one day back, never forward.
- Re-saving the booking "fixes" it, then it breaks again after the next roundtrip through the display code.
- The database column is a plain `DATE` ("2026-03-01"). The string in the network response is correct. Only the rendered calendar cell is wrong.

## When it appears

The suspicious line is short and innocent:

```js
const bookedAt = new Date("2026-03-01"); // "parse the date"

form.setValue("date", bookedAt); // a DatePicker receives a Date object
```

The trigger combination:

1.  The business concept is a **calendar date** — "the night of March 1", not "an instant in time";
2.  The code converts the string to a `Date` to feed a picker, then converts back for display;
3.  Those conversions go through **UTC** on the way in and through **local time** on the way out.

## The mechanics of the bug

The ECMAScript spec defines exactly two behaviours for string parsing, and they disagree:

| Input string | Parsed as |
| :--- | :--- |
| `"2026-03-01"` (date-only) | **UTC midnight** `00:00:00 UTC` |
| `"2026-03-01T00:00:00"` (date + time, no zone) | **local** midnight |
| `"2026-03-01T00:00:00Z"` (with `Z`) | UTC, explicitly |

So `new Date("2026-03-01")` is not "March 1" — it is *the instant UTC midnight of March 1*. For anyone west of Greenwich that instant falls on the **previous local day**:

```js
// A user in New York (UTC-5 in winter)
const d = new Date("2026-03-01");

d.toISOString();      // "2026-03-01T00:00:00.000Z"  ← the string came back intact
d.toString();         // "Sat Feb 28 2026 19:00:00 GMT-0500"
d.getDate();          // 28  ← the calendar highlights the wrong cell
```

The roundtrip then re-encodes the shifted instant. Display code usually does one of these, and both leak local time:

```js
d.toLocaleDateString("en-US"); // "2/28/2026"           ← local wall clock
d.getFullYear(), d.getMonth(); // 2026, 1               ← local components
```

The nastiest part: `toISOString()` prints the **original** string. So one log shows `"2026-03-01"` (looks fine!) while the very next line renders `Feb 28`. The bug hides between two "correct-looking" debug outputs.

Why it's always one day **back**: serialization moves midnight-UTC into a negative-offset local zone, and `Date` has no concept of "a date" to protect it — a `Date` in JavaScript is a millisecond count since epoch, nothing more. There is no such value as `Date(2026, 2, 1)` *as a calendar day*; you only ever hold instants.

The mirror-image trap also exists: `new Date("2026-03-01T00:00:00")` (without `Z`) is parsed as **local** time. Move the same string between a browser in Berlin and a server in UTC and it lands on two different instants. Date-only forms parse as UTC, date-time forms parse as local — the asymmetry is the bug.

## The diagnostic that proves it in 30 seconds

```js
function diagnose(dateInput) {
  const d = new Date(dateInput);
  console.log({
    input: dateInput,
    iso: d.toISOString(),              // which instant did it become?
    localDate: d.toLocaleDateString(), // which day does the user see?
    offsetMin: d.getTimezoneOffset(),  // minutes BEHIND UTC (NY: 300, Berlin: -60)
  });
}

diagnose("2026-03-01");
// New York: { iso: "2026-03-01T00:00:00.000Z", localDate: "2/28/2026", offsetMin: 300 }
// Berlin:   { iso: "2026-03-01T00:00:00.000Z", localDate: "3/1/2026",  offsetMin: -60 }
```

Run it in the affected user's timezone (or set the machine clock to New York). If `iso` and `localDate` disagree on the calendar day, you have this bug. An instant `getTimezoneOffset() > 0` (Americas) with a date-only input is the exact failure configuration — which is why European developers don't see it and CI doesn't either.

## The fix

### 1. Don't convert calendar dates to `Date` at all (the real fix)

A booking date, an invoice date, a birthday — these are not instants. Pass them through the system as the strings they already are:

```ts
type CalendarDate = `${number}-${number}-${number}`; // "2026-03-01"

function isCalendarDate(s: string): s is CalendarDate {
  return /^\d{4}-\d{2}-\d{2}$/.test(s);
}

// Comparison, sorting, "is it in the past" — all on strings,
// no timezone anywhere in the pipeline:
const isPast = (d: CalendarDate, today: CalendarDate) => d < today; // ISO strings sort lexicographically
```

The calendar highlights March 1 because it receives `"2026-03-01"` and the picker is configured with a `valueFormat`/`dateFormat` instead of a `Date`. Most libraries support this — use it before reaching for `Date`.

### 2. When a `Date` is required, parse the components yourself

```js
// WRONG: parsed as UTC midnight, shifts locally
const d = new Date("2026-03-01");

// RIGHT: build local noon from the string's own numbers
function parseCalendarDate(s) {
  const [y, m, day] = s.split("-").map(Number);
  return new Date(y, m - 1, day, 12, 0, 0, 0); // months are 0-based!
}
```

Anchoring at **noon local** is the safety net: even if this `Date` later shifts by a few hours through a serialization or a `toLocaleString` in some other zone, it cannot cross a midnight boundary and land on another day.

### 3. Produce the display string from the stored string, not from the instant

The "view as March 1" rendering should not go through `Date` formatting either:

```js
// If you must format, do it from parts — or pin the zone explicitly:
new Intl.DateTimeFormat("en-US", {
  timeZone: "UTC",              // read back the very instant the string encodes
  dateStyle: "long",
}).format(new Date("2026-03-01")); // "March 1, 2026" — for every user
```

`timeZone: "UTC"` on a date-only value is correct **and** stable across timezones — display stops depending on where the user lives.

### 4. The future-proof fix: `Temporal.PlainDate`

The platform now has a type that actually means "a calendar date with no clock":

```js
const day = Temporal.PlainDate.from("2026-03-01");

day.toString();        // "2026-03-01" — no drift, ever
day.equals(otherDay);  // honest comparison
day.toLocaleString();  // "March 1, 2026" — no instant, no zone involved
```

`PlainDate` cannot be shifted by a timezone because it holds no zone and no time. Where `Temporal` is available (or polyfilled), it is the correct type for every "date" field in the domain.

---

## Takeaways (checklist)

1.  **`Date` is an instant, not a date.** It stores milliseconds since epoch. "March 1" is not representable — everything you do with it goes through a timezone, explicitly or not.
2.  **The parse asymmetry is the spec:** `"2026-03-01"` → UTC midnight, `"2026-03-01T00:00:00"` (no zone) → local midnight. Same string minus the time part flips the timezone.
3.  **The drift is directional and regional:** users west of UTC see the previous day; European users and CI servers see nothing. Test with `Intl.DateTimeFormat().resolvedOptions().timeZone` set to an American zone.
4.  **`toISOString()` lies helpfully:** it re-prints the original string while every getter (`getDate`, `getMonth`, `toLocaleDateString`) reads local time. Two adjacent logs can both "look right".
5.  **Keep calendar dates as strings end to end.** Compare and sort ISO `YYYY-MM-DD` strings directly; pickers get `valueFormat`, not `Date` objects.
6.  **If a `Date` is unavoidable, construct it from parts and anchor at noon local** — `new Date(y, m - 1, d, 12)` cannot cross a midnight boundary under a timezone shift. Months are 0-based.
7.  **Format with `timeZone: "UTC"` when re-displaying a date-only value**, or use `Temporal.PlainDate` — both make the rendering identical for every user on the planet.


[⬅ Back to articles](../README.md)
