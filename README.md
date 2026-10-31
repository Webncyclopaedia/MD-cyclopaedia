# 📚 Markdown encyclopaedia

**Practical guides, snippets, and fixes collected from real production cases.**

![Articles](https://img.shields.io/badge/articles-3-6366f1?style=flat-square)
![Focus](https://img.shields.io/badge/focus-mobile%20WebKit-000000?style=flat-square&logo=apple&logoColor=white)
![PRs](https://img.shields.io/badge/PRs-welcome-2ea44f?style=flat-square)

Every entry starts with a real bug, explains **why** it happens, and ends with a fix you can paste — not with a list of links.

---

## 📂 Articles

### 📱 Mobile WebKit series

Field notes on the iOS/WKWebView quirks that only show up on a physical iPhone. All three cases share the same diagnosis rule: *if it breaks on iPhone in every browser at once, the engine is WebKit, not the browser.*

| # | Article | What it fixes | Tags |
| :-: | :--- | :--- | :--- |
| 1 | 🦊 [Firefox on iPhone: content jitters along with the toolbar](./articles/scroll-firefox-ios-toolbar.md) | Hero content jumps in sync with the toolbar hide/show animation | `viewport units` `scroll` |
| 2 | 🧭 [iOS Safari freezes on the first screen: scrolling is dead](./articles/scroll-safari-freeze.md) | Document scroll locked by a root-level `overflow-x: hidden` + `height: 100%` | `overflow` `debugging` |
| 3 | 📱 [Icons that iPhone "eats": iOS WebKit and inline SVG percentages](./articles/svg-icons-ios.md) | Inline SVG squashed and clipped inside a round flex button | `svg` `flexbox` |

---

## 💡 Navigation tip

Every article opens **and** ends with the **`⬅ Back to articles`** link. Use it to return to this list instantly, without the browser's back button.

---

## 🧪 Test matrix used in these articles

| Target | Why it's there |
| :--- | :--- |
| iPhone Safari | Reference WebKit behaviour — the "ground truth" |
| iPhone Firefox / Chrome | Same WebKit, but with app-level toolbars and WebView resizes |
| Chrome on Android | Blink, which forgives many CSS configurations WebKit doesn't |
| Desktop | Baseline: if it fails here too, the bug isn't mobile-specific |

---

## 🤝 Contributing

1. Copy the structure of an existing article: back link at the top, `# Title`, a case blockquote, sections, `---`, and a `## Takeaways (checklist)`.
2. Write in English, use fenced code blocks with an explicit language tag, and translate code comments too.
3. Add the new entry to the table above.

---

*Feel free to contribute.*
