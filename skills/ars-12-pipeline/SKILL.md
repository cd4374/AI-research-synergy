---
name: ars-12-pipeline
description: Orchestrate the end-to-end ARS research pipeline.
---

# Skill: Pipeline

> **Role**: Orchestrate the full research pipeline from brief to final review.
> Calls other skills in sequence. The "sleep mode" entry point.

## Trigger Keywords

pipeline, full pipeline, end to end, sleep mode, auto research, run everything,
全流程, 自动科研, 一键运行

## Usage

```
/ars-12-pipeline "your research topic or brief"
/ars-12-pipeline resume                      # Resume from last checkpoint
/ars-12-pipeline status                      # Show current progress
```

## Configuration

Read from `PROJECT_STATE.md` Config section (or use defaults):

```
AUTO_PROCEED: true          # true = full autopilot, false = pause at gates
MAX_REVIEW_ROUNDS: 6        # Max review-fix cycles before stopping
MAX_EXPERIMENT_HOURS: 8     # Skip experiments exceeding this
POSITIVE_THRESHOLD: 7       # Review score to consider "good enough"
REVIEW_MODEL: codex xhigh   # External model for review
```

## Pipeline Flow

```
START
  │
  ▼
┌─────────────┐
│ /ars-01-coordinator │ ← Parse brief, create plan
└──────┬──────┘
       │
       ▼
┌────────────────┐
│ /ars-02-environment │ ← Detect/select runtime + conda
│    + Gate A0     │
└──────┬─────────┘
       │
       ▼
┌─────────────┐
│ /ars-03-literature │ ← Search, SOTA table, evidence
│  + Gate B1   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ /ars-01-coordinator │ ← Design experiments based on lit findings
│ (replan)     │
│  + Gate C1   │ ← /ars-10-reviewer informs plan quality
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ /ars-04-experiment │ ← Write code, run experiments
│  + Gate D1   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ /ars-07-analysis │ ← Statistics, figures, evidence
│  + Gate E1   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ /ars-09-writer │ ← Draft paper with claim binding
│  + Gate F1   │ ← /ars-10-reviewer critiques draft
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│ Review-Fix Loop   │ ← /ars-10-reviewer scores, /ars-09-writer fixes
│ (max N rounds)    │ │ until score ≥ threshold
└──────┬───────────┘
       │
       ▼
┌─────────────┐
│ /ars-11-gate-check │ ← Gate G1: final verification
│     G1      │
└──────┬──────┘
       │
       ▼
  READY FOR HUMAN
```

## Procedure

### Phase 1 — Planning + Environment Detection

```
1. Call /ars-01-coordinator with the research brief
2. Call /ars-02-environment target=auto (or user-specified target)
3. Verify that `PROJECT_STATE.md` now contains a populated `Runtime Environment` section
4. Call /ars-11-gate-check A0
5. If environment detection or A0 fails → stop or fix the blocker before proceeding
6. Checkpoint: save state
```

### Phase 2 — Literature

```
7. Read tasks from PROJECT_STATE.md (LIT-* tasks)
8. Call /ars-03-literature for each literature task
9. Call /ars-11-gate-check B1
10. If B1 fails → identify what's missing → /ars-03-literature again
11. Checkpoint: save state
```

### Phase 3 — Experiment Design

```
12. Call /ars-01-coordinator (replan) — design experiments based on lit findings
13. Call /ars-10-reviewer on experiment plan (Gate C1 rubric)
14. Call /ars-11-gate-check C1
15. If C1 fails → /ars-01-coordinator revises plan → re-review
16. Checkpoint: save state
```

### Phase 4 — Experiment Execution (Auto-Loop Enabled)

```
17. Read tasks from PROJECT_STATE.md (EXP-* tasks)
18. Call /ars-04-experiment to write initial code and establish baseline
    - Must read selected environment and conda decision from `PROJECT_STATE.md`
    - Must adapt implementation/runtime for local_gpu, local_mps, remote_gpu, or local_cpu
19. Optional: call /ars-05-implementation-review before major runs, custom eval work, or material code changes
    - Use it to catch spec-to-code mismatches, leakage, unfair baselines, or eval drift
20. If AUTO_LOOP_EXPERIMENTS enabled (default: true):
    a. /ars-06-auto-loop mode=experiment — iterate on model/training code
       (Karpathy loop: modify → train → measure → keep/discard)
    b. /ars-06-auto-loop mode=hyperparam — sweep hyperparameters on best code
    c. Best results automatically registered in EVIDENCE.md
21. If AUTO_LOOP_EXPERIMENTS disabled:
    a. For each experiment task: /ars-04-experiment → run → collect
    b. If failed → classify failure → retry or escalate
22. Call /ars-11-gate-check D1
23. If D1 fails → identify incomplete experiments → retry
24. Checkpoint: save state
```

