[⬅ Back to articles](../README.md)


# iOS Safari zooms the page when you tap an input — and the layout never recovers

> A real-world case: Vite + React SPA, a dark landing page. In the footer there's a subscribe form — one email field and a button, 14px type, straight from the design system. On desktop and on Android everything is fine. On iPhone 11, tapping the field makes the whole page jump to ~1.4×, the footer slides out of view, and after the keyboard closes the page snaps back with a jolt. Nothing in the form is broken. The browser is doing exactly what Apple told it to.

## Symptom

Three complaints arrived, all describing the same thing in different words:

- "When I tap the email field, the site gets huge."
- "The subscribe button jumps off the screen and I can't reach it."
- "After typing, the page stays shifted to the left."

What actually happens: on focus, iOS WebKit **auto-zooms the layout viewport** to make the focused control readable. The zoom is not a CSS transition — it's the browser changing `visualViewport.scale`. The layout viewport stays 375px wide while the user now sees ~270px of it, so anything anchored to the right edge (the button, the close icon, the sticky header) is simply outside the visible area. On blur, the zoom is usually reverted — but the revert is animated, it can race with the scroll position, and the page often ends up horizontally offset, which is where "the page stays shifted" comes from.

Two details that shaped the diagnosis:

1.  **Android does not do this.** Chrome on Android never auto-zooms on focus, so "it works on Android" proves nothing.
2.  **It reproduced in every iOS browser** — Safari, Firefox, Chrome — because on iPhone they all run WebKit. Same as [the icon bug](./svg-icons-ios.md): one engine, one behaviour, four different app icons.

## When it appears

One declaration is enough:

```css
.field input {
  font-size: 14px; /* ← the culprit: anything below 16px */
}
```

The trigger combination:

1.  **A form control** — `<input>`, `<textarea>` or `<select>` (yes, `select` too);
2.  **A computed font size below 16px**;
3.  **iOS WebKit**, at any zoom level below the maximum;
4.  The field receives focus — by tap, by "Next" on the keyboard, or programmatically.

16px is the threshold, and it's about the **computed** value in pixels, not the number you typed. That's the part that makes this bug subtle: `font-size: 0.875rem` is 14px in a default-root project and 8.75px in a project with `html { font-size: 62.5% }`. A `clamp()` or a media query can also drop a control under 16px only on narrow screens — which is exactly where the phone lives, so it looks like a device bug.

```css
.field input {
  /* reads as "16px-ish", computes to 14px */
  font-size: clamp(0.75rem, 2vw, 0.875rem);
}
```

## The mechanics of the bug

WebKit decides, on focus, whether the control is legible at the current scale. If the computed font size is under 16px, it treats the field as too small to type into and zooms the visual viewport until the text is roughly 16px tall. At a 14px font that's a ~1.14× zoom; combined with the browser's own minimum, users typically perceive something closer to 1.3–1.4×.

What makes the aftermath messy:

- **The layout viewport does not change.** No `resize` event fires, `window.innerWidth` stays 375. Media queries don't re-evaluate. But `visualViewport.scale`, `visualViewport.width` and `visualViewport.offsetLeft` do change.
- **`position: fixed` elements stay pinned to the layout viewport**, not to what the user sees. A fixed bottom bar with the submit button ends up below the visible area.
- **The zoom-out on blur is animated and independent of the scroll position.** If the user scrolled while zoomed, the page can settle a few dozen pixels off — the "stays shifted" complaint.

This is not a bug in the app. It's a deliberate accessibility feature: the browser refuses to let you type into an unreadably small field. The mistake is on the design side — the design system chose 14px for form controls, and nobody checked it on a phone.

## The "fix" that breaks the site for real users

The answer you'll find in the first ten search results:

```html
<!-- don't do this -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
```

It looks like a one-line fix, and on some iOS versions it even works. But:

