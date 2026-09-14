# The judgement/dispatch ordering assertion compares two different clock sources

**2026-09-14.** `packages/node/src/speculation-agents.node.test.ts` (CHURN-02, criterion 3)
fails intermittently on a **quiet** host. Measured, attributed to a pre-existing cause, and
**not fixed** — the mechanism is not yet established and the cheap repair would weaken the
assertion. This file exists so the next person starts from the measurements rather than from
the runs.

## The symptom

```
AssertionError: the judgement must precede the dispatch it caused:
  judged at 17655 against a duplicate dispatched at 17655:
  expected 17655.0009765625 to be less than or equal to 17654.916875
 ❯ packages/node/src/speculation-agents.node.test.ts:1477
```

and, on a second occurrence with the file running alone:

```
  expected 6994.20703125 to be less than or equal to 6994.182708
```

Two inversions, **0.0841 ms** and **0.0243 ms**. Always the same assertion, always sub-millisecond,
always in the same direction: the judgement lands *after* the dispatch it caused.

## What was measured, and what each measurement rules out

| reading | value | what it settles |
|---|---|---|
| full `node` lane | 1 file failed / 266, 1 test / 3850 | the failure exists outside isolation |
| that lane's `[host conditions]` | **oversubscribed**, load/core 1.70 → 17.87, ceiling 4.00 | the banner would have blamed the host |
| file alone, run 1 | **FAILED**, host **quiet** (2.61 → 2.52) | **the banner was wrong: this is not load** |
| file alone, runs 2-3 | passed, quiet | intermittent, not deterministic |
| file alone, 10 further runs | **10 passed, 0 failed**, quiet throughout | rate is roughly **1 in 13 ≈ 8 %**, not "usually" |
| `git diff --name-only aae2396..HEAD` against this spec's import graph | no intersection | **Phase 33 is not the cause** |

The spec's repository imports are `@o2/net`, `core/src/executor/fixtures.ts`,
`capability-fixture.ts`, `fabric-node.ts`, `fs-blockstore.ts` and `strip-comments.ts`. Phase 33
touched none of them. `packages/node/src/reachability.ts` *was* touched and is **not** imported
here — it is the guard's entry-point list.

**"Passes in isolation" was false**, which `CLAUDE.md` § Measurement already warns is a claim to
verify rather than a diagnosis. Had the isolated re-run been skipped, the oversubscription banner
would have closed this as a host artefact and a real defect would have been filed as weather.

## What the two sides of the comparison actually are

`judgedAt` is the scheduler's own reading, taken at the instant `stragglers` was asked:

- `packages/core/src/job/submit.ts:2129` — `judgedAt = woke`
- `:2083` — `const woke = clock.now()`
- `:1648` — the default `JobClock` is `now: () => Date.now()` — the **wall clock**, epoch ms, integer resolution

`trackedDuplicate.startedAt` is a `performance.now()` span — the **monotonic clock**, sub-µs
resolution — and the spec converts the first into the second's basis by subtracting
`performance.timeOrigin` (`speculation-agents.node.test.ts:1461`).

The failing values carry that construction on their face: `.20703125` and `.0009765625` are the
complements of `performance.timeOrigin`'s own fraction, which is what an *integer* millisecond
minus a *fractional* origin looks like.

`submit.ts:812` already states the hazard in the field's own docblock:

> **The basis is `JobClock`'s, which by default is `Date.now()` — epoch milliseconds.** A reader
> comparing it against `performance.now()` spans must convert one of the two.

The spec converts the **basis**. It does not — and cannot, by subtraction — reconcile two
different **clock sources**. `performance.timeOrigin` fixes their relationship once, at process
start.

## A mechanism that was proposed and then refuted by its own arithmetic

The first reading of this was *`Date.now()` truncates to the millisecond, so the coarse value can
land after the fine one*. **That is wrong and is recorded here so it is not proposed again.**
Truncation moves a value only *downward*; `floor(judge) ≤ judge ≤ dispatch` holds for every
input, so quantisation alone makes the assertion **more** likely to pass, never less. Resolution
is not the mechanism.

What remains open is the divergence between `CLOCK_REALTIME` and `CLOCK_MONOTONIC` over the
seconds between process start and the judgement — tens of µs at 7 s and 17.6 s would be a few ppm.
**That is a hypothesis and nothing here measures it.** This project has buried two hypotheses whose
arithmetic fit; a number agreeing with a theory is not the theory's proof.

## Why nothing was changed

The assertion's own comment says it has **no tolerance band "because it is true by construction"**,
and names the defect it replaces: a published instant that merely reports the consequence again.
Adding a band would be the cheap repair and it would blunt exactly that case — widening what counts
as passing without understanding the cause, which is what this milestone has refused elsewhere.

The invariant is true by construction **in the code**: the decision is taken before `dispatchCopy`
is called. What is not established is that the **instrument** can see it, since it reads the two
instants from two clock sources.

## Where a fix should start

`SubmitOptions.clock` is an injectable port (`submit.ts:2411`), and the fixture already controls the
requestor process. Supplying a `JobClock` whose `now()` is `performance.timeOrigin + performance.now()`
puts both instants on **one** source at **one** resolution, after which no tolerance band is needed and
the assertion keeps its full strength. That is the shape to try first — but it must be
**flip-tested**: make the failure appear on demand before claiming a fix removed it. At roughly 1 in 13,
a green run proves nothing.
