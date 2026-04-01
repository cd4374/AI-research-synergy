---
name: ars-10-reviewer
description: Run cross-model adversarial reviews for ARS artifacts.
---

# Skill: Reviewer

> **Role**: Cross-model adversarial review. Uses Codex MCP to call an external
> LLM (GPT-5.4 xhigh) for evaluation. Claude never grades its own work.

## Trigger Keywords

review, critique, score, evaluate, feedback, weakness, strength,
审稿, 评审, 打分, 评估

## Prerequisites

- Codex MCP configured: `claude mcp add codex -s user -- codex mcp-server`
- If Codex MCP unavailable → state clearly that cross-model review is not possible,
  offer degraded self-review with explicit disclaimer

## Inputs

- What to review: paper draft, experiment plan, code, analysis, or full project
- `PROJECT_STATE.md` for review policy (`MANDATORY_REVIEW_GATES`, trigger conditions, thresholds)
- `CLAIMS.md` for `review_required` and `external_review_status`
- `EXPERIMENT_LOG.md` for `review gate` and `escalation reason`
- `NARRATIVE.md`, `EVIDENCE.md` for context
- `REVIEW_LOG.md` for prior review history
- (Optional) Specific gate to review against

## Procedure

### Step 1 — Assemble Review Context

Gather all relevant materials into a single context package:

```
REVIEW CONTEXT:
- Project goal: {from PROJECT_STATE.md}
- Current phase: {phase}
- What's being reviewed: {specific artifact or milestone}
- Review mode: mandatory | triggered
- Review policy: {mandatory gates, trigger conditions, thresholds}
- Prior review scores: {from REVIEW_LOG.md}
- Key claims: {from CLAIMS.md}
- Claims requiring external review: {all claims with review_required: yes and external_review_status != passed}
- Escalated experiments: {all experiments with review gate != none or escalation reason != none}
- Available evidence: {summary from EVIDENCE.md}
```

Consumption rules:
- If the current gate is listed in `MANDATORY_REVIEW_GATES`, mark the review as `mandatory`
- If any claim has `review_required: yes`, prioritize reviewing those claims first
- If any experiment has an `escalation reason`, focus the review on that risk
- When a review resolves a required external check, update the corresponding claim/review status in outputs

Attach the actual artifact to review (paper draft, code, analysis, etc.).

### Step 2 — Select Review Rubric

Based on what's being reviewed, use the appropriate rubric:

**Feasibility Review (Gate A0):**
```
Evaluate on 1-10 scale:
1. Problem clarity — Is the research question specific and answerable?
2. Scope realism — Can this be done with stated resources?
3. Novelty potential — Is there likely a genuine contribution?
4. Baseline selection — Are baselines appropriate and fair?
5. Risk identification — Are major risks acknowledged?
```

**Novelty Review (Gate B1):**
```
Evaluate on 1-10 scale:
1. Differentiation — How different is this from existing work?
2. SOTA awareness — Are the right papers cited?
3. Gap validity — Does the identified gap actually exist?
4. Contribution clarity — Is the contribution well-defined?
```

**Experiment Plan Review (Gate C1):**
```
Evaluate on 1-10 scale:
1. Hypothesis clarity — Are hypotheses falsifiable?
2. Metric appropriateness — Do metrics measure what we care about?
3. Baseline fairness — Are comparisons fair (same data, same compute)?
4. Statistical plan — Will we have enough runs for statistical validity?
5. Reproducibility — Seeds, environment, data splits documented?
```

**Experiment Results Review (Gate D1):**
```
Evaluate on 1-10 scale:
1. Completeness — Are all planned experiments done?
2. Failure handling — Are failures classified and addressed?
3. Reproducibility — Are results consistent across seeds?
4. Integrity — Do numbers look plausible? Any red flags?
```

**Analysis Review (Gate E1):**
```
Evaluate on 1-10 scale:
1. Statistical rigor — Are tests appropriate for the data?
2. Figure quality — Are figures clear, labeled, publication-ready?
3. No overclaiming — Do claims match the evidence?
4. Negative results — Are they reported honestly?
5. Anomaly acknowledgment — Are unexpected results discussed?
```

**Draft Review (Gate F1):**
```
Evaluate on 1-10 scale:
1. Claim coverage — Does every claim have evidence?
2. Citation accuracy — Are all citations real and correct?
3. Writing quality — Is it clear, concise, well-structured?
4. Limitations — Are limitations substantive (not humble-brag)?
5. Conclusion validity — Does conclusion follow from results?
6. Overall contribution — Would this pass peer review?
```

