---
name: run-experiment
description: Deploy and run ML experiments on local or remote GPU servers via SSH. Use when user says "run experiment", "deploy to server", "跑实验", or needs to launch training jobs.
argument-hint: [experiment-description or script path]
allowed-tools: Bash(*), Read, Grep, Glob, Edit, Write
---

# Run Experiment

Deploy and run ML experiment: $ARGUMENTS

## Configuration (PROJECT_STATE.md)

Read the project's `PROJECT_STATE.md` to determine the GPU target. The `## Remote Server`
section contains all connection info. The `conda_env` is **never hardcoded** — it is
resolved dynamically at runtime using the same policy as `ars-02-environment`.

```markdown
## Remote Server
- gpu: remote
- ssh: ars-gpu                      # SSH alias in ~/.ssh/config
- conda: /home/sh1/anaconda3        # conda installation path on server
- workdir: /home/sh1/{project}/     # remote working directory
- code_sync: rsync                  # rsync (default) or git
- wandb: false
```

For local GPU, omit the section or set `gpu: local` — the local environment from
`## Runtime Environment` in `PROJECT_STATE.md` is used directly.

If the `## Remote Server` section is missing and the user wants remote execution,
stop and tell the user to add it to `PROJECT_STATE.md`.

## Workflow

### Step 1: Read Config

Read `PROJECT_STATE.md`. Extract from `## Remote Server`:
- `gpu`: `local` | `remote` (default `local` if absent)
- `ssh`: SSH alias (remote only)
- `conda`: conda installation path on server (remote only)
- `workdir`: remote working directory (remote only)
- `code_sync`: `rsync` (default) | `git`
- `wandb`: `true` | `false` (default `false`)
- `wandb_project`: W&B project name (only if `wandb: true`)

Also read `## Runtime Environment` for:
- `Selected Conda Env` — used for local execution
- `CUDA_VISIBLE_DEVICES` — GPU subset constraint

### Step 2: Resolve Conda Environment (Remote)

**Never use a hardcoded conda env name.** Resolve dynamically on the server:

```bash
# List all non-base envs on the server
ssh <ssh> "source <conda>/etc/profile.d/conda.sh && conda env list"
```

Apply this selection policy (same as ars-02-environment):
1. Skip `base`
2. Prefer env whose name contains a task hint from `PROJECT_STATE.md` (e.g. project name,
   framework name like `torch`, `tf`, `jax`)
3. Among matches, prefer the one where `python --version` succeeds and Python ≥ 3.9
4. If no env qualifies, create one:
   ```bash
   ssh <ssh> "source <conda>/etc/profile.d/conda.sh && conda create -y -n ars-remote python=3.11"
   ```
5. Record the selected env name for use in Steps 3–4

### Step 3: Pre-flight — Check GPU Availability

**Remote:**
```bash
ssh <ssh> "nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader"
```

**Local (CUDA):**
```bash
nvidia-smi --query-gpu=index,name,memory.used,memory.total --format=csv,noheader
```

**Local (MPS):**
```bash
python -c "import torch; print('MPS:', torch.backends.mps.is_available())"
```

A GPU is free when `memory.used < 500 MiB`. Pick the GPU with the most free memory.
If `CUDA_VISIBLE_DEVICES` is set in `PROJECT_STATE.md`, restrict selection to those indices.
If no GPU is free, report clearly and stop — never launch on a busy GPU.

### Step 4: Sync Code (Remote Only)

Only sync source files; never sync data, checkpoints, or large binaries.

#### Option A: rsync (default)

```bash
rsync -avz \
  --include='*.py' --include='*.yaml' --include='*.yml' \
  --include='*.json' --include='*.txt' --include='*.sh' --include='*/' \
  --exclude='*.pt' --exclude='*.pth' --exclude='*.ckpt' \
  --exclude='__pycache__/' --exclude='.git/' \
  --exclude='data/' --exclude='results/' --exclude='wandb/' \
  ./ <ssh>:<workdir>/
```

