---
name: ars-06-auto-loop
description: Run Karpathy-style iterative improvement loops for ARS phases.
---

# Skill: Auto-Loop

> **Role**: Generalized Karpathy autoresearch loop. Apply "modify → run → measure →
> keep/discard → repeat" to any ARS phase that has a measurable fitness metric.
>
> Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch):
> make a change, measure if it helped, keep or revert, never stop.

## Trigger Keywords

autoloop, auto-loop, autoresearch, iterate, improve loop, optimize,
自动迭代, 自动优化, 迭代改进

## Core Pattern

```
LOOP FOREVER (until max_iterations or human interrupt):
  1. Assess current state → identify what to try
  2. Make ONE change (commit)
  3. Run the evaluation (fixed budget)
  4. Measure the metric
  5. If improved → KEEP (advance)
  6. If not improved → DISCARD (revert)
  7. Log result to TSV
  8. Pick next idea → go to 1
```

**Key principles from Karpathy:**
- Fixed evaluation budget (time or compute) — makes runs comparable
- One metric as ground truth — no subjective judgment
- Keep or discard, binary decision — no "maybe"
- Never stop to ask — the human might be asleep
- If stuck, think harder, try radical changes — don't plateau

## Where Auto-Loop Applies in ARS

Not every phase benefits from this pattern. It works when there's a **measurable
metric** and **iterable artifact**:

| Phase | Metric | Iterable Artifact | Applies? |
|-------|--------|-------------------|----------|
| Literature | — | — | ❌ No (search is one-shot, not iterative) |
| Experiment Code | val_loss, accuracy, etc. | `train.py`, model code | ✅ **YES** — classic autoresearch |
| Hyperparameter Tuning | val_loss, accuracy | config.yaml | ✅ **YES** — grid/random/agent search |
| Analysis Figures | — | — | ❌ No (one-shot extraction) |
| Paper Draft | review score (1-10) | paper sections | ✅ **YES** — review loop |
| Paper Writing Quality | de-AI score, readability | prose | ✅ **YES** — polish loop |
| Statistical Robustness | CI width, p-value | seed count, test choice | ⚠️ Maybe (bounded improvement) |

## Mode 1: Experiment Auto-Loop

> The classic Karpathy use case. Iterate on model/training code to minimize a metric.

### Setup

```
Auto-loop config:
  mode: experiment
  target_file: src/train.py          # The file the agent modifies
  eval_command: python src/evaluate.py --results_dir results/autoloop/
  metric_file: results/autoloop/metrics.json
  metric_key: val_loss               # JSON key to extract
  metric_direction: minimize          # minimize | maximize
  time_budget_minutes: 5              # Fixed per run
  max_iterations: 50                  # Safety cap
  branch: autoloop/{date}            # Git branch for tracking
  cuda_visible_devices: {value from PROJECT_STATE.md CUDA_VISIBLE_DEVICES, or unset}
```

**GPU Subset**: before running any training command, read `CUDA_VISIBLE_DEVICES` from
`PROJECT_STATE.md → Runtime Environment`. If the value is not `unset`, prepend it to
every `python` and `conda run` call in the loop. Propagate the same value inside
remote ssh/screen blocks.

### Procedure

**1. Initialize**
```bash
git checkout -b autoloop/{date}
```

Run baseline as-is. Record in `results/autoloop/results.tsv`:
```
commit	val_loss	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
```

**2. Loop**

```
REPEAT:
  a. Review current train.py and results.tsv history
  b. Generate ONE idea (architecture, optimizer, hyperparameter, etc.)
     - Prefer ideas that are DIFFERENT from recent attempts
     - If last 3 attempts all failed, try something radical
     - Simplicity criterion: if improvement is tiny but adds complexity, discard
  c. Edit src/train.py
  d. git commit -m "autoloop: {short description}"
  e. Run: CUDA_VISIBLE_DEVICES={gpu_subset} timeout {budget}m python src/train.py > run.log 2>&1
     (omit CUDA_VISIBLE_DEVICES prefix if GPU Subset is all/unset)
  f. Extract: grep "^val_loss:" run.log
     - If empty → crash. Read tail -50 run.log. Fix if trivial, else skip.
  g. Record in results.tsv
  h. If val_loss improved:
     → KEEP commit (advance branch)
     → Log: "KEEP: {description} ({old} → {new})"
  i. If val_loss same or worse:
     → git reset --hard HEAD~1
     → Log: "DISCARD: {description} ({result})"
  j. NEVER ask human. NEVER stop. Continue until max_iterations or interrupted.
```

**3. Register Results**

After the loop ends (max iterations or interrupt):
- Best result → register in `EVIDENCE.md`
- Write summary of what worked/didn't to `NARRATIVE.md`
- Update `EXPERIMENT_LOG.md` with the autoloop session

## Mode 2: Review Auto-Loop

> Iterate on paper quality. External LLM scores → fix weaknesses → re-score.

### Setup

```
Auto-loop config:
  mode: review
  target_files: paper/*.md             # Files the agent modifies
  eval_tool: codex_mcp                 # External reviewer
  metric_key: overall_score            # From review JSON
  metric_direction: maximize
  threshold: 7                         # Stop when score >= this
  max_iterations: 6                    # MAX_REVIEW_ROUNDS
```

### Procedure

