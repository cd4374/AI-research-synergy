---
name: ars-04-experiment
description: Implement and run ARS experiments locally or remotely.
---

# Skill: Experiment

> **Role**: Write experiment code, generate run configs, execute runs locally
> or on remote GPU servers. Collect results. Never interpret results.

## Trigger Keywords

experiment, run experiment, code, implement, train, execute, GPU, deploy,
baseline, ablation, reproduce, 实验, 跑实验, 训练, 部署

## Inputs

- Task spec from `PROJECT_STATE.md` (which experiment to run)
- Runtime environment selection from `PROJECT_STATE.md` (`Runtime Environment` section)
- SOTA table from `results/sota_table.md` (baselines to compare against)
- (Optional) Prior `EXPERIMENT_LOG.md` entries (to avoid repeating failures)
- (Optional) GPU server config from `CLAUDE.md`

## Procedure

### Step 0 — Read Selected Environment

Before writing code, read the `Runtime Environment` section in `PROJECT_STATE.md` and obey it.
If the section is missing, still pending, or lacks a final selected mode, stop and require `/ars-02-environment` to run successfully first.

Environment rules:
- `local_gpu`: write CUDA-capable code paths and local GPU run commands
- `local_mps`: write Apple Silicon / MPS-capable code paths and avoid assuming CUDA
- `remote_gpu`: prefer rsync + ssh execution flow using the selected remote target
- `local_cpu`: avoid GPU-only assumptions and prefer lighter configs when reasonable
- Reuse the recorded conda environment by default; only propose a new one if the recorded choice is missing or invalid
- If `Selected Conda Env` is `none`, do not assume `conda run`; emit plain `python` / `pip` commands that match the detected environment and note the limitation in `README_run.md`

**GPU Subset**: read `CUDA_VISIBLE_DEVICES` from `PROJECT_STATE.md`.
- If value is `unset` or field is absent → do not set `CUDA_VISIBLE_DEVICES`; all GPUs are used
- If value is a device list (e.g. `0,2`) → prepend `CUDA_VISIBLE_DEVICES=0,2` to every training and evaluation command
- For remote execution, export the variable inside the screen/tmux command block

If the requested experiment conflicts with the selected environment, log a blocker or a degraded execution plan instead of silently assuming a different runtime.

### Step 1 — Design Experiment Spec

Before writing code, document the experiment:

```markdown
## Experiment: {EXP-ID}
- **Hypothesis**: {what we expect to observe}
- **Independent variable**: {what we change}
- **Dependent variable**: {what we measure}
- **Baselines**: {what we compare against}
- **Metrics**: {exact metric names and computation method}
- **Seeds**: [42, 123, 456] (minimum 3 for statistical validity)
- **Estimated GPU-hours**: {estimate}
- **Success criteria**: {specific threshold or comparison}
```

If `estimated GPU-hours > MAX_EXPERIMENT_HOURS` (default 8):
→ **STOP**. Log to `EXPERIMENT_LOG.md` as `skipped: exceeds compute budget`.
→ Flag for human approval unless `AUTO_PROCEED: true` AND estimate < 2x limit.

### Step 2 — Write Code

Create experiment code in `src/`:

```
src/
├── train.py          # Main training/execution script
├── evaluate.py       # Evaluation script (separate from training)
├── config.py         # All hyperparameters in one place
├── data/
│   └── loader.py     # Data loading
├── models/
│   └── {model}.py    # Model implementations
└── utils/
    └── metrics.py    # Metric computation (deterministic)
```

**Code rules:**
1. Set ALL random seeds (Python, NumPy, PyTorch, CUDA/MPS where applicable)
2. Log all hyperparameters at run start
3. Save results as JSON to `results/{exp_id}/metrics.json`
4. Save model checkpoints to `results/{exp_id}/checkpoints/`
5. Use `argparse` or config file for all parameters — no hardcoded values
6. Include a `requirements.txt` or `pyproject.toml`
7. Write a `README_run.md` explaining how to reproduce
8. Match install/run instructions to the selected conda environment and runtime target

### Step 3 — Implementation Review (Optional)

If the run is expensive, the eval path is custom, or the code changed materially,
call `/ars-05-implementation-review` before running.

Use that skill's rubric rather than duplicating review logic here.

If blocking issues are found → fix before running.
Record any execution-facing note in `EXPERIMENT_LOG.md`.

### Step 4 — Execute Run

