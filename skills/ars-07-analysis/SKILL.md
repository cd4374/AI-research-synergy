---
name: ars-07-analysis
description: Analyze ARS experiment results and register evidence.
---

# Skill: Analysis

> **Role**: Analyze experiment results. Compute statistics, generate figures,
> identify patterns. Register findings as evidence. Never fabricate data.

## Trigger Keywords

analyze, analysis, results, statistics, figure, plot, chart, compare,
significance, p-value, confidence interval, ablation, 分析, 结果分析, 统计

## Inputs

- Experiment results from `results/` directory
- `EXPERIMENT_LOG.md` for run metadata
- `EVIDENCE.md` for existing evidence
- `CLAIMS.md` for claims needing evidence support

## Procedure

### Step 1 — Inventory Results

Read all `results/*/metrics.json` files. Build a summary:

```markdown
## Available Results
| Exp ID | Method | Metric1 (mean±std) | Metric2 (mean±std) | Seeds | Status |
|--------|--------|-------------------|-------------------|-------|--------|
```

Flag any experiments with fewer than 3 seeds — these need re-running.

### Step 2 — Statistical Tests

For each comparison claim we want to make:

**Paired comparison (our method vs baseline):**
```python
from scipy import stats

# Per-seed results
ours = [seed1_score, seed2_score, seed3_score]
baseline = [seed1_score, seed2_score, seed3_score]

# Paired t-test (if n >= 3 paired observations)
t_stat, p_value = stats.ttest_rel(ours, baseline)

# Also compute: effect size (Cohen's d), 95% CI of difference
diff = [o - b for o, b in zip(ours, baseline)]
mean_diff = np.mean(diff)
ci_95 = stats.t.interval(0.95, df=len(diff)-1,
                          loc=mean_diff,
                          scale=stats.sem(diff))
```

**Rules for statistical claims:**
1. Always report p-values with exact numbers (not just "p < 0.05")
2. Always report confidence intervals
3. With only 3 seeds, acknowledge limited statistical power
4. Never claim "significant" without a test — use "we observe" instead
5. For n < 5, prefer bootstrap CI over t-test

Save analysis code to `src/analysis/` for reproducibility.

### Step 3 — Generate Figures

Create publication-quality figures. Save to `figures/`:

**Required figures (if applicable):**
- `figures/main_comparison.pdf` — bar chart or table comparing methods
- `figures/ablation.pdf` — ablation study results
- `figures/training_curves.pdf` — loss/metric over training steps

**Figure standards:**
- Use matplotlib with a clean style (seaborn or science plot style)
- Font size ≥ 10pt for readability
- Include error bars (std or CI)
- Label axes clearly with units
- Use colorblind-friendly palette
- Save as both PDF (paper) and PNG (preview)

Save figure generation code to `src/analysis/plot_{name}.py`.

### Step 4 — Detect Anomalies

Check for:
- [ ] High variance across seeds (std > 0.3 × mean) → flag as unreliable
- [ ] Results that seem "too good" (> 10% above SOTA) → double-check for bugs
- [ ] Inconsistent trends across seeds → needs more seeds
- [ ] NaN or infinite values → code bug
- [ ] Metric values outside plausible range → code bug

If anomalies found, log them in `NARRATIVE.md` and flag for investigation.

### Step 5 — Write Analysis Summary

Append to `NARRATIVE.md`:

```markdown
## Analysis Summary ({date})

### Main Results
{describe what the numbers show — factually, without overclaiming}

### Statistical Validity
{report test results, confidence intervals, caveats about sample size}

### Ablation Findings
{what each ablation reveals about which components matter}

### Anomalies and Caveats
{anything unexpected, limitations of the analysis}

### Negative Results
{what did NOT work — this is mandatory, never hide negative results}
```

### Step 6 — Register Evidence

For each analyzable finding, add to `EVIDENCE.md`:

```markdown
- [EVD-{seq}] {factual statement about the result}
  - source: statistical_test | experiment_result | figure_reference
  - artifact: figures/{figure_name}.pdf | results/{exp_id}/metrics.json
  - task: {task_id}
  - strength: direct | indirect | weak
  - validation: pending_review
  - stats: {p-value, CI, effect size if applicable}
```

**Evidence strength guidelines:**
- `direct`: Paired statistical test with p < 0.05, or clear quantitative comparison
- `indirect`: Observable trend without formal test, or result from related experiment
- `weak`: Single-seed result, or comparison with different setup

### Step 7 — Map Evidence to Claims

Check `CLAIMS.md`. For each existing claim:
- Can it now be supported by new evidence? → Add link
- Is it contradicted by evidence? → Flag as `needs_revision`
- Is evidence insufficient? → Note what additional experiments are needed
- If the mapping is high-risk, ambiguous, or high-severity → request external review before promoting the claim

## Output

- Analysis code in `src/analysis/`
- Figures in `figures/`
- Analysis summary in `NARRATIVE.md`
- New entries in `EVIDENCE.md`
- Updated links in `CLAIMS.md`

## Boundaries

- **DO**: Compute statistics, generate figures, register evidence, identify patterns
- **DO**: Escalate to external review for risky result-to-claim mappings when needed
- **DO NOT**: Write paper prose, run new experiments, modify experiment code, hide negative results
- If results are inconclusive → say so clearly, suggest additional experiments for `/ars-01-coordinator`
