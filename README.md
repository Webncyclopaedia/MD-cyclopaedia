# 📚 Markdown encyclopaedia

**A Markdown knowledge base about web development: real bugs, the mechanics behind them, and fixes you can paste.**

Everything here comes from shipped front-end work — SPA landing pages, Vite + React projects, CSS layout in production. No link dumps, no "10 best frameworks" lists: each entry starts with a real symptom, explains **why** the browser behaves that way, and ends with working code.


## 📂 Articles

| # | Article | What it fixes |
| :-: | :--- | :--- |
| 1 | 🦊 [Firefox on iPhone: content jitters along with the toolbar](./articles/scroll-firefox-ios-toolbar.md) | Hero content jumps in sync with the toolbar hide/show animation |
| 2 | 🧭 [iOS Safari freezes on the first screen: scrolling is dead](./articles/scroll-safari-freeze.md) | Document scroll locked by a root-level `overflow-x: hidden` + `height: 100%` |
| 3 | 📱 [Icons that iPhone "eats": iOS WebKit and inline SVG percentages](./articles/svg-icons-ios.md) | Inline SVG squashed and clipped inside a round flex button |


## 💡 Navigation tip

Every article opens **and** ends with the **`⬅ Back to articles`** link. Use it to return to this list instantly, without the browser's back button.


## 🤝 Contributing

1. Copy the structure of an existing article: back link at the top, `# Title`, a case blockquote, sections, `---`, and a `## Takeaways (checklist)`.
2. Keep it web-dev focused: a reproducible front-end symptom, the mechanics behind it, and a fix.
3. Write in English, use fenced code blocks with an explicit language tag (`css`, `tsx`, `js`, `bash`), and translate code comments too.
4. Add the new entry to the table above.


*Feel free to contribute.*
