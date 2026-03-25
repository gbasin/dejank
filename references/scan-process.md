# Scan Process and Output Format

How to execute a Mode A (static analysis) scan and report findings.

## Scope

If no target is specified, scan files changed in the current branch (`git diff --name-only` against the base branch). If a target is given, scan that file/directory and its direct imports.

**Direct importers**: Also consider scanning files that *import* the changed files (one level up). If a parent component's layout depends on a changed child's dimensions, the child change can cause layout shift in the parent. Skip transitive dependents -- layout shift is almost always local (parent-child).

**Shared layout components**: If a changed file is a layout primitive (`<Stack>`, `<Card>`, `<Skeleton>`, etc.), expand scope to all direct importers since any sizing change affects every consumer.

## Process

1. **Identify scope**: Changed files, or target files + direct imports + direct importers.
2. **Read EVERY file in scope**: Glob for all `.tsx`/`.jsx` files in the target, excluding `node_modules/`, test files (`*.test.*`), and type-only files. **Read every single component file** -- do not cherry-pick "interesting-looking" files. Skipping files is the #1 source of missed findings. If the scope contains more than ~30 component files, batch reads in parallel but still read all of them.
3. **Follow local imports one level deep**: For each component file read, check its import statements. If it imports a local component or context provider that is NOT already in scope, read that file too. This catches:
   - Context providers with unstable values (pattern #16)
   - Utility components with layout-affecting behavior
   - Shared hooks that set state in effects
   Do not chase imports into `node_modules` or beyond one level.
4. **Read source directly**: Identify React component boundaries by pattern-matching on source text. Do not try to invoke an AST parser. Reserve AST-based tooling (dependency-cruiser, importree) for scope resolution only.
5. **React Compiler check**: Look for components that may have been silently skipped by the React Compiler (ESLint suppression comments on hook rules, complex try/catch, destructure-then-mutate patterns). These lose automatic memoization and are higher risk for render instability.
6. **Pattern match**: Check each component against ALL patterns in [static-patterns.md](./static-patterns.md).
7. **Assess severity**:
   - **Critical**: Almost always produces visible jank regardless of conditions.
   - **High**: Produces jank under normal usage conditions (user interaction, data loading).
   - **Medium**: Produces jank under specific conditions (slow device, large dataset, rapid interaction).
   - **Low**: Could produce jank, context-dependent -- note the condition.
8. **Note unavoidable patterns**: Some patterns (e.g., textarea auto-resize via write-then-read) are the standard approach with no better alternative. Still note them as "acknowledged, no actionable fix" so the report is complete and the user knows the pattern was seen and evaluated, not overlooked.
9. **Skip false positives**: See guardrails below.

## What to Skip (React Compiler Territory)

Do not duplicate what the React Compiler and its ESLint rules already handle:
- `setState` during render (caught by `react-hooks/set-state-in-render`)
- Manual `useMemo`/`useCallback` suggestions for non-jank-producing code (compiler auto-memoizes)
- Unsafe ref access during render (caught by `react-hooks/refs`)

Focus on patterns the compiler does NOT address: layout shift from missing dimensions, `useEffect` vs `useLayoutEffect` for DOM measurements, missing loading states, conditional rendering of large blocks without space reservation, font flash, async waterfalls.

## Output Format

Group findings by component/file, not by pattern category. For each finding:

```
### [ComponentName] -- path/to/file.tsx:L##

**[Pattern Name]** (Severity)
What the user sees: [one sentence describing the visual impact]
Code: [the specific lines]
Fix: [the specific fix for this instance -- not generic advice, actual code]
```

Lead with severity and visual impact. Include the specific code snippet. Provide a concrete fix. Distinguish "must fix" from "consider fixing" -- mixing these erodes trust.

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
- **Skip files in scope**. Every component file must be read, not just the ones that look relevant. Missing findings overwhelmingly come from unread files, not from misinterpreting read files. If you didn't read it, you can't clear it.
- Flag every conditional render as jank -- most are fine. Only flag when the component has significant visual height, the condition toggles during user interaction, and there is no reserved space.
- Suggest `useMemo` everywhere as a blanket fix -- only where it prevents visible jank. With React Compiler, manual memoization is increasingly unnecessary.
- Ignore the "what the user sees" test -- if you cannot articulate the visual impact, it is probably not jank.
- Recommend `useLayoutEffect` universally -- it blocks paint and creates its own problems if overused.
- Miss `transition-all` in Tailwind -- it is the single most common source of animation jank in Tailwind projects.
- Flag patterns inside generated code, `node_modules`, or vendor directories.
- Flag `useEffect` with empty deps as jank when no DOM measurement or state setter is involved.
- Silently omit "standard but unavoidable" patterns like textarea auto-resize. Note them as acknowledged.
