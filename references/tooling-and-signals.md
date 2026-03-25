# Tooling and Signals

Use this reference when choosing the next debugging layer or when setting up a layered stability workflow.

## Decision Matrix

| Symptom shape | Start with | Escalate to | Why |
| --- | --- | --- | --- |
| Stable pane blinks or feels rebuilt | Playwright + selector-level probe | React Performance Tracks | Identity loss and remounts are usually easier to prove than paints |
| Layout jumps or scroll resets | Playwright + layout/scroll probe | Chrome Performance + Rendering | CLS and geometry changes need browser timing |
| Stutter or delayed feedback | Playwright + LoAF probe | Chrome Performance | LoAF script attribution identifies the blocking code |
| Looks fine locally but breaks in prod | Existing probe if reproducible | web-vitals attribution + LoAF | Field-only issues need telemetry |
| Unsure whether React or browser | Playwright + selector-level probe | React Performance Tracks or Chrome based on first signals | Cheap evidence should pick the branch |

## What Each Tool Is Good For

### Static Analysis / Linting

Use as a pre-merge check. Catches root causes before they become runtime symptoms.

Best for:
- Detecting known jank-producing code patterns before merge
- CI enforcement with zero runtime cost
- Catching React Compiler bail-outs (`react-hooks/todo` rule)

Key tools:
- `eslint-plugin-react-hooks` (v6.1+) -- official, includes React Compiler validation rules
- `eslint-plugin-react-perf` -- inline object/array/function literals in JSX that defeat memoization
- `@eslint-react/eslint-plugin` -- 4-7x faster, covers hooks, RSC, DOM, naming
- `oxlint` -- 50-100x faster pre-pass with 700+ built-in rules

Weak at:
- Layout shift from missing dimensions (no mainstream ESLint rule for `<img>` without width/height outside Next.js)
- Runtime-dependent patterns (timing, device speed, network conditions)

### Playwright + Local Probe

Use as the default reproducible layer.

Best for:
- Scripted repro and CI artifacts
- Selector replacement and flicker detection
- Layout shift with element attribution (via `sources` array)
- LoAF with per-script attribution (Chrome 123+, falls back to long tasks)
- DOM mutation churn summaries

Weak at:
- Exact React commit attribution
- Paint invalidation regions
- Layer/compositor detail

### React Performance Tracks (React 19.2+)

Use when the problem smells like rerender or remount churn. Shows in Chrome DevTools Performance panel.

Best for:
- Mount/unmount badges per component (remount = unmount then mount)
- Cascading update detection (which component scheduled an update during render)
- Changed props inspection (dev builds)
- Effect duration flamegraph
- Four priority subtracks (Blocking, Transition, Suspense, Idle)

Weak at:
- Browser paint/compositor analysis
- Production parity unless profiling builds are enabled

### React Scan

Use as a fast visual rerender detector. Browser extension works on any React site.

Best for:
- Visual highlighting of rendering components
- "Memoizable" tags for components that would benefit from React.memo
- FPS drop detection during interactions
- Programmatic render budgets via `getReport()` for CI enforcement

Weak at:
- Render timing (only counts, not durations)
- Layout shift, paint, compositor detail
- Production use (unsafe outside dev)

### why-did-you-render

Use for deep prop/state equality analysis when you need to know exactly *what changed*.

Best for:
- Identifying when a component rerenders with semantically identical props/state
- Deep-equality comparison that catches new-object-every-render patterns

Weak at:
- **Incompatible with React Compiler** -- do not use in compiler-enabled projects
- Production realism
- Browser rendering detail

### Chrome DevTools Performance + Rendering

Use when the problem smells like layout, paint, or compositing.

Best for:
- Screenshots per frame
- Layout shift clusters with element attribution (Insights sidebar)
- INP 3-phase breakdown (input delay / processing / presentation)
- LoAF integration in console (script attribution for slow interactions)
- Forced reflow detection (auto-surfaced in Insights)
- Paint instrumentation and Layers panel

Weak at:
- Fast automation loops unless you already have a stable repro

### Visual Regression Testing

Use for screenshot-based stability checks in CI.

Best for:
- Detecting visual regressions between builds
- Element-level or full-page comparison
- Baseline management

Key tools:
- Playwright `toHaveScreenshot()` -- built-in, free, uses pixelmatch
- Chromatic -- Storybook-centric, component-level
- Percy (BrowserStack) -- cross-browser, AI review agent

Weak at:
- Cannot catch transient flicker (`toHaveScreenshot()` waits for stability before comparing)
- Requires baseline maintenance

### web-vitals Attribution

Use when the issue is intermittent, route-specific, or only visible in production.

Best for:
- CLS with element-level attribution (`largestShiftTarget`)
- INP with 3 subparts + LoAF script entries
- LCP with 4-phase breakdown
- Tying bad metrics to pages, routes, or UI states

Weak at:
- Proving a single local visual glitch
- Non-CLS jank (flicker, remount churn, Suspense flash)

## Layered Workflow

| Layer | When | Tools | What it catches |
|---|---|---|---|
| Static (every commit) | CI | ESLint + React Compiler rules + oxlint | Root cause patterns |
| Dev-time (local iteration) | Dev | React Scan + Performance Tracks | Rerender storms, remounts |
| Lab/CI (every PR or nightly) | CI | Playwright probe + `toHaveScreenshot()` | CLS, flicker, frame gaps, visual regression |
| Field (production) | Prod | web-vitals attribution + LoAF + RUM | Real-user CLS, INP, intermittent issues |

## Suggested Starting Budgets

Use report-only first. Promote these to assertions only after the interaction is stable.

| Signal | Budget | Context |
|---|---|---|
| Non-input layout shift (CLS) | `< 0.02` | Google's field "good" is 0.1 at p75; per-interaction budget is intentionally stricter |
| INP per interaction | `< 200ms` | Google's "good" threshold; aspirational: < 100ms |
| Long animation frames > 50ms | `0` preferred, `<= 1` tolerated | With LoAF, also log `sourceURL` for attribution |
| Tracked selector flicker | `0` | Any flicker on a stable surface is a finding |
| Tracked selector replacement on stable panes | `0` | Identity loss on a stable surface is a finding |
| Max frame gap | `< 75ms` | ~4 dropped frames at 60fps |

If the app intentionally resets a full workspace, mark that interaction as a controlled remount instead of pretending it is stable.
