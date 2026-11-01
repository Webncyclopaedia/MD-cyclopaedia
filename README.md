# 📚 Markdown encyclopaedia

**A Markdown knowledge base about web development: real bugs, the mechanics behind them, and fixes you can paste.**

Everything here comes from shipped front-end work — SPA landing pages, Vite + React projects, CSS layout in production. No link dumps, no "10 best frameworks" lists: each entry starts with a real symptom, explains **why** the browser behaves that way, and ends with working code.


## 📂 Articles

| # | Article | What it fixes |
| :-: | :--- | :--- |
| 1 | 🦊 [Firefox on iPhone: content jitters along with the toolbar](./articles/scroll-firefox-ios-toolbar.md) | Hero content jumps in sync with the toolbar hide/show animation |
| 2 | 🧭 [iOS Safari freezes on the first screen: scrolling is dead](./articles/scroll-safari-freeze.md) | Document scroll locked by a root-level `overflow-x: hidden` + `height: 100%` |
| 3 | 📱 [Icons that iPhone "eats": iOS WebKit and inline SVG percentages](./articles/svg-icons-ios.md) | Inline SVG squashed and clipped inside a round flex button |
| 4 | 🧱 [The flex item that refuses to shrink: `min-width: auto`](./articles/flex-item-min-width-auto.md) | Horizontal page scroll on mobile caused by a flex item that can't shrink |
| 5 | 🔍 [iOS Safari zooms the page when you tap an input](./articles/ios-input-autozoom.md) | Form controls under 16px trigger auto-zoom, shifting fixed UI off-screen |
| 6 | 🥞 [`z-index: 9999` and still underneath: the stacking context nobody declared](./articles/z-index-stacking-context.md) | Overlays trapped by an ancestor `transform`/`filter`/`opacity` |
| 7 | 🗄️ [IndexedDB finished my transaction while I was `await`ing](./articles/indexeddb-transaction-inactive.md) | `TransactionInactiveError` from awaiting inside an auto-committing transaction |
| 8 | 🪙 [The cent that never came back: `Math.round` is not symmetric](./articles/money-rounding-negative-ledger.md) | One-directional ledger drift caused by rounding negative refund amounts |
| 9 | 🌍 [€1.234,56 became €1,23: `parseFloat` and the locale](./articles/locale-money-input-parsefloat.md) | Locale-formatted money parsed back with `parseFloat` → 1000× too small |


## 💡 Navigation tip

Every article opens **and** ends with the **`⬅ Back to articles`** link. Use it to return to this list instantly, without the browser's back button.


## 🤝 Contributing

1. Copy the structure of an existing article: back link at the top, `# Title`, a case blockquote, sections, `---`, and a `## Takeaways (checklist)`.
2. Keep it web-dev focused: a reproducible front-end symptom, the mechanics behind it, and a fix.
3. Write in English, use fenced code blocks with an explicit language tag (`css`, `tsx`, `js`, `bash`), and translate code comments too.
4. Add the new entry to the table above.


*Feel free to contribute.*
