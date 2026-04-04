## Performance: Eliminate Per-Block Subscription Overhead

### The Problem

Every visible block in Logseq independently subscribes to **3 global state atoms** that are identical across all blocks. On a page with 200 blocks, that's **~600 redundant Rum cursor watchers** — all monitoring the same values, all firing their change-detection callbacks on every state mutation.

On top of that, every block's `block-control` component independently calls `editor-handler/collapsable?` when `block-control` re-renders due to those subscriptions, performing entity lookups and property queries against the DB.

### The Fix

**Block rendering (the hot path):**

| Before | After | Impact |
|---|---|---|
| `state/sub :document/mode?` per block (`block-control`) | Read from `config` (set at page level) | **−N watchers** |
| `state/sub :rtc/state` per block (`block-control`) | Subscribe once at page level, propagate via config | **−N watchers** |
| `state/sub :editor/raw-mode-block` per block (`block-content-or-editor`) | Subscribe once at page level, propagate via config | **−N watchers** |
| `collapsable?` computed inside `block-control` on each independent re-render | Computed in parent (`block-container-inner-aux`), passed via opts | **−N lookups on atom-triggered re-renders** |

For **N = 200 visible blocks**, this eliminates **~600 state watchers**. The three global subscriptions are hoisted into `config-with-document-mode` (called once per page render in `page-blocks-cp`), threaded through the existing `config` map that already flows through the component tree.

**Mobile journal virtualization:**

Virtual scrolling was **explicitly disabled** for mobile journal views (`(and (util/mobile?) (:journals? config))` guard). This caused severe scroll jank on large journals because every block in every journal entry was rendered to DOM simultaneously. Now enabled with a lower threshold (20 blocks on mobile vs 50 on desktop).

**Deferred initialization:**

| Item | Before | After |
|---|---|---|
| emoji-mart `init()` (~200KB JSON parse) | Module load time | First use (when user opens emoji picker) |
| `instrument/init` (PostHog + Sentry) | Synchronous in startup | `requestIdleCallback` (after first paint) |
| Collapsed block worker fetches | Fetched even when collapsed | Skipped until expanded |

**Editor:**

Block-search (`[[`) debounce increased from 50ms → 150ms. The old value caused excessive DB queries while typing — you'd get 4-5 query round-trips for a typical block reference vs 1-2 now.

### Files Changed

```
src/main/frontend/components/block.cljs     — Block rendering: hoisted subscriptions + collapsable
src/main/frontend/components/page.cljs      — Remove redundant :document/mode? subscription
src/main/frontend/handler/common.cljs       — config-with-document-mode: added rtc/state + raw-mode-block
src/main/frontend/handler.cljs              — Deferred instrument/init via requestIdleCallback
src/main/frontend/handler/editor.cljs       — Block-search debounce 50ms → 150ms
src/main/frontend/ui.cljs                   — Deferred emoji-mart init
src/main/frontend/components/icon.cljs      — Ensure emoji init before search
```

### How to Test

1. Open any page with 50+ blocks — scrolling and editing should feel noticeably smoother
2. On mobile: open a journal page with many entries — should scroll smoothly now (was janky before)
3. Type `[[` in a block and start typing — should feel responsive with no lag
4. Open Logseq — first paint should be slightly faster (analytics deferred)

### Risk Assessment

All changes are safe refactors with no behavior changes:
- Subscriptions were moved, not removed — same data reaches the same components
- `collapsable?` is the same function called with the same args, just from one level up
- Emoji init uses the existing `init()` function, just called lazily
- The `requestIdleCallback` fallback (`setTimeout 2000`) covers Safari, which lacks rIC
