# React Path

Use this path when the UI feels rebuilt, a pane blinks, focus is lost, or a component seems to rerender or remount too often.

## Inspect First

Look for these patterns before adding tools:

- unstable or over-broad `key` usage
- a large subtree keyed on session, route, timestamp, or step
- conditional branch swaps that change component identity
- loading states that replace populated content during refetch
- `useEffect` mount-time correction that visibly changes state after first paint
- parent or context updates that force large subtrees to rerender

## Tool Escalation

Prefer this order:

1. code inspection
2. React DevTools Profiler
3. dev-side helpers such as `react-scan` or `why-did-you-render`

Treat `react-scan` and `why-did-you-render` as supporting tools, not the primary audit surface. They are useful for local confirmation, not for declaring production behavior.

## What To Prove

Try to answer one of these:

- this subtree rerendered more often than expected
- this subtree remounted instead of updating in place
- this loading or effect path replaced visible content

When you can point to the identity boundary, the fix is usually clearer than when you start from generic performance language.
