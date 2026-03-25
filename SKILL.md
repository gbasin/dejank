---
name: dejank
description: Detect and diagnose visual jank in React UIs. Covers two modes -- static analysis of code patterns that cause flicker, layout shift, and flash of intermediate state (for pre-merge review or preventive sweeps), and runtime investigation of specific visual symptoms (flicker, blink, jump, stutter, remount churn, scroll reset, post-paint correction). Use when the user mentions jank, something feeling off, flashing, blinking, layout shifting, full repaints, or wants a render stability audit.
---

Find and fix visual instability in React and browser UIs. This skill operates in two modes depending on what is known at invocation time.

## Decision Tree

Determine the mode before doing any work:

**Is there a specific visual complaint?** (The user describes something they can see: flicker, blink, flash, jump, stutter, pop-in, scroll reset, focus loss, "the whole pane feels rebuilt.")

- **Yes** -- go to **Mode B: Runtime Investigation**
- **No** (general review, PR check, preventive sweep, or just a file/directory target) -- go to **Mode A: Static Analysis**

If unsure, start with Mode A. Its findings often explain the complaint without runtime tooling.

---

## Mode A -- Static Analysis

Use when: reviewing a PR, scanning changed files, doing a preventive sweep, or no specific symptom is reported.

1. Read [references/static-patterns.md](./references/static-patterns.md) for the 14-pattern catalog.
2. Read [references/scan-process.md](./references/scan-process.md) for scope rules, severity rubric, output format, and guardrails.
3. Identify scope: changed files in the current branch (`git diff --name-only` against base), or the user-specified target and its direct imports.
4. Scan each React component against all 14 patterns.
5. Report findings grouped by component, ranked by severity.

Stop here unless a finding warrants runtime confirmation or the user escalates.

---

## Mode B -- Runtime Investigation

Use when: the user reports something visible -- flicker, blink, flash, jump, snap, stutter, pop-in, scroll reset, focus loss, or "the whole pane feels rebuilt."

### Step 1 -- Classify the symptom

| Bucket | Symptoms |
|---|---|
| `flash / blink` | subtree replaced, hidden, or briefly emptied |
| `jump / snap` | layout shifts, size changes, scroll resets |
| `sticky / stutter` | long tasks, frame gaps, render storms |
| `pop in` | post-paint correction from effects, fonts, images, async state |
| `whole pane rebuilt` | keyed remount or identity loss |

Anchor each finding: **Which stable surface lost identity, visibility, position, or continuity during this interaction?**

### Step 2 -- Pick the lightest investigation path

- Reproducible, need first-pass evidence: [references/playwright-and-probe.md](./references/playwright-and-probe.md)
- Smells like remount or rerender churn: [references/react-path.md](./references/react-path.md)
- Smells like layout, paint, or compositor: [references/browser-path.md](./references/browser-path.md)
- Intermittent or prod-only: [references/field-path.md](./references/field-path.md)
- Need the tool decision matrix or budgets: [references/tooling-and-signals.md](./references/tooling-and-signals.md)

Do not start with a heavyweight trace unless the lighter pass fails to explain the symptom.

### Step 3 -- Escalate

Each reference file contains its own escalation rules. Move to the next layer only when the current one is insufficient.

---

## Hybrid: Static + Runtime

When a runtime complaint maps to a static pattern (e.g., "it flashes on load" and you find effect-driven state initialization), run Mode A on the relevant files first. If the static finding explains the symptom, fix it without runtime tooling. If not, proceed to Mode B.

---

## Output

**Mode A**: Findings grouped by component. Each finding includes pattern name, severity, what the user sees, code location, and a specific fix (not generic advice). End with a severity summary. Full format in [references/scan-process.md](./references/scan-process.md).

**Mode B**: Report in this order:

1. **Interaction**: what was tested
2. **Classification**: remount, layout, loading, post-paint, or paint/composite
3. **Evidence**: concrete metrics, trace signals, or selector replacement findings
4. **Likely cause**: highest-confidence explanation
5. **Next action**: code fix, deeper trace, or telemetry

If you create a reusable probe or report during the audit, keep it opt-in unless the repo already budgets against it.

## Scope

This skill covers the narrow band of issues that are **visual, temporal, and caused by React's render model or browser rendering** -- things that make a UI feel janky even when everything else is correct.

It does NOT cover runtime performance (slow renders, memory leaks), accessibility, design quality, or error handling. Defer those to the appropriate specialized skill.
