# Search & Editing Performance Improvements

## Context

These changes target large graphs (800 MB+ SQLite search index, thousands of pages/blocks) where
search and editing were noticeably slow.

---

## Changes

### 1. Debounce inline block search (`components/editor.cljs`)

**Problem:** `block-search-auto-complete` used a Rum `:did-update` lifecycle that fired a worker
search call on **every render cycle** with no debounce — one full round-trip per keystroke (or
faster, due to reactive re-renders).

**Fix:** Converted from `rum/defcs + rum/reactive + :init/:did-update` to
`rum/defc + rum/use-state + hooks/use-effect! + hooks/use-debounced-value q 150`.
This matches the existing `page-search-aux` pattern. The inline block `(( ))` autocomplete now
fires at most one worker search call per 150 ms window. The now-unused `search-blocks!` helper was
removed.

---

### 2. Cache `large-graph?` per repo (`worker/search.cljs`)

**Problem:** Every search call executed `(count (d/datoms @conn :avet :block/name))`, which scans
the entire `:block/name` datom index to count pages. For a graph with thousands of pages this is
an expensive operation per keypress, even though the value only changes when pages are created or
deleted.

**Fix:** Added a `page-counts` atom that caches `{repo -> count}`. The count is computed lazily on
first use and invalidated in `sync-search-indice` (which already detects page-level changes) and
`clear-fuzzy-search-indice!`. The datom scan runs at most once per session, plus once after each
page creation/deletion.

---

### 3. Suppress LIKE full-table-scan for inline search (`worker/search.cljs`, `handler/editor.cljs`)

**Problem:** For queries ≤2 characters the search worker executed a `LIKE '%x%'` SQL query as a
fallback. A leading wildcard `LIKE` query cannot use any index and performs a full table scan — very
expensive on a large database. This path was always hit when the user started typing a block
reference (`((a`).

**Fix:** Added `:skip-non-match?` option to `search-blocks`. `<get-matched-blocks` (the inline
search entry point) passes `:skip-non-match? true`, preventing the LIKE query. The FTS5 trigram
MATCH path still runs for all queries ≥3 characters. Short (1-2 char) inline queries now return
only FTS5 and fuzzy results.

Additionally, the inline search result limit was reduced from 100 to 20. The autocomplete dropdown
renders every result as a DOM node; 20 is sufficient for a block-reference picker and cuts rendering
cost significantly.

---

### 4. Fix O(N²) linear scan in `combine-results` (`worker/search.cljs`)

**Problem:** For each result ID, `combine-results` called
`(first (filter #(= (:id %) id) keyword-results))` — a full linear scan of the results list per
element. With 100+ results (the previous default limit) this was ~10 000 comparisons per search.

**Fix:** Replaced with a single `reduce` that builds an `id->result` map before iterating, making
the per-ID lookup O(1) instead of O(N).

---

## Verification

| Change | How to verify |
|--------|--------------|
| Debounce | Type `((abc` rapidly. DevTools > Application > Service Workers should show ≤1 worker message per 150 ms window, not one per character. |
| Page-count cache | Add `(js/console.count "page-count-computed")` inside the `large-graph?` fallback branch. Should print once per session, once more after creating a page. |
| LIKE suppression | Add `(prn :non-match (boolean non-match-input))` in `search-blocks`. Inline search with a 1-char query → `false`. Main Cmd-K search → `true`. |
| Result limit | Inspect the block-search autocomplete dropdown: ≤20 DOM nodes. |
| O(N²) fix | Search results should be identical before and after; visible as a CPU reduction in profiling. |