**Local execution (with conda):**
```bash
# If CUDA_VISIBLE_DEVICES is set (GPU Subset != all/unset):
export CUDA_VISIBLE_DEVICES={gpu_subset}   # e.g. 0,2

conda run -n {selected_env} pip install -r requirements.txt
conda run -n {selected_env} python train.py --config configs/{exp_id}.yaml --seed 42
conda run -n {selected_env} python train.py --config configs/{exp_id}.yaml --seed 123
conda run -n {selected_env} python train.py --config configs/{exp_id}.yaml --seed 456
conda run -n {selected_env} python evaluate.py --results_dir results/{exp_id}/
```

**Local execution (without conda):**
```bash
# If CUDA_VISIBLE_DEVICES is set:
export CUDA_VISIBLE_DEVICES={gpu_subset}

pip install -r requirements.txt
python train.py --config configs/{exp_id}.yaml --seed 42
python train.py --config configs/{exp_id}.yaml --seed 123
python train.py --config configs/{exp_id}.yaml --seed 456
python evaluate.py --results_dir results/{exp_id}/
```

**Remote GPU execution:**
```bash
# Sync code to server
rsync -avz src/ {user}@{host}:{workdir}/src/
rsync -avz configs/ {user}@{host}:{workdir}/configs/

# Launch in screen/tmux (survives disconnects)
ssh {user}@{host} "cd {workdir} && screen -dmS {exp_id} bash -c '
  export CUDA_VISIBLE_DEVICES={gpu_subset}   # omit line if GPU Subset is all/unset
  conda run -n {selected_env} pip install -r src/requirements.txt &&
  conda run -n {selected_env} python src/train.py --config configs/{exp_id}.yaml --seed 42 &&
  conda run -n {selected_env} python src/train.py --config configs/{exp_id}.yaml --seed 123 &&
  conda run -n {selected_env} python src/train.py --config configs/{exp_id}.yaml --seed 456 &&
  conda run -n {selected_env} python src/evaluate.py --results_dir results/{exp_id}/
'"

# Check status
ssh {user}@{host} "screen -ls"
```

### Step 5 — Monitor and Collect

Check experiment progress:
- `ssh {host} "tail -20 {workdir}/results/{exp_id}/train.log"`
- `ssh {host} "cat {workdir}/results/{exp_id}/metrics.json"`

When complete:
```bash
# Sync results back
rsync -avz {user}@{host}:{workdir}/results/{exp_id}/ results/{exp_id}/
```

### Step 6 — Log Results

Append to `EXPERIMENT_LOG.md`:

```markdown
## {EXP-ID}: {title}
- **Date**: {date}
- **Status**: completed | failed | timeout
- **Config**: configs/{exp_id}.yaml
- **Seeds**: [42, 123, 456]
- **Duration**: {wall clock time}
- **Results**:
  - Metric1: {mean ± std across seeds}
  - Metric2: {mean ± std across seeds}
- **Artifacts**: results/{exp_id}/
- **Failure class**: {if failed: infra_error | dependency_error | code_error | method_failure}
- **Notes**: {any observations, but NO interpretation}
```

### Step 7 — Register Evidence

For successful runs, add raw results to `EVIDENCE.md`:

```markdown
- [EVD-{seq}] Experiment {EXP-ID}: {metric} = {mean} ± {std} (n={seeds})
  - source: experiment
  - artifact: results/{exp_id}/metrics.json
  - task: {task_id}
  - strength: direct
  - validation: pending_review
```

**Do NOT interpret results.** Just record the numbers. Interpretation is `/ars-07-analysis`'s job.

### Step 8 — Handle Failures

On failure, classify and act:
- `infra_error`: Retry up to 3x with exponential backoff
- `dependency_error`: Fix requirements, retry
- `code_error`: Read traceback, fix bug, retry
- `method_failure`: Log as-is, do not retry — route to `/ars-10-reviewer`
- `timeout`: Log, flag for human if estimate was wrong

## Output

- Experiment code in `src/`
- Run configs in `configs/`
- Raw results in `results/{exp_id}/`
- Entries in `EXPERIMENT_LOG.md`
- Evidence entries in `EVIDENCE.md`
- `README_run.md` for reproducibility

## Boundaries

- **DO**: Write code, run experiments, collect raw results, classify failures
- **DO NOT**: Interpret results, make claims, write paper, choose what experiments to run next
- If unsure what to run → check `PROJECT_STATE.md` for the next task
