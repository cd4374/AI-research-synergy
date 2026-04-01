# Review Log

> All review rounds, scores, and action items. Reviews are performed by
> an external LLM via Codex MCP — Claude never reviews its own work.

## Format

```markdown
## Review Round {N} — {date}
- **Reviewer**: {model name via Codex MCP}
- **Review Type**: mandatory | triggered
- **Gate**: {gate ID}
- **Artifact Reviewed**: {what was reviewed}
- **Escalation Reason**: none | high_cost | novelty_uncertainty | evaluator_risk | fairness_risk | anomalous_result | review_stall
- **Overall Score**: {score}/10
- **Verdict**: accept | revise | reject

### Dimension Scores
| Dimension | Score |
|-----------|-------|

### Strengths
1. {strength}

### Weaknesses
| # | Issue | Severity | Suggested Fix |
|---|-------|----------|---------------|

### Action Items
- [ ] {actionable item with priority}

### Score Progression
| Round | Score | Delta | Key Change |
|-------|-------|-------|------------|
```

## Reviews

<!-- Reviews will be appended by /ars-10-reviewer -->
