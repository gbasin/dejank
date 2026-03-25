# Playwright and Probe

Use this as the default reproducible path.

## Goal

Replay one interaction and capture enough evidence to decide whether the problem is identity loss, layout instability, or browser work.

## Steps

1. Reproduce in the production build.
2. Name one interaction precisely.
3. Identify the stable surfaces that should not lose continuity.
4. Prefer an existing local render-stability probe or trace harness if the repo already has one.
5. If no probe exists, add the smallest possible one that records:
   - layout shifts (with element attribution via `entry.sources`)
   - long animation frames (LoAF, Chrome 123+) -- fall back to long tasks for Firefox/WebKit
   - frame gaps
   - DOM mutation churn
   - selector-level replacement, detach, flicker, empty-visible frames, and scroll jumps
6. Save a JSON or text artifact that can be compared after a fix.

## Stable Surface Examples

Track selectors for surfaces that should feel continuous:

- main layout
- active editor
- transcript
- sidebar
- proof rail
- modal body

The question is not "did the DOM change?" The question is "did this stable surface lose continuity?"

## Probe Signals

### PerformanceObserver Entry Types

**`layout-shift`**: Observe with `{ type: 'layout-shift', buffered: true }`. Each entry has:
- `value` -- shift score
- `hadRecentInput` -- true if within 500ms of user input (exclude from CLS)
- `sources` -- up to 5 shifted elements with `node`, `previousRect`, `currentRect`

Capture `sources[0]?.node` tag name or selector in the report to go from "CLS is 0.25" to "this div shifted 170px because the image above it loaded without dimensions."

**`long-animation-frame`** (LoAF, Chrome 123+): Observe with `{ type: 'long-animation-frame', buffered: true }`. Each entry provides:
- `duration`, `blockingDuration`, `renderStart`, `styleAndLayoutStart`
- `scripts[]` with `sourceURL`, `sourceFunctionName`, `forcedStyleAndLayoutDuration`

Strictly superior to `longtask` -- tells you *which script* caused the jank. Feature-detect: `PerformanceObserver.supportedEntryTypes.includes('long-animation-frame')`. Fall back to `longtask` when unavailable.

**`event`** (for INP, optional): Observe with `{ type: 'event', buffered: true, durationThreshold: 104 }`. Each entry gives `processingStart`, `processingEnd`, `duration`, `interactionId`. Measures input-to-paint latency per interaction.

**`element`** (optional): Add `elementtiming="name"` attribute to critical DOM elements, then observe `{ type: 'element', buffered: true }`. Gives the render timestamp of specific elements -- useful for measuring when a card or panel actually paints.

## Signal Interpretation

- Low CLS + selector replacement -> likely remount or identity problem
- Flicker or empty-visible frames -> hidden/show or async swap problem
- Scroll jump -> reset or subtree replacement problem
- LoAF with high `forcedStyleAndLayoutDuration` -> layout thrashing in scripts
- LoAF with high `blockingDuration` -> main-thread pressure
- Frame gaps without long tasks -> rendering pressure (large style/layout recalc)

## Measurement Pitfalls

- **CLS finalization**: Lab CLS may under-report because the page never "hides." For interaction-scoped measurement (start/stop around one action), this is fine -- you measure shifts during the window, not lifetime CLS.
- **INP requires real input events**: Playwright's `click()` and `type()` generate real input events and trigger `event` timing entries. `page.evaluate(() => button.click())` does NOT -- it bypasses the input pipeline.
- **LCP requires foreground visibility**: LCP will not report in Playwright unless the page is in the foreground and some interaction occurs.
- **`toHaveScreenshot()` cannot catch flicker**: It waits for the screenshot to stabilize before comparing. By design, it misses transient intermediate states.

## CDP Escalation Layer

Playwright exposes full Chrome DevTools Protocol via `page.context().newCDPSession(page)`. Use for deeper signals when the standard probe is insufficient:

**`Performance.getMetrics`**: Take delta snapshots before/after an interaction. Returns cumulative counters including `LayoutCount`, `RecalcStyleCount`, `RecalcStyleDuration` -- tells you how many layout recalculations happened.

**`Page.startScreencast`**: Captures individual rendered frames as images. Diff consecutive frames with `pixelmatch` to detect GPU/paint-level flicker that DOM-level probes miss. Use `{ format: 'png', quality: 80, everyNthFrame: 1 }`.

**Tracing**: `Tracing.start` with `devtools.timeline,blink.user_timing` categories captures the same events Chrome DevTools Performance panel shows. Produces large data volumes -- use only as escalation.

## Browser Launch Flags for Accurate Measurement

- `--disable-background-timer-throttling`
- `--disable-backgrounding-occluded-windows`
- `--disable-renderer-backgrounding`

These prevent Chrome from throttling timers and rendering when the window is not focused.

## Escalation Rule

If the probe explains the symptom, inspect source and fix it.

If the probe shows instability but not the cause:

- Go to [react-path.md](./react-path.md) when the issue smells like remount or rerender churn
- Go to [browser-path.md](./browser-path.md) when the issue smells like layout, paint, or compositor work
