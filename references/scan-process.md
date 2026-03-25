# Scan Process and Output Format

How to execute a Mode A (static analysis) scan and report findings.

## Scope

If no target is specified, scan files changed in the current branch (`git diff --name-only` against the base branch). If a target is given, scan that file/directory and its direct imports.

**Direct importers**: Also consider scanning files that *import* the changed files (one level up). If a parent component's layout depends on a changed child's dimensions, the child change can cause layout shift in the parent. Skip transitive dependents -- layout shift is almost always local (parent-child).

**Shared layout components**: If a changed file is a layout primitive (`<Stack>`, `<Card>`, `<Skeleton>`, etc.), expand scope to all direct importers since any sizing change affects every consumer.

## Process

1. **Identify scope**: Changed files, or target files + direct imports + direct importers.
2. **Size check**: Count `.tsx`/`.jsx` files in scope (excluding `node_modules/`, test files `*.test.*`, and type-only files).
   - **≤ 15 component files**: Read all of them. Skip to step 4.
   - **> 15 component files**: Use the grep sweep in step 3 to prioritize reads.
3. **Grep sweep** (large scopes only): Run the signatures in the table below against all files in scope. Rank files by match count -- files hitting multiple signatures are highest priority. Deep-read all files with matches plus any the user specifically called out. Files with zero signature matches get deferred -- note them in the report as "grep-scanned only."
4. **Read and analyze**: For each file being deep-read, identify React component boundaries by pattern-matching on source text. Do not try to invoke an AST parser. Reserve AST-based tooling (dependency-cruiser, importree) for scope resolution only.
5. **Follow local imports one level deep**: For each deep-read file, check its import statements. If it imports a local component or context provider not already in scope, read that file too. This catches:
   - Context providers with unstable values (pattern #16)
   - Utility components with layout-affecting behavior
   - Shared hooks that set state in effects
   Do not chase imports into `node_modules` or beyond one level.
6. **React Compiler check**: Look for components that may have been silently skipped by the React Compiler (ESLint suppression comments on hook rules, complex try/catch, destructure-then-mutate patterns). These lose automatic memoization and are higher risk for render instability.
7. **Pattern match**: Check each component against ALL patterns in [static-patterns.md](./static-patterns.md).
8. **Assess severity**:
   - **Critical**: Almost always produces visible jank regardless of conditions.
   - **High**: Produces jank under normal usage conditions (user interaction, data loading).
   - **Medium**: Produces jank under specific conditions (slow device, large dataset, rapid interaction).
   - **Low**: Could produce jank, context-dependent -- note the condition.
9. **Note unavoidable patterns**: Some patterns (e.g., textarea auto-resize via write-then-read) are the standard approach with no better alternative. Still note them as "acknowledged, no actionable fix" so the report is complete and the user knows the pattern was seen and evaluated, not overlooked.
10. **Skip false positives**: See guardrails below.

### Grep Signatures for Pre-Scan

Run these against `.tsx`/`.jsx` files in scope. Matches identify candidate files for deep reading.

**High-confidence** (match strongly suggests a finding):

| Signature | Pattern |
|---|---|
| `transition-all` | #11 Animating layout properties |
| `key={Math.random` or `key={Date.now` | #5 Unstable keys |
| `.Provider value={{` | #16 Unstable context value |

**Candidate** (match warrants deep reading to confirm):

| Signature | Pattern |
|---|---|
| `useEffect` | #1 Effect-driven init, #2 Derived state, #3 Chained cascade |
| `<img` (check for missing `width`/`height`/`aspect-`) | #6 Unsized async content |
| `<Suspense` or `React.lazy` | #8 Suspense boundary flash |
| `typeof window` or `localStorage` or `sessionStorage` | #9 Hydration mismatch (SSR only) |
| `offsetHeight` or `getBoundingClientRect` or `getComputedStyle` | #10 Layout thrashing |
| `ref={(` | #13 Ref callback remount |
| `@font-face` or `fonts.googleapis` | #15 Font flash |

**Require deep reading** (no reliable grep signature):

| Pattern | Why |
|---|---|
| #4 Conditional mount/unmount | Most conditional renders are fine -- context needed |
| #7 Skeleton/placeholder absence | Structural comparison between loading and loaded states |
| #12 Missing startTransition | Requires understanding render cost |
| #14 Z-index/stacking context flash | Requires understanding conditional styling |
| #17 Component defined inside component | Requires understanding scope nesting |
| #18 Async waterfall | Requires understanding data dependency chain |

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
- Files grep-scanned only: N (if applicable -- list them for transparency)
```

## Guardrails

Do not:
- **Skip files in scope**. For scopes ≤ 15 files, every component file must be read. For larger scopes, every file must be at least grep-scanned. If you didn't read or scan it, you can't clear it.
- Flag every conditional render as jank -- most are fine. Only flag when the component has significant visual height, the condition toggles during user interaction, and there is no reserved space.
- Suggest `useMemo` everywhere as a blanket fix -- only where it prevents visible jank. With React Compiler, manual memoization is increasingly unnecessary.
- Ignore the "what the user sees" test -- if you cannot articulate the visual impact, it is probably not jank.
- Recommend `useLayoutEffect` universally -- it blocks paint and creates its own problems if overused.
- Miss `transition-all` in Tailwind -- it is the single most common source of animation jank in Tailwind projects.
- Flag patterns inside generated code, `node_modules`, or vendor directories.
- Flag `useEffect` with empty deps as jank when no DOM measurement or state setter is involved.
- Silently omit "standard but unavoidable" patterns like textarea auto-resize. Note them as acknowledged.
