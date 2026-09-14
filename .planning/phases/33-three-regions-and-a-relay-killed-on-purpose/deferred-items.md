# Deferred items — Phase 33

## `npx tsc --noEmit -p packages/cloudflare` reports 6 pre-existing errors, unrelated to plan 33-01

**Measured 2026-09-13, before touching anything**: `git stash` to the unmodified tree at
`aae2396` (this plan's own base commit, identical to `develop`) and re-ran the exact command
from 33-01's `<verify>` block. The same six errors are present with zero edits applied:

```
packages/cloudflare/src/stop-closes-the-billed-socket.e2e.test.ts(302,52): error TS2339: Property 'o2' does not exist on type 'Window & typeof globalThis'.
packages/cloudflare/src/stop-closes-the-billed-socket.e2e.test.ts(310,16): error TS2339: Property 'o2' does not exist on type 'Window & typeof globalThis'.
packages/cloudflare/src/stop-closes-the-billed-socket.e2e.test.ts(316,52): error TS2339: Property 'o2' does not exist on type 'Window & typeof globalThis'.
packages/cloudflare/src/stop-closes-the-billed-socket.e2e.test.ts(338,44): error TS2339: Property 'o2' does not exist on type 'Window & typeof globalThis'.
packages/node/src/e2e-signin.ts(182,12): error TS2339: Property 'o2' does not exist on type 'Window & typeof globalThis'.
packages/node/src/e2e-signin.ts(207,18): error TS2339: Property 'o2' does not exist on type 'Window & typeof globalThis'.
```

**Cause, read rather than guessed**: `Window.o2` is declared by a `declare global { interface
Window { … } }` block in `packages/browser/src/tab-api.ts:1283` (also
`embedded-webview.ts:257`, `capability-harness.ts:232`). Running `tsc -p packages/cloudflare`
in isolation only includes `packages/cloudflare/src/**`, so that augmentation is never in the
program and every `page.evaluate(() => window.o2…)` callback in `stop-closes-the-billed-socket.e2e.test.ts`
(a `packages/cloudflare` file) and `packages/node/src/e2e-signin.ts` (a different package
again) fails to see it. This is a per-package `tsc -p` isolation gap, not anything plan 33-01
touched — `hosted-object.ts` is the only file this plan's Task 1 edited, and `tsc -p
packages/cloudflare` reports it clean.

**Out of scope for this plan** per CLAUDE.md's scope boundary ("only auto-fix issues directly
caused by the current task's changes") — the files are e2e specs and an unrelated node-package
script neither task in 33-01 touches, and fixing a cross-package global-augmentation visibility
gap is exactly the kind of unrelated repair the scope boundary forbids folding into this plan.
Logged here rather than fixed. Whoever picks this up: the fix is almost certainly a `tsconfig`
reference/include change (or duplicating the `Window` augmentation into a shared `.d.ts` all
three packages already include), not a change to either failing file's logic.

Both of 33-01's own tasks verified clean against this gap: Task 1's `tsc -p packages/cloudflare`
diff (before vs. after the edit) is the empty set — same six errors, same six lines, nothing
added and nothing removed by `hosted-object.ts`'s changes.
