# Tooling And Signals

Use this reference when choosing the next debugging layer.

## Decision Matrix

| Symptom shape | Start with | Escalate to | Why |
| --- | --- | --- | --- |
| Stable pane blinks or feels rebuilt | Playwright + selector-level probe | React DevTools Profiler | Identity loss and remounts are usually easier to prove than paints |
| Layout jumps or scroll resets | Playwright + layout/scroll probe | Chrome Performance + Rendering | CLS and geometry changes need browser timing |
| Stutter or delayed feedback | Playwright + frame-gap and long-task probe | Chrome Performance | Main-thread pressure is browser-visible |
| Looks fine locally but breaks in prod | Existing probe if reproducible | `web-vitals` attribution | Field-only issues need telemetry |
| Unsure whether React or browser | Playwright + selector-level probe | React DevTools or Chrome based on first signals | Cheap evidence should pick the branch |

## What Each Tool Is Good For

### Playwright + local probe

Use as the default reproducible layer.

Best for:

- scripted repro
- CI artifacts
- selector replacement and flicker detection
- layout shift, long task, frame-gap, and DOM churn summaries

Weak at:

- exact React commit attribution
- paint invalidation regions
- layer/compositor detail

### React DevTools Profiler

Use when the problem smells like rerender or remount churn.

Best for:

- commit timing
- subtree hotspots
- confirming whether a surface rerendered or remounted too often

Weak at:

- browser paint/compositor analysis
- prod parity unless profiling builds are enabled

### react-scan / why-did-you-render

Use only as dev-side helpers.

Best for:

- avoidable rerender detection
- quick local diagnosis

Weak at:

- prod realism
- browser rendering detail

### Chrome DevTools Performance + Rendering

Use when the problem smells like layout, paint, or compositing.

Best for:

- screenshots per frame
- layout shift clusters
- timing on the main thread
- paint instrumentation and layer behavior

Weak at:

- fast automation loops unless you already have a stable repro

### web-vitals attribution

Use when the issue is intermittent, route-specific, or only visible in production.

Best for:

- CLS and interaction regressions in the field
- tying bad metrics to pages or UI states

Weak at:

- proving a single local visual glitch

## Suggested Starting Budgets

Use report-only first. Promote these to assertions only after the interaction is stable.

- non-input layout shift: `< 0.02`
- long tasks over 50 ms: `0` preferred, `<= 1` tolerated
- tracked selector flicker: `0`
- tracked selector replacement on stable panes: `0`
- max frame gap: `< 75 ms`

If the app intentionally resets a full workspace, mark that interaction as a controlled remount instead of pretending it is stable.
