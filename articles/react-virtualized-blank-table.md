[⬅ Back to articles](../README.md)


# The virtualized table that rendered nothing: `AutoSizer`, `height: 0` and the percentage chain

> A real-world case: an internal ops dashboard (Vite + React + `react-virtualized`). A table of several thousand orders, virtualized so the browser survives. It worked on the "Orders" page. Then the same component was dropped into a new "Reports" page — and rendered **nothing**: no rows, no scrollbar, sometimes not even the header. No error, no warning, no console message. The data was there. The DOM was empty.

## Symptom

The component mounted, the network request succeeded, `rowCount` was `3481`, and the screen stayed blank. Three details narrowed it down:

1.  **It worked on the page where it was written** and failed on the page it was moved to. Same component, same props, same data — different layout around it.
2.  **Resizing the browser window sometimes made the rows appear.** Which means the data and the row renderer were fine; only the measurement was wrong.
3.  **There was nothing in the console.** `react-virtualized` does not throw when it is told to render into a zero-height box — it faithfully renders zero rows.

## When it appears

The page layout looked harmless:

```tsx
<div className="page">
  <header className="page__header">Reports</header>
  <div className="page__body">
    <AutoSizer>
      {({ height, width }) => (
        <Table
          width={width}
          height={height}
          rowCount={rows.length}
          rowGetter={({ index }) => rows[index]}
          rowHeight={48}
        >
          <Column label="ID" dataKey="id" width={120} />
          <Column label="Client" dataKey="client" width={240} />
          <Column label="Amount" dataKey="amount" width={140} />
        </Table>
      )}
    </AutoSizer>
  </div>
</div>
```

```css
.page {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.page__body {
  flex: 1;
  /* min-height: 0; ← missing, and it matters later */
}
```

The trigger combination:

1.  **A percentage height that has nothing to resolve against.** `AutoSizer` measures its parent's box; the parent is sized by a chain of percentages that ends in `auto`;
2.  **`AutoSizer` renders a zero-sized measuring element** and reads `parentNode.getBoundingClientRect()`;
3.  **`Table` needs explicit numeric `width` and `height`.** It has no intrinsic size, and it cannot infer one.

## The mechanics of the bug

The chain has three links, and breaking any one of them produces the same blank screen.

### Link 1: a percentage height needs a definite ancestor

```css
html, body, #root { height: 100%; } /* ← often missing in SPAs */
```

`height: 100%` on an element resolves against its containing block's height. If that height is `auto` (content-based), the percentage is treated as `auto`. In a Vite SPA the chain is usually `html → body → #root → .page`. Drop `height: 100%` anywhere in it and every percentage below collapses to `auto`:

```text
html   height: auto        ← body's 100% has nothing to resolve against
└─ body height: auto
   └─ #root height: auto
      └─ .page height: 100vh  ← only saved by the viewport unit
         └─ .page__body flex: 1 → content-based height
```

`100vh` saved the page in this case, but the moment a component is moved into a wrapper that uses `height: 100%` instead of `100vh`, the chain snaps and the body becomes content-sized.

### Link 2: `AutoSizer` measures the parent, and its own content is zero-sized

`AutoSizer` does not measure itself. It renders a zero-sized element, walks to `parentNode`, and reads that box with `getBoundingClientRect()`. Its children are rendered inside a container that contributes **no height of its own** — the virtualized grid is absolutely positioned inside it.

So the parent's height is derived from content that has no height. A flex item with `flex: 1` (= `flex: 1 1 0%`) and a zero-height content box gets exactly what the flex algorithm gives it: whatever space is left — and if the container has no definite height, that is **0**.

### Link 3: `Table` with `height={0}` renders zero rows, silently

`react-virtualized` renders a window of rows sized by `height` and `rowHeight`. With `height: 0`, the window is zero rows. No exception, no warning: rendering nothing is a valid answer to "show me the rows that fit in 0 pixels".

That's why the bug is invisible in code review. Every individual piece is correct:

- the CSS is valid;
- `AutoSizer` is used exactly as documented;
- `Table` receives the props it requires;
- the data is present.

The failure is a **contract between three layers**, none of which can see the others.

## Why it worked on the other page

The original page had `height: 100vh` on the wrapper **and** a `min-height: 0` somewhere in a parent that happened to be copied from a different component. The new page used `height: 100%` inside a flex container without a definite height. Nothing in the component changed; the contract around it did.

This is the general shape of layout bugs in component libraries: **the component is fine, the containing block is not.**

## How we found it

Measure, don't guess. In the console on the broken page:

```js
const body = document.querySelector(".page__body");
body.getBoundingClientRect().height;          // 0
getComputedStyle(body).height;               // "0px"
getComputedStyle(body.parentElement).height; // "auto"  ← the chain ends here
```

Then the control experiment that closed it — set an explicit height on the wrapper and watch the table appear:

```js
document.querySelector(".page__body").style.height = "600px";
```

Rows, header and scrollbar all materialised instantly. Cause confirmed, not guessed.

A reusable guard while debugging: log what `AutoSizer` actually computed.

```tsx
<AutoSizer>
  {({ height, width }) => {
    if (height === 0) console.warn("AutoSizer measured 0 — check the height chain");
    return (
      <Table width={width} height={height} rowCount={rows.length}
             rowGetter={({ index }) => rows[index]} rowHeight={48} />
    );
  }}
</AutoSizer>
```

## The fix

### 1. Give the chain a definite height, once, at the root

