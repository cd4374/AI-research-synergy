# Experiment Log

> Every experiment run — planned, completed, failed, or skipped.
> Raw record of what happened. Interpretation belongs in NARRATIVE.md.

## Format

```markdown
## {EXP-ID}: {title}
- **Date**: {date}
- **Status**: planned | running | completed | failed | timeout | skipped | cancelled
- **Hypothesis**: {what we expected}
- **Config**: configs/{exp_id}.yaml
- **Code**: src/{entry_point}
- **Seeds**: [42, 123, 456]
- **Estimated GPU-hours**: {estimate}
- **Actual duration**: {wall clock}
- **Server**: local | {server_name}

### Results (if completed)
| Metric | Seed 42 | Seed 123 | Seed 456 | Mean ± Std |
|--------|---------|----------|----------|------------|

### Artifacts
- results/{exp_id}/metrics.json
- results/{exp_id}/train.log

### Failure Info (if failed)
- **Failure class**: infra_error | dependency_error | code_error | method_failure | timeout
- **Error**: {error message or traceback summary}
- **Retry count**: {n}/3
- **Resolution**: {what was done — fixed, retried, escalated, abandoned}

### Notes
{any raw observations — NO interpretation}
```

## Experiments

<!-- Experiments will be logged by /ars-04-experiment -->
