# dejank

Detect and diagnose visual jank in React UIs. Static analysis of anti-patterns and runtime investigation of specific symptoms.

## Install

```bash
npx skills add gbasin/dejank --all -g
```

## What it does

React's commit model means state changes paint across multiple frames. Intermediate states leak into what the user sees as flicker, layout shift, or flash. These issues don't throw errors, pass tests, and satisfy type-checkers — but users feel them.

This skill operates in two modes based on a decision tree:

**Mode A — Static Analysis** (pre-merge review, no specific complaint)

Scans React components against 14 anti-patterns that produce visual jank:

| # | Pattern | Severity |
|---|---------|----------|
| 1 | Effect-driven state initialization | Critical |
| 2 | Derived state in effects | High |
| 3 | Chained effect cascade | High |
| 4 | Conditional mount/unmount | High |
| 5 | Unstable key props | High |
| 6 | Unsized async content (layout shift) | Critical |
| 7 | Skeleton/placeholder absence | Medium |
| 8 | Suspense boundary flash | Medium |
| 9 | Hydration mismatch sources | Critical (SSR) |
| 10 | Layout thrashing in event handlers | Medium |
| 11 | Animating layout properties | Medium |
| 12 | Missing `startTransition` for expensive updates | Low |
| 13 | Ref callback remount | Medium |
| 14 | Z-index / stacking context flash | Low |

**Mode B — Runtime Investigation** (user reports a specific visual symptom)

Classifies the symptom, then routes to the lightest investigation path that can explain it:

- Playwright probe for reproducible first-pass evidence
- React DevTools for remount/rerender churn
- Chrome Performance for layout, paint, compositor issues
- Production telemetry for intermittent/prod-only problems

## Structure

```
SKILL.md                          Decision tree entrypoint (91 lines)
agents/openai.yaml                Codex UI metadata
references/
  static-patterns.md              14 anti-pattern catalog
  scan-process.md                 Scope, severity, output format, guardrails
  playwright-and-probe.md         Runtime: Playwright probe path
  react-path.md                   Runtime: React DevTools investigation
  browser-path.md                 Runtime: Chrome Performance investigation
  field-path.md                   Runtime: production telemetry
  tooling-and-signals.md          Tool decision matrix + budgets
```

## Agent support

Harness-agnostic. Works with any agent that follows the [skills convention](https://skills.sh):

- **Claude Code**: `/dejank` or `/dejank src/components/`
- **Codex**: `$dejank` or `$dejank src/components/`
- **Others**: Discovered via root `SKILL.md` with standard frontmatter

## Credits

Combines two independently developed skills:
- Static analysis catalog from a Claude Code session (Opus 4.6)
- Runtime diagnostic workflow from a Codex session (GPT-5.4)
