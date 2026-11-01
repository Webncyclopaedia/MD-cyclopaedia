[⬅ Back to articles](../README.md)


# `z-index: 9999` and still underneath: the stacking context nobody declared

> A real-world case: Vite + React admin dashboard, desktop only. A "⋯" menu inside a project card opens a dropdown that must sit above the next card in the grid. The dropdown has `z-index: 9999`. It still renders **behind** the neighbouring card. Raising it to `99999` changes nothing. Deleting the dropdown and re-rendering it changes nothing. The CSS says 9999 — the browser says "no".

## Symptom

The overlay is painted under a sibling that has no `z-index` at all, or under a sticky header, or under a card that comes later in the DOM. Sometimes it's not hidden but **clipped** — the dropdown gets cut off at the card's bottom edge, as if the card had `overflow: hidden` (it doesn't).

What makes it maddening:

- The number keeps going up. `9999` → `99999` → `2147483647`. Nothing.
- DevTools shows the right `z-index` in the Styles pane. It's not being overridden, not crossed out, no `!important` fight.
- It's **layout-dependent**: the menu works in one card and breaks in the one directly below it.
- It reproduces in every desktop browser, which kills the "it's a Chrome bug" theory immediately.

## When it appears

The markup is boring:

```tsx
<section className="card-grid">
  <article className="card">
    <div className="card__actions">
      <button aria-haspopup="menu">⋯</button>
      <ul className="menu">…</ul>
    </div>
  </article>
  <article className="card">…</article>
</section>
```

```css
.card__actions {
  position: relative;
  transform: translateZ(0); /* ← "for smoother animations" */
}

.menu {
  position: absolute;
  top: 100%;
  right: 0;
  z-index: 9999; /* ← powerless */
}
```

The trigger combination:

1.  An ancestor of the overlay **creates a stacking context** — here via `transform`, but any of a dozen properties will do;
2.  The overlay's `z-index` only orders it **inside that ancestor's context**;
3.  That ancestor competes with its own siblings using the ancestor's own position in the stack — which is "somewhere in the middle", not 9999.

## The mechanics of the bug

`z-index` is not a global "bring to front" button. It's a **local** ordering key: it sorts elements **within one stacking context**, and nothing else. A stacking context is an independent paint unit — everything inside it is composited as a single layer, and no descendant can escape it, no matter what number it carries.

The spec has a list of things that create a stacking context. Every one of them is a property somebody added for an unrelated reason:

| Property | Typical innocent origin |
| :--- | :--- |
| `transform` ≠ `none` | `translateZ(0)` "GPU acceleration" hack, hover scale |
| `filter` ≠ `none` | `drop-shadow` on a card wrapper |
| `backdrop-filter` ≠ `none` | frosted-glass header |
| `opacity` < 1 | fade-in animation in progress |
| `will-change: transform/filter/opacity` | "performance tuning" left in the code |
| `isolation: isolate` | a reset or a `mix-blend-mode` guard |
| `contain: paint/layout/strict/content` | rendering optimisation |
| `container-type: inline-size` | container queries |
| `mix-blend-mode` ≠ `normal` | multiply/darken effects |
| `position: fixed` or `sticky` | any sticky header |
| `z-index` ≠ `auto` on a **flex or grid item** | a stray `z-index: 0` |
| `clip-path`, `mask`, `perspective` | decorative effects |

So the real stack looks like this:

```text
root stacking context
├─ .card (position: relative, z-index: auto)      ← paints first
│   └─ .card__actions  ← NEW STACKING CONTEXT (transform)
│       └─ .menu (z-index: 9999)  ← king of this tiny kingdom
└─ .card (second one, later in DOM)               ← paints after, on top
```

The menu is the most powerful element **inside** `.card__actions`. But `.card__actions` itself is an unpositioned nobody in the root context, and the second card comes later in the DOM — so it paints over the entire subtree, `9999` included. The number never leaves the box.

The same mechanism explains the "clipped dropdown": a stacking context with `contain: paint` or `filter` also clips its contents, so the menu is cut at the ancestor's bounds even without `overflow: hidden`.

## The diagnostic that finds the culprit in 30 seconds

Instead of guessing, ask the browser which ancestor is guilty:

```js
// Walk up from the overlay and report every ancestor
// that creates a stacking context, and why.
const CREATORS = {
  transform: (v) => v !== "none",
  filter: (v) => v !== "none",
  backdropFilter: (v) => v !== "none",
  perspective: (v) => v !== "none",
  clipPath: (v) => v !== "none",
  mask: (v) => v !== "none",
  opacity: (v) => Number(v) < 1,
  mixBlendMode: (v) => v !== "normal",
  isolation: (v) => v === "isolate",
  contain: (v) => /paint|layout|strict|content/.test(v),
  containerType: (v) => v !== "normal",
  willChange: (v) => /transform|filter|opacity|perspective/.test(v),
};

function stackingTrail(el) {
  const trail = [];
  for (let node = el.parentElement; node; node = node.parentElement) {
    const cs = getComputedStyle(node);
    const reasons = Object.entries(CREATORS)
      .filter(([prop, test]) => test(cs[prop]))
      .map(([prop]) => prop);
    if (cs.position === "fixed" || cs.position === "sticky") reasons.push(cs.position);
    if (reasons.length) trail.push({ node, reasons });
  }
  return trail;
}

console.table(stackingTrail(document.querySelector(".menu")));
```

The output names the element and the property. In our case it was one line: `.card__actions` — `transform`. The control experiment is one click: uncheck `transform` in DevTools and the menu jumps above the neighbour instantly. Cause confirmed, not guessed.

## The fix

### 1. Stop creating contexts you don't need

The `translateZ(0)` was added years ago to "fix jank" and survived three refactors. Removing it fixed the bug with zero visual difference:

```css
.card__actions {
  position: relative;
  /* transform: translateZ(0); — removed: it created
     a stacking context and trapped the dropdown */
}
```

Do this when the context is accidental. Do **not** do it when the context is load-bearing (a card that must clip its rounded corners, a header that must blur what's behind it).

### 2. Render overlays at the top level (the real fix)

Menus, tooltips, popovers and modals should not live inside the card. Render them into `document.body` and position them against the trigger:

```tsx
import { createPortal } from "react-dom";
import { useEffect, useRef, useState } from "react";

function CardMenu() {
  const triggerRef = useRef<HTMLButtonElement>(null);
  const [pos, setPos] = useState<{ top: number; left: number } | null>(null);

  const open = () => {
    const rect = triggerRef.current!.getBoundingClientRect();
    setPos({ top: rect.bottom + window.scrollY, left: rect.left + window.scrollX });
  };

  return (
    <>
      <button ref={triggerRef} onClick={open} aria-haspopup="menu">
        ⋯
      </button>
      {pos &&
        createPortal(
          // rendered as a direct child of <body>: no ancestor
          // stacking context can trap it any more
          <ul className="menu" style={{ top: pos.top, left: pos.left }}>
            …
          </ul>,
          document.body,
        )}
    </>
  );
}
```

Now the menu's nearest stacking context is the root, and `z-index` behaves the way everyone expects.

### 3. Use the top layer when you can

For modals and popovers, the browser has a layer that sits **outside every stacking context**: the top layer. `<dialog>.showModal()` and the `popover` attribute both promote the element there, so no `z-index` and no portal is needed:

```html
<button popovertarget="card-menu">⋯</button>

<ul id="card-menu" popover>
  <li><button>Rename</button></li>
  <li><button>Delete</button></li>
</ul>
```

The menu is painted above everything, ignores ancestor `transform`, and gets Escape-to-close and light-dismiss for free. It also solves the clipping problem — the top layer is not clipped by `contain` or `overflow`.

### 4. Only then, raise the ancestor

If restructuring is off the table, the working (but fragile) trick is to give the **ancestor** a `z-index` on a positioned element, so the whole subtree — the context included — moves up in the parent's order:

```css
.card {
  position: relative;
  z-index: 1; /* the whole card, and everything inside it, goes above the rest */
}

.card:has(.menu) {
  z-index: 2; /* the card that owns the open menu wins */
}
```

It works, and it's exactly the kind of fix that breaks the moment a second overlay opens in another card. Treat it as a stopgap.

---

## Takeaways (checklist)

1.  **`z-index` is local.** It orders elements inside one stacking context and never escapes it. A huge number inside a trapped subtree is worth nothing.
2.  **A dozen ordinary properties create stacking contexts:** `transform`, `filter`, `backdrop-filter`, `opacity < 1`, `will-change`, `isolation`, `contain`, `container-type`, `clip-path`, `mask`, `mix-blend-mode`, plus `fixed`/`sticky` positioning.
3.  **Diagnose with `getComputedStyle`, not with bigger numbers.** Walk the ancestor chain, list the properties that create a context, and uncheck the suspect in DevTools to confirm.
4.  **Portals for overlays, top layer for dialogs.** Render menus and tooltips into `document.body`; use `<dialog>`/`popover` when you need to beat every stacking context by design.
5.  **Audit "performance" CSS.** `translateZ(0)`, `will-change`, `contain: paint` are the usual suspects — they were added for speed years ago and now silently reshape the paint order.
6.  **If you must patch, raise the ancestor, not the child.** A positioned ancestor with `z-index` moves the whole subtree; raising the child does nothing outside its context.
7.  **`opacity < 1` during an animation creates a transient context.** An overlay can be broken only for the 300ms of a fade-in and work perfectly afterwards — a bug that "fixes itself" is still a bug.


[⬅ Back to articles](../README.md)
