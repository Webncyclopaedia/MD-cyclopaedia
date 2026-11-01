[⬅ Back to articles](../README.md)


# IndexedDB finished my transaction while I was `await`ing

> A real-world case: an offline-first project dashboard (Vite + React + `idb`), desktop. Every project is stored in IndexedDB and synced when the network is available. The "Save" button worked for weeks. Then it started throwing `TransactionInactiveError: Failed to execute 'put' on 'IDBObjectStore': The transaction has finished.` — intermittently. Retry always helped. Tests were green the whole time.

## Symptom

Three behaviours, all from the same line of code:

1.  **Sometimes it throws.** Save a project → red toast, `TransactionInactiveError` in the console. Press Save again → works.
2.  **Sometimes it silently loses data.** The write throws inside a `try`, the `catch` logs a warning, the UI says "Saved". The record never lands in the database. On the next page load the user's edit is gone.
3.  **It correlates with the network.** Fresh data → fails. Cached data → fine. Which is the detail that sent everyone hunting for a race condition in React, in the sync layer, in the service worker — anywhere but the transaction.

The error itself is oddly phrased: nothing in the code "finished" the transaction. There is no `commit()` call, no `close()`, no `abort()`. The transaction decided on its own.

## When it appears

The save function looked completely reasonable:

```ts
import { openDB } from "idb";

async function saveProject(project: Project) {
  const tx = db.transaction("projects", "readwrite");
  const store = tx.objectStore("projects");

  // enrich the record with fresh metadata before writing
  const meta = await fetch(`/api/meta/${project.id}`).then((r) => r.json()); // ← the culprit

  store.put({ ...project, meta });

  await tx.done; // resolves on commit, rejects on abort
}
```

The trigger combination:

1.  A transaction is open (`db.transaction(...)`);
2.  Control is handed back to the event loop **before** the write — here via `await fetch(...)`, but any real async work does it;
3.  The next request (`store.put(...)`) is placed against a transaction that is no longer active.

No transaction, no write, no error at the `put` call site that tells you *why*.

## The mechanics of the bug

IndexedDB transactions have no `commit()`. They **auto-commit**, and the rule is a lifetime, not a method call:

> A transaction stays active during the task in which it was created (and its microtasks). It becomes inactive as soon as control returns to the event loop with no pending requests. If every request completed before that, it commits; otherwise it aborts.

Read that again, because it explains everything:

- `await Promise.resolve()` — **survives**. Microtasks drain before the transaction is deactivated.
- `await fetch(...)`, `await new Promise(r => setTimeout(r, 0))`, `await someOtherDbRequest`, `await import(...)`, `await file.text()` — **kills it**. Each of these resolves in a *later task*, which means the current task ended, which means the transaction committed.

That's why the bug is intermittent. The fast path (data already in memory, promise resolved synchronously) stayed inside the microtask checkpoint and worked. The slow path (a real network round trip) yielded to the event loop and the transaction was gone. Same function, two lifetimes.

And the trap in the test suite: `fetch` was mocked with `vi.fn().mockResolvedValue(meta)`. A resolved promise is a microtask. **The tests were green because the mock was faster than reality.** The bug was never reachable in the test environment — a test double that resolves immediately can hide an entire class of lifetime bugs.

## The second trap: `onsuccess` is not `oncomplete`

While debugging this, we found a latent bug one line away. The codebase also had this pattern:

```ts
const tx = db.transaction("projects", "readwrite");
tx.objectStore("projects").put(project);

tx.objectStore("projects").onsuccess = () => {
  toast.success("Saved"); // ← lies
};
```

`request.onsuccess` fires when **that request** succeeded — not when the transaction committed. If any later request in the same transaction fails, the whole transaction aborts and everything rolls back, while the user has already been told "Saved". The only reliable signal is `tx.oncomplete`:

```ts
tx.oncomplete = () => toast.success("Saved");
tx.onabort = () => toast.error("Save failed", tx.error);
```

Same family of bug: the success of a step is not the success of the unit of work.

## How we found it

The error message was the whole clue — `The transaction has finished` at a line that never touched the transaction lifecycle. Two checks:

```ts
// 1. Which await is it? Log around the yield.
const tx = db.transaction("projects", "readwrite");
console.log("before await", tx.mode); // "readwrite" — alive
const meta = await fetch(`/api/meta/${project.id}`).then((r) => r.json());
console.log("after await"); // the next put throws: the tx died here
```