- **It disables pinch-zoom for everyone.** That is a WCAG 1.4.4 ("Resize text") failure — people who need to magnify the page can no longer do it. Lighthouse and most accessibility audits flag it as a blocking issue.
- **Apple has been ignoring these flags since iOS 10.** `user-scalable=no` is not honoured any more; the site owner doesn't get to decide that. So the "fix" is unreliable *and* harmful.
- **It's a symptom-level patch.** The real defect — 14px form controls — remains, along with the tiny tap targets that usually come with it.

## How we found it

Two checks in the Safari console on the live page, no DevTools required:

```js
// 1. What does the browser think the control's font size is?
getComputedStyle(document.querySelector(".field input")).fontSize;
// → "14px". The rule was 0.875rem; the root is 16px. There it is.

// 2. Is the page zoomed right now? Focus the field first, then run:
visualViewport.scale;
// → 1 before focus, ~1.14 after. Confirmed: the viewport is scaled,
//   not the layout.
```

The control experiment that closed it: bump the field to `font-size: 16px` in DevTools, blur, tap again — no zoom. Cause confirmed, not guessed.

A useful habit: when a bug is described as "the page jumps / gets huge / goes off-screen", check `visualViewport` before touching CSS. `window.innerWidth` will happily tell you everything is normal while the user is staring at a magnified fragment of the page.

## The fix

Make the control legible at its natural size:

```css
.field input,
.field textarea,
.field select {
  /* fix: iOS WebKit auto-zooms any control
     with a computed font size below 16px */
  font-size: 16px;
}
```

Keep the smaller type on wider screens where nothing zooms:

```css
.field input,
.field textarea,
.field select {
  font-size: 14px;
}

@media (max-width: 768px) {
  .field input,
  .field textarea,
  .field select {
    font-size: 16px; /* computed, not declared */
  }
}
```

If the design absolutely requires 14px on mobile, the least-bad workaround is to keep the computed size at 16px and scale the box down visually:

```css
@media (max-width: 768px) {
  .field input {
    font-size: 16px;
    transform: scale(0.875);
    transform-origin: left center;
    width: 114.28%; /* compensate: 1 / 0.875 */
  }
}
```

It works, because WebKit reads the computed font size and never looks at the transform — but it costs you blurry text on non-integer scales and a layout box that no longer matches the visual one. Use it only when the brand book wins over engineering.

### The part that makes it truly annoying

The 16px rule is per control, and forms are full of controls that nobody thinks of:

- `<select>` dropdowns — same zoom, and the zoom usually fires *after* the native picker opens;
- `<textarea>` in a comment box;
- `input[type="date"]`, `input[type="search"]` — the reset stylesheet in your project may set `font: inherit`, which inherits 14px from the parent;
- a component library that ships `font-size: 0.875rem` in its own stylesheet, overriding your rule with higher specificity.

The reliable check is not "did I write 16px somewhere" but "what does `getComputedStyle` say" — for every control in the form, at a 375px viewport.

---

## Takeaways (checklist)

1.  **iOS WebKit auto-zooms on focus when a form control's computed font size is below 16px.** The zoom changes `visualViewport.scale`, not the layout viewport — no `resize` event, no media query re-evaluation.
2.  **Fix the font size, not the viewport.** `maximum-scale=1, user-scalable=no` disables pinch-zoom (a WCAG 1.4.4 failure), is ignored by modern iOS anyway, and leaves the real defect in place.
3.  **16px means computed px.** `0.875rem`, `clamp()` and media queries can all land below the threshold without the word "14px" appearing anywhere in the code.
4.  **Audit every control, not just `<input type="text">`:** `<select>`, `<textarea>`, date/search inputs, and anything inheriting `font: inherit` from a 14px parent.
5.  **Diagnose with `visualViewport`.** `scale > 1` after focus plus a computed `fontSize` under 16px is the whole diagnosis in two console lines.
6.  **"Works on Android" is not a test result.** Chrome Android doesn't auto-zoom; only an iOS device (or the iOS simulator) shows this.
7.  If the design must keep 14px, scale the box with `transform` while keeping the computed size at 16px — and accept blurry text and a mismatched layout box as the price.


[⬅ Back to articles](../README.md)