If `requirements.txt` exists, install after sync:
```bash
ssh <ssh> "source <conda>/etc/profile.d/conda.sh && conda activate <resolved_env> && cd <workdir> && pip install -q -r requirements.txt"
```

#### Option B: git (`code_sync: git`)

```bash
git add -A && git commit -m "sync: experiment deployment" && git push
ssh <ssh> "cd <workdir> && git pull"
```

### Step 5: Deploy

Each experiment gets its own `screen` session so it survives disconnects.

#### Remote (SSH + screen + conda)

```bash
ssh <ssh> "screen -dmS <exp_name> bash -c '
  source <conda>/etc/profile.d/conda.sh &&
  conda activate <resolved_env> &&
  cd <workdir> &&
  export CUDA_VISIBLE_DEVICES=<gpu_id> &&
  python <script> <args> 2>&1 | tee results/<exp_name>/train.log
'"
```

- `<resolved_env>`: env name resolved in Step 2
- `<exp_name>`: short identifier, e.g. `exp-baseline-42`
- `<gpu_id>`: GPU index with most free memory from Step 3
- Omit `CUDA_VISIBLE_DEVICES` line if running on all GPUs

Multiple seeds — parallel if enough GPUs, sequential otherwise:
```bash
# Parallel (one GPU per seed)
ssh <ssh> "screen -dmS <exp>-s42  bash -c 'source <conda>/etc/profile.d/conda.sh && conda activate <env> && cd <workdir> && CUDA_VISIBLE_DEVICES=0 python train.py --seed 42  2>&1 | tee results/<exp>/seed42.log'"
ssh <ssh> "screen -dmS <exp>-s123 bash -c 'source <conda>/etc/profile.d/conda.sh && conda activate <env> && cd <workdir> && CUDA_VISIBLE_DEVICES=1 python train.py --seed 123 2>&1 | tee results/<exp>/seed123.log'"
```

#### Local (CUDA)

```bash
CUDA_VISIBLE_DEVICES=<gpu_id> python <script> <args> 2>&1 | tee results/<exp_name>/train.log
```

Use `run_in_background: true` for long-running local jobs.

#### Local (MPS)

```bash
python <script> <args> 2>&1 | tee results/<exp_name>/train.log
```

### Step 6: Verify Launch

**Remote:**
```bash
ssh <ssh> "screen -ls"
```
Confirm the session appears as `(Detached)`.

**Local:**
```bash
nvidia-smi
```

### Step 7: Monitor and Collect Results

```bash
# Tail log
ssh <ssh> "tail -30 <workdir>/results/<exp_name>/train.log"

# Check if still running
ssh <ssh> "screen -ls | grep <exp_name>"

# Check JSON results
ssh <ssh> "cat <workdir>/results/<exp_name>/metrics.json 2>/dev/null"
```

When complete (screen session gone), sync results back:
```bash
rsync -avz <ssh>:<workdir>/results/<exp_name>/ results/<exp_name>/
```

### Step 8: W&B Integration (when `wandb: true`)

**Skip entirely if `wandb: false` or not set.**

Check if training script already has `import wandb`. If not, add:
```python
import wandb
wandb.init(project="<wandb_project>", name="<exp_name>", config={...hyperparams...})
wandb.log({"train/loss": loss, "train/lr": lr, "step": step})
wandb.log({"eval/loss": eval_loss, "eval/accuracy": acc})
wandb.finish()
```

Verify W&B login:
```bash
ssh <ssh> "wandb status"
# If not: ssh <ssh> "wandb login <WANDB_API_KEY>"
```

## Key Rules

- ALWAYS check GPU availability before assigning — never blindly use GPU 0
- NEVER hardcode a conda env name — always resolve from server's env list
- One screen session per experiment seed
- Always use `tee` so logs survive session death
- Report: resolved env, GPU assigned, screen session name, workdir

## PROJECT_STATE.md Template

```markdown
## Remote Server
- gpu: remote
- ssh: ars-gpu
- conda: /home/sh1/anaconda3
- workdir: /home/sh1/{project_name}/
- code_sync: rsync
- wandb: false
```