```css
html, body, #root {
  height: 100%;
}

.page {
  display: flex;
  flex-direction: column;
  height: 100%;      /* instead of 100vh, if the app is embedded */
}

.page__body {
  flex: 1;
  min-height: 0;     /* allow the flex item to shrink below its content */
  overflow: hidden;  /* the table owns its own scrollbar */
}
```

`min-height: 0` is the second half of the fix, and it is the same automatic minimum size that traps `flex: 1` items in the other axis — see [the flex article](./flex-item-min-width-auto.md). Without it, a flex item refuses to shrink below its content, the table grows past the viewport, and the *page* scrolls instead of the table — the classic "the header scrolls away and the virtualized rows are all rendered at once".

### 2. If the page itself should scroll, say so explicitly

For a table that scrolls with the document rather than inside a box, the documented pattern is `WindowScroller` plus `AutoSizer` with `disableHeight`:

```tsx
<WindowScroller>
  {({ height, scrollTop, registerChild }) => (
    <AutoSizer disableHeight>
      {({ width }) => (
        <Table
          ref={registerChild}
          width={width}
          height={height}
          scrollTop={scrollTop}
          rowCount={rows.length}
          rowGetter={({ index }) => rows[index]}
          rowHeight={48}
        >
          <Column label="ID" dataKey="id" width={120} />
          <Column label="Client" dataKey="client" width={240} />
          <Column label="Amount" dataKey="amount" width={140} />
        </Table>
      )}
    </AutoSizer>
  )}
</WindowScroller>
```

Now the height comes from the window, not from a parent box that may or may not have one.

### 3. Never render a zero-height table silently

A one-line guard turns a blank screen into a diagnosable state:

```tsx
{height === 0 ? (
  <div className="table-skeleton" aria-busy="true" />
) : (
  <Table height={height} width={width} rowCount={rows.length}
         rowGetter={({ index }) => rows[index]} rowHeight={48} />
)}
```

And if the table lives in a tab that is mounted while hidden (`display: none`), mount it lazily or force a re-measure when the tab becomes visible — a hidden container measures `0 × 0` and, depending on the resize detection in use, may not re-measure on its own.

## The second trap: heights cached by position, not by row

Once the table renders, the next bug is waiting in variable-height rows. `CellMeasurerCache` memoises measured heights **by index**:

```tsx
const cache = new CellMeasurerCache({ defaultHeight: 48, fixedWidth: true });

<List height={400} width={800} rowCount={rows.length} rowRenderer={rowRenderer} rowHeight={cache.rowHeight} />
```

Sort the table, and row `0` becomes the record that used to be row `57` — but the cache still remembers that "row 0 is 48px", because the cache knows nothing about your data. Rows overlap, text is clipped, the scrollbar length lies, and `scrollToIndex` lands in the wrong place. It reproduces only after a sort, which is why it survives for months.

The reliable fix is to invalidate on every dataset change, because the library's `keyMapper` is not honoured for `List` row heights in current versions:

```tsx
useEffect(() => {
  cache.clearAll(); // sort, filter, pagination, new page of data
}, [rows, sortKey, sortDirection]);
```

If heights are known in advance — a fixed two-line cell, a single-line amount — skip measurement entirely and pass a number to `rowHeight`. A cache you never need is a bug you never ship.

## Neighbouring traps in the same table

- **Mutable rows plus `PureComponent`.** `rows[i].amount = x` followed by `setRows(rows)` keeps the same object identity, so the row never re-renders. Copy the row, don't mutate it.
- **A component defined inside render.** A `const Row = () => ...` declared inside the parent creates a new component type on every render, so React remounts the whole subtree — scroll position resets, measurement restarts, jank everywhere. Hoist it.
- **Percentage column widths.** `Column width="20%"` collapses to `0`. `react-virtualized` needs pixels; compute them from the measured `width`.
- **`scrollToIndex` outside `rowCount`.** After a filter shrinks the dataset, a stale scroll target throws. Clamp it and re-run when `rowCount` changes.
- **`overscanRowCount={0}`.** Fast scrolling shows blank bands while rows mount. Two or three rows of overscan is usually enough.
- **A module-level cache in React 18 StrictMode.** A `CellMeasurerCache` created at module scope survives the double mount and keeps heights from the discarded tree. Create it in a `useRef`/`useState`, not at import time.

---

## Takeaways (checklist)

1.  **`AutoSizer` measures the parent, not itself.** Its content contributes no height, so a parent with `height: auto` reports `0` and the table renders zero rows.
2.  **Percentage heights need a definite ancestor.** Keep `html, body, #root { height: 100% }`, or use `100dvh`/`100svh` deliberately, and check the chain whenever a component moves to a new page.
3.  **`flex: 1` needs `min-height: 0`** in a column layout, or the item refuses to shrink and the page scrolls instead of the table.
4.  **`react-virtualized` fails silently on a zero-size box.** Add a guard that renders a skeleton (or logs) when `height === 0`; a blank screen should never be the only symptom.
5.  **For document-scrolling tables use `WindowScroller` + `AutoSizer disableHeight`** instead of hoping a parent box has a height.
6.  **`CellMeasurerCache` keys heights by index.** Sorting or filtering without `clearAll()` produces overlapping rows and a lying scrollbar.
7.  **Immutability, stable component identities and pixel column widths** are prerequisites, not optimisations — virtualization punishes all three.


[⬅ Back to articles](../README.md)
