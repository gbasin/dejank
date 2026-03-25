# Field Path

Use this path when the issue is intermittent, route-specific, or only visible in production.

## Approach

1. Add telemetry before guessing.
2. Prefer `web-vitals` attribution for CLS and interaction issues.
3. Log enough metadata to tie a bad metric back to a route, interaction, or UI state.

## Rules

- do not claim a local fix solved a prod-only problem without field evidence
- keep local repros and field metrics separate
- if the issue never reproduces locally, optimize for better production attribution rather than deeper local speculation
