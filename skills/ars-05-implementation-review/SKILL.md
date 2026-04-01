---
name: ars-05-implementation-review
description: Audit spec-to-code fidelity and evaluation integrity.
---

# Skill: Implementation Review

> **Role**: Review whether code faithfully implements the intended research
> method and evaluation protocol. Uses Codex MCP for adversarial spec-to-code
> review when available. Never implements fixes or runs experiments.

## Trigger Keywords

implementation review, code review, spec-to-code, method fidelity,
idea correctness, leakage check, baseline fairness, ablation review,
eval immutability, silent bug, 实现评审, 代码审查, 实现正确性

## Prerequisites

- Codex MCP configured: `claude mcp add codex -s user -- codex mcp-server`
- If Codex MCP unavailable → state clearly that cross-model review is not possible,
  offer degraded self-review with explicit disclaimer

## Inputs

- Method spec, proposal, pseudocode, or experiment brief
- Relevant code files (`src/`), configs (`configs/`), and eval scripts
- Baseline definitions and comparison setup
- `PROJECT_STATE.md` for current phase and experiment intent
- `REVIEW_LOG.md` and `EXPERIMENT_LOG.md` for prior issues (if present)

## Suggested Invocation

```text
/ars-05-implementation-review
- spec: method summary or pseudocode
- code: src/train.py, src/evaluate.py, src/models/foo.py
- config: configs/main.yaml
- focus: baseline fairness + eval immutability
```

## Procedure

### Step 1 — Assemble Review Context

Gather the minimum context needed to compare the intended method to the actual
implementation:

```
IMPLEMENTATION REVIEW CONTEXT:
- Intended method/spec: {brief, pseudocode, formula, or task description}
- What to review: {specific code path, config, or experiment setup}
- Current phase: {from PROJECT_STATE.md}
- Baselines: {what comparisons are supposed to be fair against}
- Eval protocol: {metric, split, script, checkpoint selection rule}
- Prior implementation concerns: {from REVIEW_LOG.md / EXPERIMENT_LOG.md}
```

Attach the relevant code/config/eval artifacts to review.

### Step 2 — Apply the Implementation Review Rubric

Evaluate against these dimensions:

**1. Method fidelity**
- Does the code actually implement the intended algorithm/mechanism?
- Are key steps from the spec missing, reordered, or changed semantically?

**2. Train/eval alignment**
- Is the optimized objective aligned with the reported metric?
- Are train/val/test paths clearly separated?
- Is checkpoint selection consistent with the stated protocol?

**3. Baseline fairness**
- Are data, preprocessing, compute budget, and tuning effort comparable?
- Is the proposed method using extra information or budget unavailable to baselines?

**4. Leakage and silent bugs**
- Any label leakage, future-information leakage, split contamination, caching leaks?
- Any silent tensor/shape, detach/no_grad, broadcast, or metric bugs that could
  produce misleading results without crashing?

**5. Ablation isolation**
- Does each ablation change only one intended factor?
- Are hidden confounds introduced by config or code-path differences?

**6. Eval immutability**
- Is the evaluation rule fixed across comparisons?
- Has the metric function or eval pipeline been changed in a way that breaks fair comparison?

### Step 3 — Review via Codex MCP or Degraded Self-Review

If Codex MCP is available, submit the context + code to Codex MCP:

```
Use mcp__codex__codex tool with:

Prompt:
"You are a rigorous research implementation auditor.
Review whether the following code faithfully implements the intended method.
Focus on spec-to-code correctness, train/eval alignment, baseline fairness,
leakage, silent bugs, ablation isolation, and eval immutability.

Be critical and concrete. Identify:
- mismatches between the intended method and the implementation
- unfair comparison setup or hidden extra budget
- any leakage or silent bug risks
- ablations that do not isolate a single factor
- evaluation protocol mutations that make results incomparable

For each issue, state why it matters and what should be checked or changed.

Respond in JSON:
{
  'implemented_correctly': 'yes|partial|no',
  'overall_risk': 'low|medium|high',
  'strengths': [str],
  'issues': [
    {
      'issue': str,
      'severity': 'fatal|major|minor',
      'evidence': str,
      'why_it_matters': str,
      'suggested_check_or_fix': str
    }
  ],
  'fairness_concerns': [str],
  'leakage_risks': [str],
  'ablation_concerns': [str],
  'eval_immutability_concerns': [str],
  'verdict': str
}"
```

### Step 4 — Parse and Log Findings

If Codex MCP is unavailable, run the same rubric as a degraded self-review and
state that the result is not cross-model validated.

Only append to `REVIEW_LOG.md` when findings are high-signal, high-severity, or
the user asked for a persistent review record. Otherwise, return findings inline.

When logging, append the result to `REVIEW_LOG.md`:

```markdown
## Implementation Review {N} — {date}
- **Reviewer**: Codex MCP ({model_name})
- **Focus**: spec-to-code correctness
- **Implemented Correctly**: {yes|partial|no}
- **Overall Risk**: {low|medium|high}

### Strengths
{list}

### Issues
| # | Issue | Severity | Why it matters | Suggested check/fix |
|---|-------|----------|----------------|---------------------|
{issues}

### Fairness Concerns
{list}

### Leakage Risks
{list}

### Ablation Concerns
{list}

### Eval Immutability Concerns
{list}
```

### Step 5 — Generate Action Items

Convert findings into actionable follow-ups:
- `fatal` → must fix before trusting the result
- `major` → should fix before major runs or paper claims
- `minor` → add to backlog

Update `PROJECT_STATE.md` with follow-up tasks only for `fatal` or `major` findings when the project is already tracked there.

## Output

- Implementation review entry in `REVIEW_LOG.md`
- Action items added to `PROJECT_STATE.md` when appropriate
- Clear verdict on whether the idea appears faithfully implemented

## Boundaries

- **DO**: Review code, configs, and eval setup against the intended method
- **DO**: Identify fidelity gaps, leakage risks, fairness issues, and silent bugs
- **DO NOT**: Implement fixes, run experiments, reinterpret results, or override gates
- This skill evaluates implementation integrity only; execution belongs to `/ars-04-experiment`
  and broad research critique belongs to `/ars-10-reviewer`
