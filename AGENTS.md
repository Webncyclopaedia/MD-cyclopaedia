# AGENTS.md

Project conventions for this repository (Markdown articles about software development).

## Article format

- Every article lives in `articles/<slug>.md` and is written in English.
- Start the file with a back link as the very first line, then a blank line:

  ```
  [⬅ Back to articles](../README.md)
  ```

- Repeat the same back link as the last line of the file, separated from the content by a blank line (the `README.md` promises it: "Inside each article find the `⬅ Back to articles` link").
- Keep the rest of the structure consistent with existing articles: `# Title`, an opening blockquote with the real-world case, `##` sections, a `---` divider, and a `## Takeaways (checklist)` list at the end.
- Use fenced code blocks with an explicit language tag (`css`, `tsx`, `js`, `bash`, ...) — GitHub only syntax-highlights fenced blocks that declare a language; 4-space indented blocks render monochrome.
- Never leave Cyrillic text behind when converting or translating an article; verify with a search for `[А-Яа-яЁё]`.
