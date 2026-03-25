# Browser Path

Use this path when the symptom looks like layout movement, repaint, compositing, or general browser work rather than a pure React identity problem.

## Tools

Prefer Chrome DevTools:

- Performance panel for traces, screenshots, layout shifts, and interaction timing
- Rendering tools for paint flashing, layout shift regions, layer borders, and FPS overlays

## What To Look For

- layout shift clusters
- repeated style or layout work during one interaction
- paint-heavy frames
- large invalidation areas
- compositor or layer churn
- long tasks on the main thread

## Decision Rule

If the selector-level probe already proved a stable surface was replaced, stay on the React path first.

If the selector-level probe does not explain the symptom, or if the issue looks like geometry or paint behavior, use this browser path.
