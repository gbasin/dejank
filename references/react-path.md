# React Path

Use this path when the UI feels rebuilt, a pane blinks, focus is lost, or a component seems to rerender or remount too often.

## Inspect First

Look for these patterns before adding tools:

- Unstable or over-broad `key` usage
- A large subtree keyed on session, route, timestamp, or step
- Conditional branch swaps that change component identity
- Loading states that replace populated content during refetch
- `useEffect` mount-time correction that visibly changes state after first paint
- Parent or context updates that force large subtrees to rerender
- Component defined inside another component's render body (full remount every parent render)

## React Compiler Check

Before deep investigation, check if the suspect component is compiled:

- **DevTools sparkle badge**: Successfully compiled components show a sparkle icon in React DevTools. Absent badge means the compiler bailed out -- the component loses automatic memoization and may exhibit instability that surrounding compiled components do not.
- **Enable `react-hooks/todo` as error in ESLint**: This surfaces silent compilation failures. One team found 100+ components silently un-optimized.
- **Common bail-out causes**: destructuring + mutating props, complex try/catch blocks, ESLint suppression comments on any hook rule.

## Tool Escalation

### Tier 1 -- Zero setup
**React Scan** browser extension (Chrome Web Store). Hooks into React's fiber tree to visually highlight components as they render. Shows unnecessary renders, "Memoizable" tags, FPS drops during interactions. Use `getReport()` for programmatic render count data.

### Tier 2 -- Dev build
**React Performance Tracks** (React 19.2+, Chrome DevTools Performance panel). Shows:
- Four priority subtracks (Blocking, Transition, Suspense, Idle) with Update/Render/Commit/Effects phases
- Components flamegraph with yellow **Mount** and **Unmount** badges -- a component that unmounts then mounts is a remount
- Changed Props inspection (dev builds) -- click any render entry to see exactly which props changed
- Effect duration flamegraph -- shows effect execution time and which effects triggered further updates
- Cascading update detection -- shows which component scheduled an update during render

This replaces the old standalone Profiler tab for most render stability investigations.

### Tier 3 -- Code instrumentation
**`<Profiler>` component** for programmatic detection:
- `phase` parameter distinguishes `"mount"` / `"update"` / `"nested-update"`
- `"nested-update"` = a layout effect triggered a state change, causing a second commit before paint (cascading update)
- Two `"mount"` phases for the same id in a short window = unexpected remount
- Wire into E2E probes for CI-level render budget enforcement

**react-render-tracker** for fiber-level mount/update/unmount event logs across the full component tree over time.

### Tier 4 -- Specialized
**why-did-you-render** for deep prop/state equality checks. Shows exactly what changed in context/state/hooks using deep-equality comparison. **Incompatible with React Compiler** -- do not use in compiler-enabled projects. Use React Scan instead.

## Relevant React 19 APIs

**`useEffectEvent`** (stable in React 19.2): Extracts non-reactive logic from effects. Use when an effect re-synchronizes unnecessarily because its callback references frequently-changing values (e.g., theme, locale). The wrapped callback always reads the latest values but does not cause the effect to re-run.

**`<Activity>`** (React 19.2): For "whole pane feels rebuilt" symptoms on show/hide toggles. `<Activity mode="hidden">` unmounts effects and deprioritizes rendering while preserving React state and DOM. Performance Tracks show Reconnect/Disconnect events for Activity components.

**`startTransition`**: Prevents Suspense boundaries from re-showing fallbacks for already-revealed content. Use when navigating between views triggers a brief fallback flash.

## What To Prove

Try to answer one of these:

- This subtree rerendered more often than expected
- This subtree remounted instead of updating in place
- This loading or effect path replaced visible content
- This effect triggered a cascading update (`nested-update` phase)

When you can point to the identity boundary, the fix is usually clearer than when you start from generic performance language.

## Strict Mode Note

Always reproduce against a production build. React Strict Mode double-invokes component bodies and effects in dev, inflating all render/commit counts 2x. However, develop *with* Strict Mode on -- its double effect execution catches non-idempotent effects that are often the same effects causing mount-time jank.
