# 📚 Markdown encyclopaedia

**Practical guides, snippets, and fixes collected from real production cases.**


## 📂 Articles

Every entry starts with a real bug, explains **why** it happens, and ends with a fix you can paste — not with a list of links.

| # | Article | What it fixes |
| :-: | :--- | :--- |
| 1 | 🦊 [Firefox on iPhone: content jitters along with the toolbar](./articles/scroll-firefox-ios-toolbar.md) | Hero content jumps in sync with the toolbar hide/show animation |
| 2 | 🧭 [iOS Safari freezes on the first screen: scrolling is dead](./articles/scroll-safari-freeze.md) | Document scroll locked by a root-level `overflow-x: hidden` + `height: 100%` |
| 3 | 📱 [Icons that iPhone "eats": iOS WebKit and inline SVG percentages](./articles/svg-icons-ios.md) | Inline SVG squashed and clipped inside a round flex button |


## 💡 Navigation tip

Every article opens **and** ends with the **`⬅ Back to articles`** link. Use it to return to this list instantly, without the browser's back button.


## 🤝 Contributing

1. Copy the structure of an existing article: back link at the top, `# Title`, a case blockquote, sections, `---`, and a `## Takeaways (checklist)`.
2. Write in English, use fenced code blocks with an explicit language tag, and translate code comments too.
3. Add the new entry to the table above.


*Feel free to contribute.*
