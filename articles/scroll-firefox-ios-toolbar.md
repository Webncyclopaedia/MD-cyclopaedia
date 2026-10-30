# Firefox on iPhone: content jitters along with the toolbar

> A real-world case: Vite + React SPA, a dark landing page with an SVG hero at `100svh`. iPhone 11. The first screen is assembled from absolutely positioned elements inside a full-viewport-height hero. The symptom appears only in Firefox on iPhone; Safari on the same phone is perfectly clean, and desktop even more so.

## Symptom

When scrolling starts, the content of the first screen "jitters" — strictly in sync with the animation that hides Firefox's bottom toolbar. Swipe down — the toolbar slides away, and the hero content (name, profile, scroll hint) shifts with a jolt. Swipe up — the toolbar comes back, and the jolt goes in the opposite direction. Scrolling itself still works — the page doesn't get stuck — but visually the first screen "floats".

## The key fact that explains everything

**On iPhone, every browser is WebKit inside.** Apple requires third-party browsers to render through WKWebView; Firefox/Chrome have no "engine of their own" on iOS. So any bug here is a bug of the WebKit environment, not "broken Firefox".

The difference between the Safari app and Firefox on iOS lies in how toolbars are managed:

- **Safari**: the toolbars are a native part of WebKit. When they hide, the layout viewport stays stable, the `svh`/`vh` units are not recomputed, and the layout doesn't move.
- **Firefox iOS**: the toolbar is part of the app's UI. Hiding/showing it means **resizing the entire WKWebView frame**. And when the WebView itself is resized, everything is recomputed: `svh`, `vh`, `dvh`, `lvh` — for an embedded WebView, the very notion of a "stable small/large viewport" effectively doesn't exist.

## When it appears

A single visible element whose size is tied to viewport units is enough. In our case it was the hero:

    .hero {
      height: 100svh; /* ← in Firefox iOS this is a dynamic height */
    }

The chain: swipe → the toolbar starts hiding → the WKWebView is resized by ~60–120px → `100svh` is recomputed → the hero changes height → the absolutely positioned name, profile, and scroll hint shift → a visual jolt. `min-height: 100vh` on the sections below the fold changes at that moment too, but it goes unnoticed (only the document's total height changes) — which is why it's the first screen that "jumps".

## A false lead worth your time — so you don't waste your own

The first hypothesis was `overflow-x`: we tried every variation of "overflow on html", "on body", `hidden` vs `clip` — and the symptom wouldn't go away. Overflow was not the culprit at all; the recomputation of viewport units was. The marker pointing to the real cause: the jolt correlates strictly with the toolbar showing/hiding, not with scrolling itself.

## The fix

This can't be cured with CSS (any viewport unit is dynamic in this environment). What does cure it is locking the height in pixels while ignoring "toolbar" resizes:

    useEffect(() => {
      const hero = heroRef.current;
      if (!hero) return;

      const isTouch = window.matchMedia("(pointer: coarse)").matches;
      const threshold = isTouch ? 150 : 0; // toolbar ~60–120px < 150 → ignore
      let lockedHeight = window.innerHeight;

      const apply = () => { hero.style.height = `${lockedHeight}px`; };
      const onResize = () => {
        const h = window.innerHeight;
        if (Math.abs(h - lockedHeight) < threshold) return;
        lockedHeight = h;
        apply();
      };
      const onOrient = () => requestAnimationFrame(() => {
        lockedHeight = window.innerHeight;
        apply();
      });

      apply();
      window.addEventListener("resize", onResize);
      window.addEventListener("orientationchange", onOrient);
      return () => {
        window.removeEventListener("resize", onResize);
        window.removeEventListener("orientationchange", onOrient);
        hero.style.height = "";
      };
    }, []);

Nuances of the solution:

- **The 150px threshold** is empirical: mobile browser toolbars stay within ~120px, while real events (rotation, window resize) are larger. On desktop the threshold is 0 — the height always follows the window.
- `orientationchange` is handled separately via `requestAnimationFrame` — right after rotation, `innerHeight` is still the old value.
- The inline height is removed on unmount, and the CSS fallback `100svh` remains for the first frame.
- `min-height: 100vh` on the other sections doesn't need touching — their change is invisible to the eye. We lock only what the user actually sees.

---

## Takeaways (checklist)

1.  "A bug in Firefox on iPhone" is almost never Gecko: on iPhone, all browsers render with WebKit. If Safari on the same phone is fine, the culprit isn't the engine's layout but _how_ the app manages the WebView (toolbars, frame resizing).
2.  `svh`/`vh`/`dvh` in browsers running on an embedded WKWebView are dynamic: any element that is visible on screen and tied to viewport units will "float" when the toolbar shows/hides. Lock critical first screens in px with a thresholded `resize` handler.
3.  The marker of the real cause is the symptom correlating with the toolbar, not with scrolling. If the jolt coincides with the toolbar animation, forget overflow and look for viewport units.
4.  Lock the height only of the visible first-screen elements: invisible `min-height` changes below the fold don't bother users, and extra JS is an extra point of failure.
5.  Choose the threshold in the `resize` handler based on the physics of toolbars (~120px max) — it's what distinguishes a "toolbar" resize from a real one (rotation, window change).
