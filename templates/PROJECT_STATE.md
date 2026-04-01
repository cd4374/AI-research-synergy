# Project State

## Brief
- **Topic**: {research topic}
- **Goal**: {what we are trying to show or discover}
- **Hypothesis**: {our central hypothesis}
- **Venue Target**: {target venue or "none"}
- **Constraints**: {time, compute, page limit, etc.}
- **Success Criteria**:
  1. {quantitative criterion}
  2. {qualitative criterion}

## Current Phase
Planning — Gate A0 not yet passed

## NOVELTY_STATUS
pending

## Milestones
| # | Milestone | Gate | Status |
|---|-----------|------|--------|
| M1 | Literature Survey | B1 | pending |
| M2 | Experiment Design | C1 | pending |
| M3 | Experiment Execution | D1 | pending |
| M4 | Data Analysis | E1 | pending |
| M5 | Paper Draft | F1 | pending |
| M6 | Final Review | G1 | pending |

## Task Backlog
| ID | Title | Owner | Depends | Priority | Status |
|----|-------|-------|---------|----------|--------|
<!-- Tasks will be added by /ars-01-coordinator -->

## Runtime Environment
- **Target Preference**: auto (priority: local_gpu > local_mps > remote_gpu > local_cpu)
- **Selected Mode**: pending
- **GPU Subset**: pending
- **CUDA_VISIBLE_DEVICES**: pending
- **Conda Strategy**: reuse_existing_first
- **Selected Conda Env**: pending
- **Selected Python**: pending
- **Environment Created**: no
- **Hardware Summary**: pending
- **Software Summary**: pending
- **Remote Summary**: pending
- **Constraints**: pending
- **Implementation Guidance**: pending

## ARS Toolchain
- **Codex MCP**: pending
- **Codex MCP Note**: pending
- **npm/npx**: pending
- **WebSearch**: pending
- **arXiv**: pending
- **Semantic Scholar**: pending
- **Git**: pending
- **Cross-Model Review**: pending
- **Toolchain Checked**: pending

## Checkpoints
- [ ] Phase 1: Planning
- [ ] Phase 2: Literature
- [ ] Phase 3: Experiment Design
- [ ] Phase 4: Experiment Execution
- [ ] Phase 5: Analysis
- [ ] Phase 5.5: Figure Polish
- [ ] Phase 6: Writing
- [ ] Phase 7: Review Loop
- [ ] Phase 8: Final Gate

## Active Decisions
<!-- Pending pivot or scope decisions -->

## Config
- AUTO_PROCEED: true
- MAX_REVIEW_ROUNDS: 6
- MAX_EXPERIMENT_HOURS: 8
- POSITIVE_THRESHOLD: 7

## Review Policy
- DEFAULT_MODE: selective
- MANDATORY_REVIEW_GATES: C1, F1, G1
- TRIGGER_REVIEW_ENABLED: true
- HIGH_COST_EXPERIMENT_THRESHOLD_HOURS: 2
- STAGNATION_TRIGGER_ROUNDS: 2
- TRIGGER_CONDITIONS: high_cost, novelty_uncertainty, anomalous_result, evaluator_risk, fairness_risk, review_stall, high_severity_claim

### Auto-Loop Settings
- AUTO_LOOP_EXPERIMENTS: true
- AUTOLOOP_MAX_ITERATIONS: 50
- AUTOLOOP_TIME_BUDGET_MINUTES: 5
- AUTOLOOP_METRIC_KEY: val_loss
- AUTOLOOP_METRIC_DIRECTION: minimize
