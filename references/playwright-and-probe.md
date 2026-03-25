# Playwright And Probe

Use this as the default reproducible path.

## Goal

Replay one interaction and capture enough evidence to decide whether the problem is identity loss, layout instability, or browser work.

## Steps

1. Reproduce in the production build.
2. Name one interaction precisely.
3. Identify the stable surfaces that should not lose continuity.
4. Prefer an existing local render-stability probe or trace harness if the repo already has one.
5. If no probe exists, add the smallest possible one that records:
   - layout shifts
   - long tasks
   - frame gaps
   - DOM mutation churn
   - selector-level replacement, detach, flicker, empty-visible frames, and scroll jumps
6. Save a JSON or text artifact that can be compared after a fix.

## Stable Surface Examples

Track selectors for surfaces that should feel continuous:

- main layout
- active editor
- transcript
- sidebar
- proof rail
- modal body

The question is not "did the DOM change?" The question is "did this stable surface lose continuity?"

## Signal Interpretation

- low CLS + selector replacement -> likely remount or identity problem
- flicker or empty-visible frames -> hidden/show or async swap problem
- scroll jump -> reset or subtree replacement problem
- long tasks or frame gaps -> main-thread or rendering pressure

## Escalation Rule

If the probe explains the symptom, inspect source and fix it.

If the probe shows instability but not the cause:

- go to `react-path.md` when the issue smells like remount or rerender churn
- go to `browser-path.md` when the issue smells like layout, paint, or compositor work
