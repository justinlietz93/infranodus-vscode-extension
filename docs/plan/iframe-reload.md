# Iframe Reload — Why the Graph Loaded Empty After Switching Graph Sites

How the webview survives being destroyed and recreated, and the bugs that made the
graph render empty (and *stay* empty) after switching the `graphUrl` setting between
`https://graph.infranodus.com` and a local dev server like `https://localhost:5173`.

Companion to [`iframe-graph-protocol.md`](./iframe-graph-protocol.md), which documents the
`LOAD_JSON` vs `RECALCULATION` message contract this bug abused.

## The core lifecycle fact

The graph view is registered as a `WebviewView` (`extension.ts` `resolveWebviewView`) **without**
`retainContextWhenHidden`. VS Code therefore **disposes the webview's DOM whenever the view is
hidden** (collapsed, tab switched away, window reloaded) and re-runs `resolveWebviewView` when it
becomes visible again. Each re-resolve produces a **brand-new, empty `<iframe id="graphFrame">`**.

But two things *persist* across that destruction:

- **The provider object** (`InfraNodusViewProvider`) is registered once at activation, so its
  instance fields — notably `_lastProcessedKey` and `_initialLoadDoneForKey` — outlive any single
  iframe.
- **`vscode.setState(...)`** in the webview persists serialized state across reloads.

The recreate-empty-iframe vs. persistent-provider/state mismatch is the root of every bug below.

## The two ways graph data reaches a fresh iframe

```
extension host ──postMessage──> webview.html ──contentWindow.postMessage──> iframe
   topicsSubject                  message handlers                          graph app
```

1. **Push:** when `topicsSubject` emits, the extension posts `LOAD_JSON` (first time) or
   `RECALCULATION` (subsequent) to the webview, which forwards it into `iframe.contentWindow`.
   If the iframe app hasn't booted yet, that forwarded message is **dropped**.
2. **Replay fallback:** the webview persists the last graph to `vscode.setState({ graphData })`.
   When the iframe boots it posts `READY`; the webview's `sendGraphDataToIframe()` then replays
   `state.graphData` as a `LOAD_JSON`. This is what's supposed to repaint a recreated iframe.

Both paths have to stay healthy or a recreated iframe shows nothing.

## The bug chain

### Bug A — extension sends a partial `RECALCULATION` to an empty iframe (primary)

In `topicsSubject` (`extension.ts`):

```js
const isInitialLoad = this._initialLoadDoneForKey !== this._lastProcessedKey;
// isInitialLoad → full LOAD_JSON ; else → partial RECALCULATION ({ entriesAndGraphOfContext } only)
```

`_initialLoadDoneForKey` is **keyed to the document**, and only reset when the document changes
(`processDocument` / `processContent`). It is **not** reset when the iframe is recreated. So:

1. First render of a doc → full `LOAD_JSON`, `_initialLoadDoneForKey` set. Graph renders. ✅
2. Hide/show, window reload, or a `graphUrl` switch → **fresh empty iframe**, same document.
3. `isInitialLoad` is now `false` → extension sends `RECALCULATION`, a diff that assumes the iframe
   already holds a base graph (see `iframe-graph-protocol.md`). The empty iframe has nothing →
   **renders empty.**

Stuck in this state forever for that document; the only escape was switching to a different file
(which resets the key) or reloading the whole extension host.

### Bug B — `RECALCULATION` poisons the replay cache (makes it "sticky")

The webview's `RECALCULATION` handler persisted the **partial** payload as `graphData`:

```js
vscode.setState({ ...currentState, graphData: message.payload }); // payload = { entriesAndGraphOfContext } only
```

The full `LOAD_JSON` payload is the whole `data` object (`{ entriesAndGraphOfContext, topicNames }`);
the `RECALCULATION` payload is a subset. Overwriting `graphData` with the subset means the **replay
fallback** (`READY` → `sendGraphDataToIframe`) now sends a malformed `LOAD_JSON` on the next reload —
so even a perfectly healthy `graph.infranodus.com` iframe renders empty. This is why switching *back*
to the working site didn't fix it.

### Bug C — `graphUrl` changes weren't observed

