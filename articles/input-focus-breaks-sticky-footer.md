[⬅ Back to articles](../README.md)

# Chrome on Android: input focus breaks sticky footer positioning

> A real-world case: Next.js + Tailwind CSS PWA, a light checkout page with a sticky CTA button at the bottom. Google Pixel 7. The layout uses a classic flexbox container where the action button is pinned to the bottom using sticky bottom-0. The symptom appears only in Chrome on Android; mobile Firefox and iOS devices handle the interaction perfectly.

## Symptom

When a user taps any text input field on the page, the virtual keyboard slides up, and the sticky footer button "jumps" directly on top of the keyboard, blocking half of the input form. Tapping outside the input hides the keyboard, and the button drops back to the bottom with a noticeable delay. The page remains scrollable, but the sticky element creates a massive visual obstruction, obscuring the fields the user is trying to type into.

## The key fact that explains everything

**On Android, Chrome resizes the visual viewport when the software keyboard appears.** While iOS pushes the entire page viewport upward without changing the layout height, Android Chrome modifies the actual height of the window. This causes elements relying on absolute or sticky positioning relative to the viewport boundary to immediately adapt to the new, truncated screen space.

The difference in viewport behavior during keyboard invocation breaks down as follows:

- **iOS Safari / WebKit:**: The keyboard acts as an overlay. The layout viewport remains unchanged, meaning elements anchored to the bottom stay at the true bottom of the page, underneath the keyboard layout layer, until scrolled.
- **Android Chrome / Blink:**: The keyboard reduces the available window height. The browser triggers a global resize event, shrinking the viewport. As a result, position: sticky; bottom: 0; evaluates the "bottom" as the top edge of the virtual keyboard.

## When it appears

This layout flaw triggers on any mobile view where an element is locked to the viewport boundary while form inputs are active on the same screen:

```html
<div class="fixed bottom-0 left-0 w-full">
  <!-- in Android Chrome, this container rides up with the keyboard -->
  <button>Proceed to Payment</button>
</div>
```

The chain: tap input → virtual keyboard slides up → the browser window height shrinks by ~40% → the viewport boundary shifts upward → the sticky/fixed element repositions itself relative to the new boundary → the button blocks the user's field of view. While standard content flows naturally, any element anchored via bottom-0 or h-screen gets compressed into the remaining visible space.

## A false lead worth your time — so you don't waste your own

The initial approach was trying to fix it with CSS padding adjustments and visualViewport API listeners to dynamically add bottom margins. We also experimented with changing the container to position: absolute inside a relative wrapper. None of these CSS-only overrides solved the issue; as long as the layout relied on the default window resizing behavior, the browser kept forcing the element upward. The definitive clue was that the layout only broke when the input field gained active focus.

## The fix

The most robust solution is to use the modern CSS interactive-widget meta tag property. This explicitly tells Chrome how to handle the viewport layout when interactive elements like the virtual keyboard appear, preventing the frame resize entirely:

```html
<head>
  <meta
    name="viewport"
    content="width=device-width, initial-scale=1.0, interactive-widget=resizes-content"
  />
</head>
```

However, if you need a runtime JavaScript fallback for older Android WebView environments or specific progressive web app configurations, you can actively toggle a utility class based on input focus states:

```tsx
useEffect(() => {
  const inputs = document.querySelectorAll("input, textarea, select");
  const footer = footerRef.current;
  if (!footer) return;

  const handleFocus = () => {
    // Temporarily hide or un-stick the footer when keyboard is active
    footer.classList.add("android-keyboard-active");
  };

  const handleBlur = () => {
    footer.classList.remove("android-keyboard-active");
  };

  inputs.forEach((input) => {
    input.addEventListener("focus", handleFocus);
    input.addEventListener("blur", handleBlur);
  });

  return () => {
    inputs.forEach((input) => {
      input.removeEventListener("focus", handleFocus);
      input.removeEventListener("blur", handleBlur);
    });
  };
}, []);
```

Nuances of the solution:

- **The interactive-widget property** is the cleanest modern declaration. Setting it to resizes-content or utilizing overlays ensures the layout ignores keyboard dimensions.
- The JavaScript focus/blur combination acts as a targeted override, ensuring the button remains resting naturally in the document flow instead of fighting the keyboard interface.
- Standard text layout remains completely readable since the page body can still be scrolled independently beneath the static elements.

---

## Takeaways (checklist)

1.  "A bug with sticky elements on Android" stems from window layout behavior: Android Chrome resizes the actual viewport frame when the keyboard appears, whereas iOS treats the keyboard as an overlay interface.
2.  `position: fixed` and `position: sticky` anchored to the bottom will always ride up on top of the Android keyboard by default. Use the viewport `interactive-widget` meta property to control this behavior globally.
3.  The definitive indicator of this issue is an element shifting positions only when a form field gains focus. If the layout breaks exclusively during text entry, shift focus away from general layout styling and target viewport resize mechanics.
4.  Apply overrides specifically to the bottom-pinned elements: altering the entire page container layout can break native scroll behaviors across other mobile operating systems.
5.  Combine modern meta tags with targeted focus state listeners if your application needs to support legacy hybrid app webviews or older mobile browser versions.

[⬅ Back to articles](../README.md)
