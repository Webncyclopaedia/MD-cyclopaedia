[⬅ Back to articles](../README.md)


# Icons that iPhone "eats": iOS WebKit and percentage sizes on inline SVG

> A follow-up in the series on mobile WebKit surprises. The case: a dark portfolio landing page (Vite + React), round social buttons in the contacts block. On desktop and Android — perfect; on iPhone the Telegram and WhatsApp icons looked "eaten": collapsed, cropped, with broken geometry. Device — iPhone 11.

## Symptom

The social buttons are round 40×40px icon links. Two of the three contain inline SVG with custom paths (Telegram, WhatsApp); the third contains the MAX icon from the lucide-react library. On iPhone the first two rendered with a distorted box: squashed, clipped by the round container, or falling apart. The third one (lucide) was perfectly crisp. It reproduces in **every** iPhone browser — Safari, Firefox, Chrome (all of them are WebKit/WKWebView inside). On desktop the same code works flawlessly.

## When it appears

The markup and styles that break iOS WebKit:

```html
<a class="socialLink" aria-label="Telegram">
  <svg viewBox="0 0 16 16" fill="currentColor">
    <path d="..." />
  </svg>
</a>
```

```css
.socialLink {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
}

.socialLink svg {
  width: 45%;   /* ← the culprit */
  height: 45%;  /* ← the culprit */
}
```

The trigger combination:

1.  **Inline SVG without its own `width`/`height` attributes** — the size is set only through CSS percentages;
2.  **SVG inside a flex container** (`display: flex; align-items: center`);
3.  **iOS WebKit** — the Safari app and every WKWebView browser on iPhone.

## The mechanics of the bug

An SVG's percentage sizes are computed relative to its containing block. An SVG is a replaced element with its own intrinsic logic based on the viewBox, and when it sits inside a flex container (where a flex item's height is often derived from `auto`), WebKit resolves the SVG's final box differently from Blink and Gecko: the percentage height resolves incorrectly, and the icon gets a squashed or near-zero box that is then clipped by the round container. That is exactly what looks like the button's space "eating" the icon.

Nobody has documented WebKit's exact internals here — but the pattern reproduces reliably and is widely reported by developers: **percentage sizes on the inline SVG itself inside flex are a lottery on mobile WebKit**.

## The key clue that closed the diagnosis

Three buttons stood in one row. Two were broken, one wasn't. The only structural difference: the MAX icon is rendered by lucide-react and carries **fixed** `width="16" height="16"` attributes, while Telegram/WhatsApp rely on CSS percentages. The moment you pin the SVG sizes in pixels, both "broken" icons instantly straighten out. The cause was confirmed by comparison, not guessed.

## The fix

Percentages out, explicit pixels in:

```css
.socialLink svg {
  /* fix: iOS WebKit resolves % sizes incorrectly
     for inline SVG inside a flex container */
  width: 16px;
  height: 16px;
}

/* if the button shrinks on narrow screens —
   scale the icon with a separate rule */
@media (max-width: 500px) {
  .socialLink { width: 36px; height: 36px; }
  .socialLink svg { width: 14px; height: 14px; }
}
```

The same `45%` pattern lived in a second place on the page (the icons in the footer) — fixed there too before it surfaced. Classic: one questionable technique copied into several components means several future breakages.

## How to hunt for such bugs without DevTools on the phone

- **Contrast method**: place a known-good element next to the broken one (an icon with fixed attributes). A two-line difference in the code = suspect found;
- **Minimal repro**: extract the button with the percentage SVG into a separate snippet and compare rendering in desktop Safari (fine) and on iPhone (broken) — that filters out all the other layers;
- **Priority rule**: if the bug shows up only on iPhone and in all browsers at once, it's WebKit, not a specific browser; look in the CSS for non-standard techniques (percentages on SVG, overflow on the root, viewport units in dynamic toolbars).

---

## Takeaways (checklist)

1.  Inline SVG in UI elements — **explicit sizes only**: `width/height` attributes or CSS pixels. No percentages on the SVG itself.
2.  Scaling an icon relative to the button is done either with a separate rule per button size (media query) or via `em` with `font-size` on the container — but not with percentages on an SVG inside flex.
3.  If you use an icon library (lucide, heroicons), you're fine: the sizes are set as attributes. It's usually hand-inserted custom SVGs that break.
4.  A CSS technique that works "in every desktop browser" doesn't mean it works: mobile WebKit is its own universe. Test matrix: iPhone Safari + one WKWebView browser (Firefox iOS) + Chrome Android + desktop.
5.  One technique — many copies: before rolling a CSS pattern out across components, test it on a real iPhone. Otherwise every copy-paste site is a future bug report.


[⬅ Back to articles](../README.md)
