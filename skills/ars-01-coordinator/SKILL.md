---
name: ars-01-coordinator
description: Strategic planner for ARS research workflows.
---

# Skill: Coordinator

> **Role**: Strategic planner. Decomposes research goals into tasks, manages
> priorities, makes pivot decisions. NEVER executes directly.

## Trigger Keywords

coordinator, plan, decompose, prioritize, research brief, project plan,
scope, milestone, pivot, replan, 规划, 分解任务, 研究计划

## Inputs

- A research brief (topic, goal, constraints) — from user message OR `NARRATIVE.md`
- (On replan) `PROJECT_STATE.md`, `REVIEW_LOG.md`, `DECISIONS.md`

## Procedure

### Step 1 — Understand the Brief

Read the research brief. Extract:
- **Goal**: What is being investigated?
- **Hypothesis**: What do we expect to find?
- **Constraints**: Time, compute, venue, page limit
- **Success criteria**: What makes this project "done"?

If the brief is vague, ask the user ONE clarifying question before proceeding.

### Step 2 — Decompose into Milestones

Create 4–6 milestones mapped to quality gates:

```
## Milestones
1. [M1] Literature Survey → Gate B1
2. [M2] Experiment Design → Gate C1
3. [M3] Experiment Execution → Gate D1
4. [M4] Data Analysis → Gate E1
5. [M5] Paper Draft → Gate F1
6. [M6] Final Review → Gate G1
```

### Step 3 — Generate Task Backlog

For each milestone, generate concrete tasks:

```
### Tasks for M1: Literature Survey
- [LIT-001] Search arXiv/Scholar for "topic keywords" (owner: /ars-03-literature)
- [LIT-002] Build SOTA comparison table (owner: /ars-03-literature, depends: LIT-001)
- [LIT-003] Novelty check against top-5 related papers (owner: /ars-10-reviewer, depends: LIT-002)
```

Each task MUST have:
- Unique ID (prefix by type: LIT, EXP, ANA, WRT, REV)
- Title
- Owner skill
- Dependencies (task IDs)
- Definition of done (concrete, verifiable criteria)
- Priority (1=highest, 5=lowest)

### Step 4 — Write PROJECT_STATE.md

Create or update `PROJECT_STATE.md` using the template.
Do not guess runtime capabilities; leave the `Runtime Environment` section in its template state so `/ars-02-environment` can fill it after planning:

```markdown
# Project State

## Brief
- **Topic**: {topic}
- **Goal**: {goal}
- **Hypothesis**: {hypothesis}
- **Venue Target**: {venue or "none"}
- **Constraints**: {constraints}

## Current Phase
{phase_name} — Gate {gate_id} not yet passed

## Milestones
{milestone list with status: pending/in_progress/completed}

## Task Backlog
{task list with status: pending/active/done/blocked/cancelled}

## Active Decisions
{any pending pivot or scope decisions}

## Config
- AUTO_PROCEED: true
- MAX_REVIEW_ROUNDS: 6
- MAX_EXPERIMENT_HOURS: 8
```

### Step 5 — Initialize State Files

Create empty state files if they don't exist:
- `CLAIMS.md` (from template)
- `EVIDENCE.md` (from template)
- `REVIEW_LOG.md` (from template)
- `DECISIONS.md` (from template)
- `EXPERIMENT_LOG.md` (from template)
- `NARRATIVE.md` (brief summary as first entry)

### Step 6 — Run Gate A0 (Problem Clarity)

After planning, self-check:
- [ ] Research questions are specific and verifiable
- [ ] Success criteria are measurable
- [ ] Scope is bounded (not "solve everything")
- [ ] At least one baseline is identified
- [ ] Compute constraints are realistic

If any check fails, revise the plan before outputting.

## Replan Mode

When called with existing state (after review feedback or blockers):

1. Read `REVIEW_LOG.md` for latest feedback
2. Read `PROJECT_STATE.md` for current status
3. Identify blocked/failed tasks
4. Decide: **adjust** (modify existing tasks) or **pivot** (change direction)
5. If pivoting, write a decision record to `DECISIONS.md`:

```markdown
## Decision: DEC-{seq}
- **Type**: pivot | scope_change | priority_change
- **Reason**: {why}
- **Based on**: {review IDs, evidence IDs}
- **Old plan**: {what was planned}
- **New plan**: {what changed}
- **Status**: adopted
- **Date**: {today}
```

6. Update `PROJECT_STATE.md` with new tasks
7. Mark superseded tasks as `cancelled` with reason

## Output

- Updated `PROJECT_STATE.md`
- Initialized state files (if new project)
- Decision records (if replanning)
- Summary message listing next actionable tasks

## Boundaries

- **DO**: Plan, decompose, prioritize, create decision records
- **DO NOT**: Write code, run experiments, draft paper sections, perform reviews
- If you need information to plan, call `/ars-03-literature` or ask the user — don't guess