**Final Review (Gate G1):**
All of the above, plus:
```
7. Package completeness — Code, data, supplementary all present?
8. Reproducibility — Can a reviewer reproduce the main result?
```

### Step 3 — Call External LLM via Codex MCP

Before calling, check `PROJECT_STATE.md` → `ARS Toolchain.Cross-Model Review`:
- If `full` → proceed with Codex MCP as normal
- If `degraded_self_review` → skip Codex MCP, run self-review with explicit disclaimer:
  > ⚠️ **Degraded Mode**: Codex MCP is unavailable (`{Codex MCP status}`). This review
  > was performed by Claude on its own work and does NOT meet the cross-model review
  > requirement. Gate passage in degraded mode is permitted only if this was recorded
  > during `/ars-02-environment` and acknowledged by the user.

Submit the review context + rubric to Codex MCP:

```
Use mcp__codex__codex tool with:

Prompt:
"You are a critical reviewer at a top ML venue ({venue}).
Review the following research artifact with extreme rigor.
Use the provided rubric. Score each dimension 1-10.

Be harsh but constructive. Identify:
- Top 3 strengths
- Top 3 weaknesses (with specific fixes)
- Any fatal flaws (issues that would cause desk-reject)
- Specific experiments or changes that would address each weakness

DO NOT be polite at the expense of honesty.
DO NOT hide weaknesses to give a positive score.

{rubric}

{artifact content}

Respond in JSON:
{
  'overall_score': number,
  'dimension_scores': {dimension: score},
  'strengths': [str],
  'weaknesses': [{'issue': str, 'severity': 'fatal|major|minor', 'fix': str}],
  'missing_experiments': [str],
  'verdict': 'accept|revise|reject',
  'detailed_feedback': str
}"
```

### Step 4 — Parse and Log Review

Parse the external LLM's response. Append to `REVIEW_LOG.md`:

```markdown
## Review Round {N} — {date}
- **Reviewer**: Codex MCP ({model_name})
- **Review Type**: {mandatory|triggered}
- **Gate**: {gate_id}
- **Artifact Reviewed**: {artifact}
- **Escalation Reason**: {none|high_cost|novelty_uncertainty|evaluator_risk|fairness_risk|anomalous_result|review_stall}
- **Overall Score**: {score}/10
- **Verdict**: {accept|revise|reject}

### Dimension Scores
| Dimension | Score |
|-----------|-------|
{scores}

### Strengths
{list}

### Weaknesses
| # | Issue | Severity | Suggested Fix |
|---|-------|----------|---------------|
{weaknesses}

### Missing Experiments
{list}

### Action Items
{prioritized list of what to do next}
```

### Step 5 — Generate Action Items

Convert weaknesses into concrete tasks for `/ars-01-coordinator`:

For each weakness:
- `fatal` → Must fix before proceeding. Block gate passage.
- `major` → Should fix. Create task in `PROJECT_STATE.md`.
- `minor` → Nice to fix. Add to backlog.

Field-consumption updates:
- If a reviewed claim passes external scrutiny, set `external_review_status: passed`
- If a reviewed claim fails or remains unsupported, set `external_review_status: failed` or leave as `pending`
- If a trigger came from an experiment, preserve the `review gate` / `escalation reason` trail in `REVIEW_LOG.md`
- Use `PROJECT_STATE.md` review thresholds to decide whether the review loop should continue or stop

Update `PROJECT_STATE.md` task backlog with new tasks.

### Step 6 — Check Stopping Condition

If this is part of an auto-review loop:
- Score ≥ threshold (default 7/10) → **STOP**, mark gate as passed
- Round ≥ MAX_REVIEW_ROUNDS → **STOP**, flag for human
- Score improving trend → continue
- Score plateaued (< 0.5 improvement for 2 rounds) → **STOP**, reframe strategy

## Output

- Review entry in `REVIEW_LOG.md`
- Action items added to `PROJECT_STATE.md`
- Updated claim review statuses for claims that required external review
- Gate pass/fail status

## Boundaries

- **DO**: Review, score, critique, identify weaknesses, suggest fixes
- **DO NOT**: Implement fixes, run experiments, rewrite the paper, override gates
- The reviewer ONLY evaluates — execution is always someone else's job