### Phase 5 — Analysis

```
25. Call /ars-07-analysis on all completed experiment results
26. Call /ars-11-gate-check E1
27. If E1 fails → identify gaps → /ars-07-analysis again
28. Checkpoint: save state
```

### Phase 5.5 — Figure Polish (VLM Critique Loop)

```
29. Call /ars-08-figure-polish on all figures in figures/
    - VLM critiques each figure (via Codex MCP or self-review)
    - Auto-loop: fix → re-render → re-critique (max 5 rounds/figure)
    - Apply consistent style across all figures
    - Consider merging subplots for space efficiency
    - Generate figures/FIGURE_NOTES.md with caption drafts
30. Checkpoint: save state
```

### Phase 6 — Writing

```
31. Call /ars-09-writer to draft all paper sections
    (uses figures/FIGURE_NOTES.md for accurate figure references)
32. Call /ars-11-gate-check F1
33. If F1 fails → identify issues (missing evidence, bad citations)
    → /ars-09-writer fixes → recheck
34. Checkpoint: save state
```

### Phase 7 — Review Loop (Auto-Loop Enabled)

```
35. First, run /ars-06-auto-loop mode=writing_polish
    (automated prose quality iteration: de-AI, burstiness, coverage)
36. Then run /ars-06-auto-loop mode=review
    (Codex MCP adversarial review → fix → re-review)
    - Stops when score >= POSITIVE_THRESHOLD or plateau or MAX_REVIEW_ROUNDS
    - Each round: /ars-10-reviewer scores → prioritize weaknesses → fix → re-score
    - Reframing preferred over new experiments (cheaper path)
    - If a weakness requires new experiments:
      → /ars-06-auto-loop mode=experiment for that specific question
      → /ars-07-analysis on new results
      → /ars-09-writer updates draft
      → Continue review loop
37. Checkpoint: save state
```

### Phase 8 — Final Gate

```
38. Call /ars-11-gate-check G1
39. If G1 passes:
    → Update PROJECT_STATE.md: status = ready_for_human_approval
    → Print final summary
40. If G1 fails:
    → List remaining issues
    → Flag for human intervention
```

## Checkpoint & Resume

After each phase, save a checkpoint line in `PROJECT_STATE.md`:

```markdown
## Checkpoints
- [x] Phase 1: Planning — completed {date}
- [x] Phase 2: Literature — completed {date}
- [x] Phase 3: Experiment Design — completed {date}
- [ ] Phase 4: Experiment Execution — IN PROGRESS
- [ ] Phase 5: Analysis
- [ ] Phase 5.5: Figure Polish
- [ ] Phase 6: Writing
- [ ] Phase 7: Review Loop
- [ ] Phase 8: Final Gate
```

On `/ars-12-pipeline resume`: read checkpoints, skip completed phases, resume from
the first incomplete phase.

## Error Recovery

If the pipeline crashes mid-phase:
1. State files (`PROJECT_STATE.md`, etc.) are the recovery mechanism
2. `/ars-12-pipeline resume` picks up where it left off
3. Incomplete experiments can be detected from `EXPERIMENT_LOG.md`
4. No data is lost — everything is in Markdown files

## Final Summary

When the pipeline completes (or stops), print:

```
═══════════════════════════════════════════
  ARS Pipeline Complete
═══════════════════════════════════════════
  Project: {title}
  Final Review Score: {score}/10
  Claims: {covered}/{total} covered ({rate}%)
  Experiments: {completed}/{planned}
  Review Rounds: {rounds}
  Gate G1: {PASSED/FAILED}

  Artifacts:
  - Paper: paper/
  - Code: src/
  - Results: results/
  - Figures: figures/

  Status: {ready_for_human_approval | needs_human_intervention}
═══════════════════════════════════════════
```

## Boundaries

- The pipeline is an **orchestrator** — it calls skills, it does not do work itself
- It respects gate results — no skipping failed gates
- It respects safety limits — no exceeding MAX_REVIEW_ROUNDS or MAX_EXPERIMENT_HOURS
- It saves state after each phase so crashes remain recoverable
