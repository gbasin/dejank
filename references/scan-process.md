# Scan Process and Output Format

How to execute a Mode A (static analysis) scan and report findings.

## Scope

If no target is specified, scan files changed in the current branch (`git diff --name-only` against the base branch). If a target is given, scan that file/directory and its direct imports.

**Direct importers**: Also consider scanning files that *import* the changed files (one level up). If a parent component's layout depends on a changed child's dimensions, the child change can cause layout shift in the parent. Skip transitive dependents -- layout shift is almost always local (parent-child).

**Shared layout components**: If a changed file is a layout primitive (`<Stack>`, `<Card>`, `<Skeleton>`, etc.), expand scope to all direct importers since any sizing change affects every consumer.

## Process

1. **Identify scope**: Changed files, or target files + direct imports + direct importers.
2. **Read source directly**: Read the source code and identify React component boundaries. Do not try to invoke an AST parser -- pattern-match on the source text. Reserve AST-based tooling (dependency-cruiser, importree) for scope resolution only.
3. **React Compiler check**: Look for components that may have been silently skipped by the React Compiler (ESLint suppression comments on hook rules, complex try/catch, destructure-then-mutate patterns). These lose automatic memoization and are higher risk for render instability.
4. **Pattern match**: Check each component against ALL 18 patterns in [static-patterns.md](./static-patterns.md).
5. **Assess severity**:
   - **Critical**: Almost always produces visible jank regardless of conditions.
   - **High**: Produces jank under normal usage conditions (user interaction, data loading).
   - **Medium**: Produces jank under specific conditions (slow device, large dataset, rapid interaction).
   - **Low**: Could produce jank, context-dependent -- note the condition.
6. **Skip false positives**: See guardrails below.

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
- Flag every conditional render as jank -- most are fine. Only flag when the component has significant visual height, the condition toggles during user interaction, and there is no reserved space.
- Suggest `useMemo` everywhere as a blanket fix -- only where it prevents visible jank. With React Compiler, manual memoization is increasingly unnecessary.
- Ignore the "what the user sees" test -- if you cannot articulate the visual impact, it is probably not jank.
- Recommend `useLayoutEffect` universally -- it blocks paint and creates its own problems if overused.
- Miss `transition-all` in Tailwind -- it is the single most common source of animation jank in Tailwind projects.
- Flag patterns inside generated code, `node_modules`, or vendor directories.
- Flag `useEffect` with empty deps as jank when no DOM measurement or state setter is involved.
