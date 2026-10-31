[⬅ Back to articles](../README.md)


# Safari (iOS): the page "sticks" on the first screen — scrolling is dead

> A real-world case: Vite + React SPA, a dark landing page with an SVG hero at `100svh`, deployed on Vercel. Test device — iPhone 11. Everything worked perfectly on desktop — the problem showed up only on the phone, and only in Safari.

## Symptom

The page opens, the first screen renders, but swiping/scrolling does nothing — the document is pinned at scroll position zero. Chrome on the same phone scrolled fine. An important clue: going straight to `/project/:id` (a different component tree, the hack wasn't applied) — scrolling was alive. So the browser itself isn't to blame; it's what this particular page does to the document.

## When it appears

The code contained this legacy hack:

    // Portfolio.tsx — on mount
    [document.documentElement, document.body, root]
      .forEach((el) => { el.style.height = "100%"; el.style.margin = "0"; });

plus this in the global CSS:

    html, body, #root {
      min-height: 100vh;
      overflow-x: hidden; /* ← on three levels at once */
    }

The trigger combination:

1.  **`height: 100%` on `<html>` and `<body>`** (not `min-height`) — the root becomes a rigidly fixed box, and all content "overflows" it;
2.  **`overflow-x: hidden` on `html` and `body` at the same time.** Per spec, when one axis is `hidden`, the other computes to `auto` — so _both_ html and body claim the role of scroll container, and on top of that the clip propagates to the viewport. In iOS WebKit this creates a conflict of scrolling models: the engine can't decide who scrolls the document.

iOS Safari scrolls the document relative to the viewport with rubber-band logic, and when the root is a clipping box of fixed height, touch scrolling of the document is simply blocked. Chrome/Blink forgives this configuration; WebKit doesn't.

## How we diagnosed it (a method worth reusing)

In the Safari console — three checks:

    document.body.style.height        // "" — if it's not empty, this is suspect #1
    getComputedStyle(document.body).overflowX
    window.scrollTo(0, 2000); window.scrollY   // 0 with a live scrollHeight = scrolling is broken

And the control experiment that closed the diagnosis: right on the live page, `document.body.style.height = ""` — scrolling instantly came back to life. The cause was confirmed, not guessed.

## The fix

1.  **Remove the JS hack with `height: 100%`** entirely — it isn't needed: `min-height: 100vh` is already in the CSS, and `margin: 0` on body is set by any reset (Tailwind preflight).
2.  Take `overflow-x` off the document root (`html`/`body`) altogether. Horizontal overflow, if there is any, gets clipped on content wrappers via `overflow-x: clip`: `clip` doesn't create a scroll container, so `position: sticky` and native document scrolling are unaffected (the `visible` + `clip` pair is allowed by the spec for exactly this reason).

---

## Takeaways (checklist)

1.  `overflow-x: hidden` — in **one** place only, and never on `html`+`body` as a pair. Need horizontal clipping — use `overflow-x: clip` on a content wrapper.
2.  Don't manage the height of `html`/`body` from JS. Hacks like "let's set height: 100% so something stretches" on the document root are a landmine for iOS WebKit.
3.  If the bug reproduces on one page but not another — compare how the component trees differ and what styles/scripts they apply to the document. This clue narrowed the search down to a single component.
4.  Diagnosing mobile WebKit bugs without DevTools: console checks (inline styles, computed overflow, the `scrollTo` test) + the control experiment "zero out the suspect style right on the live page".
5.  Chrome/Blink forgives what WebKit doesn't: "it works in Chrome on the phone" is not an argument — test in Safari.


[⬅ Back to articles](../README.md)