`onDidChangeConfiguration` watched only `infranodus-graph-view.theme`. Changing `graphUrl` did
nothing until some unrelated event forced a webview re-resolve — and that forced reload is exactly
what triggered Bugs A and B.

### Bug D — `SET_IFRAME_URL` handler wiped `graphData` (latent; surfaced by the Bug C fix)

The `SET_IFRAME_URL` handler in the webview called `vscode.setState({ infraNodusIframeUrl, infraNodusTheme })`
**without spreading existing state**, silently dropping `graphData`. Once the Bug C fix made a
`graphUrl` change call `initializeWebview()` (which posts `SET_IFRAME_URL`), this path became live:
it would re-point the iframe correctly but wipe the replay cache, re-introducing an empty graph on
the *next* reload. Caught in review.

## The reproduction (user's exact sequence)

1. Switch to `localhost:5173` (server down) → iframe can't load, never sends `READY` → empty
   (expected). If a `RECALCULATION` fired here it already corrupted `state.graphData` (Bug B).
2. Switch back to `graph.infranodus.com` → reload recreates an empty iframe → extension sends
   `RECALCULATION` (Bug A) → empty; replay fallback uses the corrupted partial state (Bug B) →
   still empty.
3. Turn `localhost` back on → document key unchanged, so it's stuck in `RECALCULATION` mode →
   never recovers.

## The fixes

| # | Bug | File | Change |
|---|-----|------|--------|
| 1 | A | `extension.ts` `resolveWebviewView` | Set `this._initialLoadDoneForKey = null` before `initializeWebview()`. A recreated webview always has an empty iframe, so the next emission must be a full `LOAD_JSON`. Ties the "did we do the initial load" flag to the *iframe's* lifetime, which is what it actually means. |
| 2 | C | `extension.ts` `onDidChangeConfiguration` | Also handle `infranodus-graph-view.graphUrl` → `provider.initializeWebview()`. Made `initializeWebview` `public`. |
| 3 | B, D | `webview.html` | Introduced a `patchState(patch)` helper (`{ ...(vscode.getState() \|\| {}), ...patch }`) and routed **all four** `setState` sites through it, so persisted state is always merged, never overwritten. The `RECALCULATION` site now merges its partial payload into the stored full graph instead of replacing it. |

### Why the `RECALCULATION` merge is a *shallow* merge

```js
patchState({ graphData: { ...(currentRecalcState.graphData || {}), ...message.payload } });
```

`entriesAndGraphOfContext` is a **complete subtree** (the same field `LOAD_JSON` sends), not a diff
*within* itself. A shallow merge therefore replaces the graph wholesale while preserving sibling keys
like `topicNames` from the prior full load. A *deep* merge would be wrong — it could leave stale nodes
from the previous calculation. There's a `Do NOT turn this into a deep merge.` comment at the call site.

## The architectural lesson

`vscode.setState` is a flat key-value blob written from several handlers, each previously responsible
for remembering to spread prior state. Three did; the oldest (`SET_IFRAME_URL`) didn't — that was
Bug D. The `patchState` helper makes the "always preserve other keys" invariant **structural** rather
than something each new handler must remember, and prevents both Bug B and Bug D by construction.

A heavier alternative worth weighing: setting `retainContextWhenHidden: true` on the webview view
would keep the iframe alive when hidden and eliminate this entire recreate-empty-iframe bug class,
at the cost of memory for the live iframe. The fixes above make the recreate path correct either way.

## How to verify

- `npx tsc -p . --noEmit` — clean apart from two pre-existing errors in the `updateRemovedNodes`
  handler, unrelated to this work.
- Manual: load a graph on `graph.infranodus.com`, switch `graphUrl` to a dead `localhost:5173`
  (empty, expected), switch back → graph repaints without reloading the window.
- Logs: filter the webview devtools console by `[InfraNodus][webview]`. A healthy recreate shows
  `iframe → READY` followed by `sending stored graphData to iframe (LOAD_JSON)`, and the extension
  side logs `LOAD_JSON (initial load for key)` rather than `RECALCULATION` on the first emission
  after a reload.
