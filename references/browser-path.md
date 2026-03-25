# Browser Path

Use this path when the symptom looks like layout movement, repaint, compositing, or general browser work rather than a pure React identity problem.

## Diagnostic Workflow

### Step 1 -- Live Metrics (zero setup)

Open Chrome DevTools Performance panel. The landing page shows live Core Web Vitals (CLS, INP, LCP) updating in real time as you interact. Color-coded cards indicate good/needs-improvement/poor. If CLS or INP lights up during the suspect interaction, you have a starting point.

### Step 2 -- Record a Trace

1. Performance panel > Record (Cmd+E)
2. Perform the interaction you want to measure
3. Stop recording
4. Check the **Insights sidebar** first -- it auto-surfaces curated findings:
   - "Layout shift culprits" -- identifies the worst shift cluster and the affected DOM elements
   - "Forced reflow" -- highlights layout thrashing (>30ms forced reflows)
   - "Render-blocking requests" -- resources delaying paint

### Step 3 -- Diagnose by Symptom

**For layout shifts / jump / snap:**
- Look at the **Layout Shift Regions** overlay (Rendering tab > check "Layout Shift Regions")
- In the trace, expand the "Experience" track to see layout shift entries
- Use the Layout Instability API `sources` array to identify *which element* shifted:
  - `entry.sources[0].node` -- the shifted element
  - `entry.sources[0].previousRect` / `currentRect` -- where it was and where it went
- Heuristic: the element immediately *preceding* the shifted element is most likely the cause

**For sluggish interaction / stutter:**
- Find the interaction in the **Interactions track**
- Check the three INP sub-parts:
  - **Input delay** -- main thread was blocked before handler ran (look for preceding long tasks)
  - **Processing duration** -- event handler itself is slow (look at script attribution)
  - **Presentation delay** -- rendering work after handler (look for large style/layout recalcs)
- Click "Local duration" to see Long Animation Frame (LoAF) data with per-script attribution

**For paint/repaint storms:**
- Enable **Paint Flashing** (Rendering tab) -- green highlights show areas being repainted
- Large green areas on every frame indicate excessive paint work
- Check if affected elements can use `will-change: transform` or `contain: paint` to isolate paint regions

**For compositor/layer issues:**
- Open the **Layers panel** (Cmd+Shift+P > "Show Layers")
- 3D view shows all compositor layers with dimensions, compositing reasons, memory cost, and paint count
- Excessive layers waste GPU memory; too few layers force main-thread paint

## Key APIs for Deeper Diagnosis

### Long Animation Frames (LoAF) -- replaces Long Tasks

Available Chrome 123+. Strictly superior to the Long Tasks API for attribution.

Each LoAF entry provides:
- `duration`, `renderStart`, `styleAndLayoutStart`, `blockingDuration`
- `scripts[]` with `sourceURL`, `sourceFunctionName`, `sourceCharPosition`, `forcedStyleAndLayoutDuration`

`forcedStyleAndLayoutDuration` on script entries directly identifies layout thrashing -- the time a script spent forcing synchronous reflows.

### Layout Instability API -- element-level shift attribution

Each `LayoutShift` entry's `sources` array contains up to 5 shifted elements, sorted by impact area. Each has `node`, `previousRect`, `currentRect`. An all-zero rect means the element entered/left the viewport.

## CSS Techniques for Prevention

### `content-visibility: auto`

Skips layout, style, and paint for off-screen elements entirely. Up to 7x faster initial render on content-heavy pages.

```css
.list-item {
  content-visibility: auto;
  contain-intrinsic-size: auto 300px; /* prevents scrollbar jitter */
}
```

Baseline browser support since September 2025 (Chrome 85+, Firefox 125+, Safari 18+).

### CSS `contain`

- `contain: content` (layout + paint) -- safe default for independent components (cards, list items, modals). Limits the blast radius of layout changes.
- `contain: strict` (size + layout + paint) -- only when you explicitly set dimensions.

### Compositor-safe animations

Only `transform` and `opacity` run on the compositor thread (GPU-accelerated, no main-thread involvement). Everything else triggers layout or paint.

- Use `scale()` instead of animating `width`/`height`
- Use `translate()` instead of animating `top`/`left`
- In Tailwind: use `transition-transform` or `transition-opacity`, not `transition-all`
- Toggle `will-change` before/after animations; do not leave it permanently set

## Forced Reflow Triggers (Curated)

Reading any of these forces the browser to synchronously compute layout:

**Box metrics**: `offsetLeft/Top/Width/Height`, `clientLeft/Top/Width/Height`, `getBoundingClientRect()`, `getClientRects()`
**Scroll**: `scrollWidth/Height`, `scrollLeft/Top`, `scrollIntoView()`
**Computed style**: `getComputedStyle()` (for dimensions, positioning, transforms, grid properties)
**Content**: `innerText` (computes visible text)
**Focus**: `focus()`, `select()`

If you read any of these then write to the DOM in the same synchronous block, you force a reflow. Batch all reads first, then all writes.

Full list: Paul Irish's "What forces layout/reflow" (https://gist.github.com/paulirish/5d52fb081b3570c81e3a)

## Rendering Tab Quick Reference

| Tool | Overlay Color | What It Shows |
|---|---|---|
| Paint Flashing | Green | Areas being repainted |
| Layout Shift Regions | Purple | Areas where layout shifts occur |
| Layer Borders | Orange/cyan | Compositor layer boundaries |
| Frame Rendering Stats | Top-right | Real-time FPS, GPU raster, memory |
| Scrolling Performance Issues | Highlighted | Scroll listeners harming perf |

## Agent Browser Tools

When you need to interact with a live browser during investigation:

- **chrome-cdp** (`npx skills add pasky/chrome-cdp-skill --all -g`): Quick inspection of a tab the user already has open. Use for screenshots, JS eval, accessibility snapshots, clicking elements. Lightweight CLI -- no server, no Playwright. Good for "look at this tab and tell me what's happening."
- **[dev-browser](https://github.com/SawyerHood/dev-browser)** (`npx skills add sawyerhood/dev-browser --all -g`): Full Playwright-based automation with persistent named pages. Use for multi-step workflows, scripted repro sequences, PerformanceObserver injection, request interception, headless CI runs.

Pick `chrome-cdp` for one-shot inspection, `dev-browser` for scripted investigation.

## Escalation

If the selector-level probe already proved a stable surface was replaced, stay on the React path first.

If the probe shows instability but not the cause, or if the issue looks like geometry or paint behavior, use this browser path. If symptoms remain unclear after a Performance trace, use `chrome://tracing` with `devtools.timeline,blink.user_timing` categories for frame-level detail.
