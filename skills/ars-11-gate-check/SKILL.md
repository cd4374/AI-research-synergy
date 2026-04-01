---
name: ars-11-gate-check
description: Verify ARS quality gates without making fixes.
---

# Skill: Gate Check

> **Role**: Verify quality gate conditions. Pure verification — no fixes,
> no overrides. Reports pass or fail with specific violations.

## Trigger Keywords

gate, gate-check, quality gate, check gate, verify gate, 质量门, 检查

## Usage

```
/ars-11-gate-check A0      # Check specific gate
/ars-11-gate-check all     # Check all gates in sequence
/ars-11-gate-check next    # Check the next gate based on current phase
```

## Gate Definitions

### Gate A0 — Problem Clarity
**Trigger**: After `/ars-01-coordinator` produces initial plan

**Conditions** (ALL must pass):
- [ ] `PROJECT_STATE.md` exists and has all required fields
- [ ] `PROJECT_STATE.md` includes a populated `Runtime Environment` section
- [ ] A target environment and final selected mode are recorded
- [ ] A conda decision is recorded (reuse/new, env name, Python version)
- [ ] Research questions are stated as specific, verifiable statements
- [ ] At least one success criterion is quantitative
- [ ] At least one baseline is identified
- [ ] Scope is bounded (explicit out-of-scope listed)
- [ ] Compute budget is estimated and compatible with the selected environment
- [ ] `CLAIMS.md` exists (can be empty)
- [ ] `EVIDENCE.md` exists (can be empty)

---

### Gate B1 — Literature Complete
**Trigger**: After `/ars-03-literature` phase

**Conditions:**
- [ ] `results/sota_table.md` exists with ≥ 3 baselines
- [ ] `EVIDENCE.md` has ≥ 5 literature-sourced evidence items
- [ ] `NARRATIVE.md` contains a "Gap Analysis" section
- [ ] `PROJECT_STATE.md` has `NOVELTY_STATUS` field
- [ ] If `NOVELTY_STATUS: conflict` → `/ars-10-reviewer` must have assessed it
- [ ] All literature tasks in `PROJECT_STATE.md` are marked `done`

---

### Gate C1 — Experiment Plan Approved
**Trigger**: After experiment design

**Conditions:**
- [ ] At least one experiment spec exists in `EXPERIMENT_LOG.md` (status: planned)
- [ ] Each experiment has: hypothesis, metrics, baselines, seed list
- [ ] Each comparison experiment uses the same dataset/split as baselines
- [ ] Statistical plan stated (which tests, how many seeds)
- [ ] Compute estimate within budget
- [ ] `/ars-10-reviewer` has reviewed the experiment plan (score ≥ 6)

---

### Gate D1 — Experiments Complete
**Trigger**: After `/ars-04-experiment` runs

**Conditions:**
- [ ] All planned experiments have status `completed` or `cancelled` (with reason)
- [ ] Each completed experiment has results in `results/` with `metrics.json`
- [ ] Each experiment ran with ≥ 3 seeds
- [ ] Failed experiments are classified with failure class
- [ ] `EVIDENCE.md` has experiment-sourced evidence for each completed run
- [ ] No experiments in `running` or `blocked` status

---

### Gate E1 — Analysis Complete
**Trigger**: After `/ars-07-analysis`

**Conditions:**
- [ ] Statistical tests computed for main comparisons
- [ ] Figures exist in `figures/` for key results
- [ ] Analysis code saved in `src/analysis/` (reproducible)
- [ ] `NARRATIVE.md` has "Analysis Summary" section
- [ ] Negative results documented (not hidden)
- [ ] Anomalies flagged if present
- [ ] Evidence items have `stats` field where applicable

---

### Gate F1 — Draft Complete
**Trigger**: After `/ars-09-writer`

**Conditions (STRICT):**
- [ ] All paper sections exist in `paper/`
- [ ] Limitations section exists and has ≥ 3 substantive items
- [ ] **Claim coverage check:**
  - All claims in Results section: 100% must have evidence
  - All claims in Conclusion: 100% must reference Results claims
  - All claims with `severity_if_wrong: high|critical`: must have `direct` evidence
  - Claims with status `needs_evidence`: ZERO allowed
- [ ] Citation check: no `TODO: verify citation` markers remain
- [ ] `paper/references.bib` exists
- [ ] No fabricated citations (spot-check 3 random references)
- [ ] Figures referenced in text exist in `figures/`
- [ ] `/ars-10-reviewer` has reviewed the draft (score ≥ 6)

---

### Gate G1 — Final Review
**Trigger**: Before human handoff

**Conditions:**
- [ ] ALL previous gates (A0–F1) passed
- [ ] `/ars-10-reviewer` final review score ≥ 7
- [ ] Claim coverage: 100% across all sections
- [ ] Evidence chain intact: every claim → evidence → artifact → experiment
- [ ] Code artifact exists and has `README_run.md`
- [ ] `results/` directory contains all referenced data
- [ ] `EXPERIMENT_LOG.md` is complete
- [ ] No open action items with severity `fatal` or `major`
- [ ] `DECISIONS.md` documents any pivots that occurred

## Procedure

### Step 1 — Identify Gate

From user input or `PROJECT_STATE.md` current phase, determine which gate to check.

### Step 2 — Check Each Condition

For each condition:
1. Check if the file/artifact exists
2. Check if the content meets the requirement
3. Record: ✅ PASS or ❌ FAIL with specific reason

### Step 3 — Compute Claim Coverage (Gates F1 and G1)

```
Read CLAIMS.md:
  total_claims = count all claims
  covered_claims = count claims with ≥ 1 evidence link
  high_severity_covered = count high/critical claims with direct evidence
  needs_evidence_count = count claims with status needs_evidence

  coverage_rate = covered_claims / total_claims
  blocking = needs_evidence_count > 0 OR high_severity not fully covered

Report:
  "Claim Coverage: {covered}/{total} ({rate}%)"
  "High-severity coverage: {high_covered}/{high_total}"
  "Blocking issues: {count}"
```

### Step 4 — Report

Output a structured gate report:

```markdown
## Gate {ID} Check — {date}

### Result: ✅ PASSED | ❌ FAILED

### Conditions
| # | Condition | Status | Details |
|---|-----------|--------|---------|
| 1 | {condition} | ✅/❌ | {details if failed} |

### Claim Coverage (if applicable)
- Total claims: {n}
- Covered: {n} ({rate}%)
- Needs evidence: {n}
- High-severity uncovered: {n}

### Blocking Issues
{list of conditions that must be fixed before gate can pass}

### Next Steps
{what needs to happen to pass this gate}
```

### Step 5 — Update Project State

Update `PROJECT_STATE.md`:
- If passed: advance phase, record gate passage date
- If failed: record failure, list blocking issues

## Boundaries

- **DO**: Check conditions, report pass/fail, compute coverage
- **DO NOT**: Fix issues, override gates, modify artifacts, run experiments
- Gates are binary: pass or fail. No "conditional pass" or "pass with warnings"
- The ONLY entity that can advance a project past a gate is this skill reporting PASS
