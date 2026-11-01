[⬅ Back to articles](../README.md)


# The flex item that refuses to shrink: `min-width: auto` and the horizontal scroll on mobile

> A real-world case: Vite + React SPA, a dark portfolio landing page. The contacts block is a list of rows: an icon on the left, a link on the right. Desktop — perfect. iPhone 11 — the page could be dragged sideways, the header "floated" off the screen edge, and the whole layout looked a few pixels wider than the device. Nothing in the CSS looked wrong. Nothing was wrong — it was working exactly as specified.

## Symptom

On the phone the document became horizontally scrollable. The overscroll was small — 10–40px, sometimes more — so the page didn't look "broken", it looked *drunk*: a swipe could shift everything left, the fixed header stopped being fixed, and text near the right edge got cut off. Sometimes the effect only appeared on one route (the one with a long email address in the contacts block), sometimes on all of them.

Two details that shaped the diagnosis:

1.  **On desktop, at full width, the bug was invisible.** Not "hard to notice" — genuinely absent. Shrinking the browser window to ~360px made it appear instantly.
2.  **It reproduced in every browser** on the phone — Safari, Firefox, Chrome — and in Chrome on Android. Unlike the other entries in this series, this is not a WebKit bug.

## When it appears

The markup was a perfectly ordinary flex row:

```tsx
<div className="contact">
  <span className="contact__icon" aria-hidden="true">
    <MailIcon />
  </span>
  <div className="contact__body">
    <a href="mailto:hello@verylongdomainname.example">
      hello@verylongdomainname.example
    </a>
  </div>
</div>
```

```css
.contact {
  display: flex;
  align-items: center;
  gap: 12px;
}

.contact__body {
  flex: 1; /* ← the culprit */
}
```

The trigger combination:

1.  **A flex container** (or a grid container — the same story);
2.  **A flex item that is allowed to shrink** — `flex: 1`, `flex: 1 1 0`, or just the default `flex-shrink: 1`;
3.  **Content with a wide "minimum"** — an unbreakable string (email, URL, hash, long word), a `<pre>` block, a table, a wide image, a nested `display: flex` row;
4.  **A viewport narrower than that minimum.**

## The mechanics of the bug

`flex: 1` is shorthand for `flex: 1 1 0%`: grow 1, shrink 1, base size 0. Reading that, you'd expect the item to shrink to whatever space is left. But there is a second rule that overrides the shrinking — **the automatic minimum size**.

Every flex item has `min-width: auto` by default, and per the Flexbox spec `auto` on a flex item resolves to the **content-based minimum size** — roughly, the width of the widest unbreakable piece of content. `flex-shrink` is not allowed to go below the minimum. So:

- `flex-basis: 0%` says "start from zero";
- `flex-shrink: 1` says "shrink if needed";
- `min-width: auto` says "but never below 214px, that's how wide this email address is".

The item wins. It stays at its min-content width, the flex container overflows, and since nothing clips the overflow, it escapes all the way up to the document. The document becomes wider than the viewport → horizontal scroll on the page. On a 1400px desktop the email fits inside the available space, so the minimum is never reached and the bug never fires. Narrow the viewport — and the trap closes.

The same applies in the block direction: `min-height: auto` on a flex item in a `column` container, and `min-width: auto` on **grid** items (which is why `grid-template-columns: 1fr` sometimes blows out a layout and `minmax(0, 1fr)` fixes it).

## The "fix" that makes it worse

This is the part that costs a day of debugging. The natural reaction to "the page scrolls sideways" is:

```css
html, body {
  overflow-x: hidden; /* ← the trap */
}
```

The horizontal scroll disappears, so the ticket is closed. But:

- the content is still 40px too wide — it's just clipped now, and on a narrow phone that means text cut off mid-word with no way to reach it;
- on iOS Safari this exact pair of rules on `html` + `body` can kill document scrolling entirely — see [the previous entry in this series](./scroll-safari-freeze.md);
- `overflow-x: hidden` on a wrapper (not the root) creates a scroll container, which silently breaks `position: sticky` inside it.

The symptom gets hidden, the cause stays, and a second, nastier bug is added on top. This is the classic shape of a subtle bug: **the workaround is more expensive than the fix.**

## How we found it

The diagnostic that took five minutes after two hours of guessing. In the console on the live page:

```js
// Is the document actually wider than the viewport?
document.documentElement.scrollWidth > document.documentElement.clientWidth;
// → true. Confirmed: something sticks out.
```

