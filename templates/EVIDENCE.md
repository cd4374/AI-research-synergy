# Evidence Registry

> Every piece of evidence that supports a claim. Each item traces to a
> concrete artifact (paper, experiment result, statistical test, figure).

## Format

```
- [EVD-{seq}] {claimable factual statement}
  - source: literature | experiment | statistical_test | figure_reference | human_note
  - artifact: {file path to supporting artifact}
  - paper: {[P-xxx] title, if literature source}
  - task: {task ID that produced this evidence}
  - strength: direct | indirect | weak
  - validation: pending_review | approved | rejected | superseded
  - stats: {p-value, CI, effect size — if applicable}
```

## Strength Guidelines

| Strength | Meaning | Example |
|----------|---------|---------|
| direct | Formal test with p<0.05, or exact quantitative match | Paired t-test result |
| indirect | Observable trend, related experiment | Same method on different dataset |
| weak | Single seed, different setup, or anecdotal | Preliminary run |

## Evidence Items

<!-- Evidence will be registered by /ars-03-literature, /ars-04-experiment, and /ars-07-analysis -->
