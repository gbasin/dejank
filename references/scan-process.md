# Scan Process and Output Format

How to execute a Mode A (static analysis) scan and report findings.

## Scope

If no target is specified, scan files changed in the current branch (`git diff --name-only` against the base branch). If a target is given, scan that file/directory and its direct imports.

## Process

1. **Identify scope**: Changed files, or target files + their direct imports.
2. **Parse each file**: Read the source, identify React component boundaries (function components, class components).
3. **Pattern match**: Check each component against ALL patterns in [static-patterns.md](./static-patterns.md).
4. **Assess severity**:
   - **Critical**: Almost always produces visible jank regardless of conditions.
   - **High**: Produces jank under normal usage conditions (user interaction, data loading).
   - **Medium**: Produces jank under specific conditions (slow device, large dataset, rapid interaction).
   - **Low**: Could produce jank, context-dependent -- note the condition.
5. **Skip false positives**: If a pattern is present but mitigated (e.g., conditional render inside a `Suspense` with `startTransition`, or `useLayoutEffect` instead of `useEffect`), note the mitigation and move on.

## Output Format

Group findings by component/file, not by pattern category. For each finding:

```
### [ComponentName] -- path/to/file.tsx:L##

**[Pattern Name]** (Severity)
What the user sees: [one sentence]
Code: [the specific lines]
Fix: [the specific fix for this instance -- not generic advice, actual code]
```

End with a summary:

```
## Summary
- Critical: N findings (must fix -- users definitely see these)
- High: N findings (should fix -- users likely see these)
- Medium: N findings (consider fixing -- visible under certain conditions)
- Low: N findings (optional -- minor or context-dependent)
```

## Guardrails

Do not:
- Flag every conditional render as jank -- most are fine
- Suggest `useMemo` everywhere as a blanket fix -- only where it prevents visible jank
- Ignore the "what the user sees" test -- if you cannot articulate the visual impact, it is probably not jank
- Recommend `useLayoutEffect` universally -- it blocks paint and creates its own problems if overused
- Miss `transition-all` in Tailwind -- it is the single most common source of animation jank in Tailwind projects