```js
// Who exactly sticks out? Walk the DOM and compare right edges.
const vw = document.documentElement.clientWidth;
[...document.querySelectorAll("*")]
  .filter((el) => el.getBoundingClientRect().right > vw + 1)
  .map((el) => `${el.tagName}.${el.className} → ${Math.round(el.getBoundingClientRect().right - vw)}px`);
```

The list came back with the contacts row and everything inside it — the classic signature of "one child is too wide, its ancestors are just reporting the damage". Then the control experiment: in DevTools, on the suspect element, set `min-width: 0`. `scrollWidth` immediately returned to `clientWidth`. Cause confirmed, not guessed.

Two habits worth keeping:

- **Shrink the desktop window to 360px before opening DevTools on the phone.** Most "mobile-only" layout bugs are viewport-width bugs, and they reproduce for free on desktop.
- **Suspect the deepest element, not the outermost.** The DOM walk above shows a whole chain; the culprit is the innermost one whose own content is wide.

## The fix

One declaration:

```css
.contact__body {
  flex: 1;
  min-width: 0; /* allow shrinking below the content's minimum */
}
```

If you want the long string to actually wrap instead of being ellipsized, use `overflow-wrap: anywhere` — and note the trap: `overflow-wrap: break-word` (and `word-break: break-word`) allow a break in the rendered text but **do not change the min-content size**, so the flex item still refuses to shrink and the overflow survives. `anywhere` does change the intrinsic size, which is exactly what's needed here:

```css
.contact__body {
  flex: 1;
  min-width: 0;
  overflow-wrap: anywhere; /* break the email/URL itself */
}
```

For the "truncate with an ellipsis" variant, `min-width: 0` is equally mandatory — without it `text-overflow: ellipsis` never gets a chance to fire, because the box is never narrower than its content:

```css
.contact__body {
  flex: 1;
  min-width: 0;
}

.contact__link {
  display: block;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
```

And the grid equivalent:

```css
.cards {
  display: grid;
  /* 1fr has an automatic minimum too: use minmax(0, 1fr) */
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
}
```

### The part that makes it truly annoying

`min-width: 0` only works on the element that is *itself* a flex/grid item. If the chain is nested, every level needs it:

```tsx
<div className="row">        {/* flex container */}
  <div className="row__main">{/* flex item → needs min-width: 0 */}
    <div className="row__inner">{/* flex item again → needs it too */}
      <pre>…</pre>
    </div>
  </div>
</div>
```

Fix only the innermost one and the overflow stays, because the middle wrapper is now the one that can't shrink. When a "fixed" layout still overflows, re-run the DOM walk — the culprit just moved one level up.

## Where else this pattern hides

- A code block (`<pre>`) inside a card: the card refuses to shrink and the whole page scrolls sideways instead of the code block.
- A header with `justify-content: space-between`: a long project title pushes the burger button off-screen.
- A chat bubble with a pasted URL: one message blows out the entire column.
- A `<table>` inside a `1fr` grid cell.
- A nested flex row inside a flex row inside a flex row — three `min-width: 0`s, and someone always forgets the middle one.

---

## Takeaways (checklist)

1.  **`min-width: auto` is the default on every flex and grid item**, and it resolves to the content's minimum size. `flex: 1` does not mean "shrink freely" — it means "shrink down to min-content".
2.  **Horizontal scroll on mobile is almost never a "mobile" bug.** Shrink the desktop window to 360px first; if it reproduces, you're debugging layout, not a device.
3.  **Diagnose with numbers, not with eyes:** `scrollWidth > clientWidth` proves the overflow, the DOM walk finds the innermost offender, `min-width: 0` in DevTools confirms the cause.
4.  **Fix the minimum, don't hide the overflow.** `overflow-x: hidden` on `html`/`body` clips content, breaks `position: sticky`, and on iOS Safari can kill scrolling outright.
5.  **`min-width: 0` belongs on every nested flex/grid item in the chain**, not just the innermost one. If the fix "didn't work", the culprit moved one level up.
6.  **`overflow-wrap: anywhere` changes min-content size; `break-word` does not.** If the long string must break, use `anywhere` — otherwise the item keeps refusing to shrink.
7.  Grid has the same trap: `1fr` → `minmax(0, 1fr)` when the cell content can be wider than the column.


[⬅ Back to articles](../README.md)
