# Field Path

Use this path when the issue is intermittent, route-specific, or only visible in production.

## Rules

- Add telemetry before guessing.
- Do not claim a local fix solved a prod-only problem without field evidence.
- Keep local repros and field metrics separate.
- If the issue never reproduces locally, optimize for better production attribution rather than deeper local speculation.

## web-vitals Attribution Build

Import from `web-vitals/attribution` (adds ~1.5KB brotli). Each metric callback gains an `attribution` property with diagnostic data.

### CLS Attribution

Log these fields to identify what shifted:
- `attribution.largestShiftTarget` -- CSS selector of the first element in the largest shift
- `attribution.largestShiftValue` -- score of the single largest shift
- `attribution.largestShiftTime` -- when it happened
- `attribution.loadState` -- document loading state at the time
- `attribution.largestShiftSource.previousRect` / `currentRect` -- element position before/after

**Aggregation strategy**: Rank `largestShiftTarget` selectors across all users to find the elements affecting the most sessions.

### INP Attribution

Log these fields to identify slow interactions:
- `attribution.interactionTarget` -- CSS selector of the element the user interacted with
- `attribution.interactionType` -- `'pointer'` or `'keyboard'`
- `attribution.inputDelay` -- time blocked before handler ran
- `attribution.processingDuration` -- time in event handlers
- `attribution.presentationDelay` -- time from handler end to next paint
- `attribution.longAnimationFrameEntries` -- LoAF entries intersecting the interaction (Chrome 123+), with per-script attribution

### Reporting

Report on `visibilitychange` to `hidden` -- this is the only reliable "final value" signal. Use `navigator.sendBeacon()` for delivery during page unload.

## Long Animation Frames (LoAF)

Chrome 123+. Successor to Long Tasks API. Measures entire animation frames >50ms and provides per-script attribution:
- `scripts[].sourceURL`, `sourceFunctionName`, `sourceCharPosition` -- which code caused the long frame
- `scripts[].forcedStyleAndLayoutDuration` -- time spent on forced reflows within that script

LoAF tells you *which script* caused the jank, not just that a long frame occurred. Integrate with INP attribution (web-vitals provides `longAnimationFrameEntries` on INP callbacks) for full correlation.

Feature-detect: `PerformanceObserver.supportedEntryTypes.includes('long-animation-frame')`

## CrUX (Chrome User Experience Report)

Use CrUX as a smoke detector -- it tells you *which pages* have problems. Then drill into your own RUM for element-level attribution.

- CrUX API provides URL-level and origin-level p75 data for CLS, INP, LCP
- Thresholds: CLS good <= 0.1, INP good <= 200ms
- Updated daily, 28-day rolling window
- Does NOT tell you which elements shifted -- that requires your own telemetry

## Stable Element Identification

CSS selectors in telemetry break when class names contain build hashes (CSS modules, Tailwind JIT). Mitigation:
- Add stable `id` attributes or `data-` attributes to key UI surfaces
- SpeedCurve walks up the DOM tree and stops at the first `id` or `data-sctrack` attribute
- Ensure your critical surfaces have stable selectors that survive deploys

## Non-CLS Jank Telemetry

CLS only captures layout shifts. It misses flicker, remount churn, Suspense flashes, and theme pop-in. For these:

- **React Profiler API**: Wrap key subtrees in `<Profiler onRender={callback}>`. Log when `phase === "mount"` after initial load (unexpected remount) or when `actualDuration` exceeds a threshold.
- **MutationObserver on stable containers**: Count mutations per animation frame. Rapid add/remove cycles are a flicker signature.
- **Suspense fallback mount logging**: Wrap Suspense fallbacks in a component that logs when it mounts. If it mounts for <100ms, that's a flash.

## Tool Landscape

| Tool | CLS Attribution | INP Attribution | LoAF | Session Replay |
|---|---|---|---|---|
| **web-vitals (self-hosted)** | Yes (full) | Yes (full) | Yes (via INP) | No |
| **Vercel Speed Insights** | Element-level | Element-level | No | No |
| **Sentry** | Score only (no element) | Element + subparts | No | Yes |
| **Datadog RUM** | Element-level | Element + subparts | Yes (SDK v6+) | Yes |
| **DebugBear** | Element + frequency | Element + subparts | Yes | No |
| **SpeedCurve** | Element-level | Element + subparts | Yes | No |

Sentry captures CLS as a numeric value but not `largestShiftTarget` or `sources` -- supplement with the web-vitals attribution build if you need element-level CLS diagnosis. Sentry Session Replay correlates Web Vital breadcrumbs with video playback, which can identify shifting elements visually.