```
REPEAT (max MAX_REVIEW_ROUNDS):
  a. Submit current draft to /ars-10-reviewer (Codex MCP)
  b. Parse score and weaknesses
  c. IF score >= threshold → STOP (success)
  d. IF score plateau (< 0.5 gain for 2 rounds) → STOP (diminishing returns)
  e. Prioritize weaknesses by severity (fatal > major > minor)
  f. For each addressable weakness:
     - If "reframe/rewrite" → edit paper section directly
     - If "missing experiment" → flag for Mode 1, skip in this loop
     - If "missing evidence" → flag for /ars-07-analysis
  g. Prefer reframing over new experiments (cheaper path)
  h. After fixes, DO NOT self-evaluate. Go to (a) for external re-review.
  i. Log round in REVIEW_LOG.md
```

**Critical rules:**
- Never hide weaknesses to game a higher score
- Never skip the external review step (no self-grading)
- If a weakness requires a new experiment, don't fake it — flag and exit

## Mode 3: Writing Polish Loop

> Iterate on prose quality with an automated scoring function.

### Setup

```
Auto-loop config:
  mode: writing_polish
  target_files: paper/*.md
  eval_script: src/analysis/writing_score.py
  metric_key: composite_score
  metric_direction: maximize
  max_iterations: 10
```

### Scoring Function

Create `src/analysis/writing_score.py` that checks:

```python
def score_writing(paper_dir: str) -> dict:
    scores = {}

    # 1. AI word detection (penalize: delve, pivotal, landscape, etc.)
    ai_words = count_ai_patterns(text)
    scores["de_ai"] = max(0, 10 - ai_words)

    # 2. Sentence length variance (penalize uniform length)
    lengths = [len(s.split()) for s in sentences]
    scores["burstiness"] = min(10, np.std(lengths) / 3)

    # 3. Claim coverage (from CLAIMS.md)
    covered, total = count_claim_coverage()
    scores["coverage"] = (covered / total) * 10 if total > 0 else 10

    # 4. Citation density (flag sections with claims but no citations)
    scores["citation_density"] = check_citation_density(text)

    # 5. Limitations substantiveness
    scores["limitations"] = score_limitations_section(text)

    composite = sum(scores.values()) / len(scores)
    return {"composite_score": composite, **scores}
```

### Procedure

```
REPEAT:
  a. Run writing_score.py → get composite_score
  b. Identify lowest-scoring dimension
  c. Make targeted edit to improve THAT dimension
     - Low de_ai → rewrite sentences with flagged AI words
     - Low burstiness → vary sentence lengths
     - Low coverage → add claim references
     - Low citation_density → add citations to unsupported claims
     - Low limitations → expand limitations section
  d. Re-run writing_score.py
  e. If improved → KEEP
  f. If not → REVERT
  g. Continue to next lowest dimension
```

## Mode 4: Hyperparameter Auto-Loop

> Grid/random search with the autoresearch keep/discard pattern.

### Setup

```
Auto-loop config:
  mode: hyperparam
  target_file: configs/{exp_id}.yaml
  eval_command: CUDA_VISIBLE_DEVICES={gpu_subset} python src/train.py --config configs/{exp_id}.yaml
  metric_key: val_accuracy
  metric_direction: maximize
  search_space:
    lr: [0.0001, 0.0003, 0.001, 0.003, 0.01]
    batch_size: [16, 32, 64, 128]
    weight_decay: [0, 0.01, 0.1]
    warmup_ratio: [0, 0.05, 0.1]
  strategy: random  # random | grid | agent
  max_iterations: 30
```

### Procedure

```
REPEAT:
  a. Select next hyperparameter configuration
     - random: sample randomly from search space
     - grid: systematic sweep
     - agent: LLM picks based on prior results (like Karpathy's approach)
  b. Write config to configs/{exp_id}.yaml
  c. Run training with fixed time budget
  d. Measure metric
  e. Record in results.tsv
  f. If best so far → save as configs/best_config.yaml
  g. Continue
```

## Results Tracking

All autoloop sessions log to `results/autoloop/results.tsv`:

```
commit	metric	memory_gb	status	description	mode	iteration
```

Additionally, append a session summary to `EXPERIMENT_LOG.md`:

```markdown
## AUTOLOOP-{date}: {mode} session
- **Mode**: experiment | review | writing_polish | hyperparam
- **Iterations**: {n}
- **Kept**: {k} ({k/n}%)
- **Best metric**: {best} (iteration {i})
- **Starting metric**: {baseline}
- **Improvement**: {delta} ({percent}%)
- **Key findings**:
  - {what worked}
  - {what didn't}
- **Ideas exhaustion**: {still generating ideas | running low | stuck}
```

## When NOT to Use Auto-Loop

- **Literature search** — not iterative, one-shot extraction
- **Analysis** — deterministic computation on fixed data, no improvement axis
- **Gate checking** — pass/fail verification, nothing to optimize
- **Planning** — strategic decisions, not metric-driven

For these phases, use the dedicated skill directly.

## Safety Limits

- `max_iterations` — hard cap, never exceed
- `time_budget_minutes` — per run, kill if exceeded
- Git branch isolation — all autoloop work on a branch, easy to discard entirely
- `results.tsv` — full audit trail, never deleted
- Revert on failure — `git reset --hard` ensures clean state
- **NEVER modify evaluation code** — the metric function is read-only
  (same as Karpathy's `prepare.py` rule)

## Integration with ARS Pipeline

The `/ars-12-pipeline` skill can invoke auto-loop at appropriate phases:

```
Phase 4 (Experiments):
  → /ars-06-auto-loop mode=experiment for model/training iteration
  → /ars-06-auto-loop mode=hyperparam for hyperparameter search

Phase 7 (Review Loop):
  → /ars-06-auto-loop mode=review for draft improvement
  → /ars-06-auto-loop mode=writing_polish for prose quality
```

The pipeline respects auto-loop's stopping conditions and incorporates
the best results back into the main project state.