```ts
// 2. Confirm the microtask rule in isolation.
const tx1 = db.transaction("projects", "readwrite");
await Promise.resolve();
tx1.objectStore("projects").put({ id: "micro" }); // works

const tx2 = db.transaction("projects", "readwrite");
await new Promise((r) => setTimeout(r, 0));
tx2.objectStore("projects").put({ id: "task" }); // TransactionInactiveError
```

The control experiment: move the `fetch` above `db.transaction(...)`. Error gone, permanently. Cause confirmed, not guessed.

## The fix

**Never hold a transaction across a real `await`.** Do the async work first, open the transaction last, and keep it synchronous:

```ts
async function saveProject(project: Project) {
  // 1. all async work happens BEFORE the transaction exists
  const meta = await fetch(`/api/meta/${project.id}`).then((r) => r.json());

  // 2. the transaction contains only synchronous request placement
  const tx = db.transaction("projects", "readwrite");
  tx.objectStore("projects").put({ ...project, meta });

  // 3. wait for the commit, not for the request
  await tx.done;
}
```

Three rules that fall out of it:

- **One transaction = one synchronous burst of requests.** Build the whole payload in memory first, then write.
- **Multiple stores are fine** — `db.transaction(["projects", "tags"], "readwrite")` — as long as every `put`/`delete` is placed in the same burst.
- **`tx.done` is a waiter, not a keep-alive.** Awaiting it after the burst is correct; awaiting it *between* writes does nothing to extend the lifetime.

If the logic genuinely needs to read something and then write based on it, do it in **two transactions**: read in one, compute, write in another. It's not atomic — and that's the honest trade. For atomicity across an async gap you need a different tool: `navigator.locks` to serialise the critical section across tabs, plus a version field on the record to detect a stale write.

```ts
// cross-tab critical section, if you really need read-modify-write
await navigator.locks.request(`project:${project.id}`, async () => {
  const current = await db.get("projects", project.id); // tx #1
  const next = merge(current, patch);
  const tx = db.transaction("projects", "readwrite"); // tx #2
  tx.objectStore("projects").put(next);
  await tx.done;
});
```

## The same family of traps, worth knowing before they bite

- **`put()` success ≠ committed.** Only `oncomplete` means durable; `onabort` means rolled back. Wire the UI to the transaction, not to the request.
- **One failed request aborts the whole transaction.** If you `put` five records and the third violates a unique index, all five are rolled back — and the error surfaces as `AbortError` on the transaction, not at the `put` line.
- **A version upgrade blocks on other open tabs.** If another tab holds a connection, `openDB` with a new version fires `blocked` and waits forever. Handle `versionchange` and close the old connection:

  ```ts
  db.onversionchange = () => {
    db.close();
    showReloadBanner("A new version is available — reload.");
  };
  ```

- **Structured clone drops your prototypes.** Store a class instance and you get a plain object back: methods gone, `instanceof` false. Store `Date`, `Map`, `Set`, `Blob` and they survive; store a function and you get `DataCloneError`.
- **`getAll()` returns key order, not insertion order.** `autoIncrement` keys are ascending, so "the newest record" is the last one only if you sort by a timestamp field.
- **Storage can be evicted.** Best-effort storage is cleared under pressure, so an offline-first app can lose everything. Call `navigator.storage.persist()` and handle the refusal.

---

## Takeaways (checklist)

1.  **IndexedDB transactions auto-commit.** They stay active for the current task plus its microtasks, and die the moment you yield to the event loop. There is no `commit()` to call.
2.  **`await fetch()` / `setTimeout` / another IDB request inside a transaction = `TransactionInactiveError`.** `await Promise.resolve()` survives — which is exactly why the bug looks random.
3.  **Do async work first, open the transaction last.** One transaction is one synchronous burst of requests; anything else needs two transactions.
4.  **A mock that resolves immediately hides the bug.** If your test double never crosses a task boundary, tests will never see a lifetime violation. Add one test with a real `setTimeout` delay in the middle.
5.  **`request.onsuccess` is not `oncomplete`.** Only the transaction's `oncomplete` means the data is durable; `onabort` means it was rolled back.
6.  **One failing request rolls back the whole transaction**, and the error arrives on the transaction as `AbortError`, not at the offending `put`.
7.  **Handle `versionchange` and `blocked`**, or a schema upgrade will hang forever because some other tab forgot to close its connection.


[⬅ Back to articles](../README.md)
