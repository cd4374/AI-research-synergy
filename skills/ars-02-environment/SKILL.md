---
name: ars-02-environment
description: Detect and select runtime environment and conda env for ARS experiments.
---

# Skill: Environment

> **Role**: Detect runtime capabilities, choose an execution target, decide whether to reuse or create a conda environment, and write the result into `PROJECT_STATE.md`.
> Never write experiment code. Never interpret results.

## Trigger Keywords

environment, runtime, detect environment, conda, gpu, mps, cpu, remote gpu,
setup environment, 环境检测, 运行环境, conda环境, MPS

## Usage

```bash
/ars-02-environment
/ars-02-environment target=auto
/ars-02-environment target=local_gpu
/ars-02-environment target=local_gpu gpu_ids=0
/ars-02-environment target=local_gpu gpu_ids=0,2
/ars-02-environment target=local_mps
/ars-02-environment target=remote_gpu
/ars-02-environment target=local_cpu
```

`gpu_ids` is optional and only applies to `local_gpu` targets. Accepts a
comma-separated list of NVIDIA device indices (e.g. `0`, `0,1`, `2,3`).
Omit to use all visible GPUs.

## Supported Targets

- `auto` → choose by priority: `local_gpu` > `local_mps` > `remote_gpu` > `local_cpu`
- `local_gpu`
- `local_mps`
- `remote_gpu`
- `local_cpu`

## Responsibilities

1. Detect available hardware/software/runtime capabilities by actually running checks
2. Resolve the target environment from user input or `auto`
3. Check whether conda is available
4. Reuse an existing conda env when possible
5. Create a new conda env only when no suitable env exists
6. Write a structured `Runtime Environment` section into `PROJECT_STATE.md`
7. Stop with a blocker if detection or writeback cannot be completed

## Detection Scope

### Hardware
- NVIDIA GPU visibility
- Apple MPS availability
- CPU-only fallback availability
- Basic memory visibility if easy to obtain

### Software
- `python`
- `conda`
- `pip`
- `uv` (optional)
- `ssh`
- `rsync`
- `screen` or `tmux`

### Remote
- Whether `GPU_SERVERS` is configured in `PROJECT_STATE.md` (Config section)
- Whether local machine has tools needed to launch remote jobs

## Conda Policy

Default policy: reuse existing env first.

A conda env is considered **usable** if:
- Python runs successfully
- Python version meets the minimum required version
- It is compatible with the selected target runtime
- It does **not** need to already contain all task-specific packages
- It is **not** the `base` environment

**Never select `base`**: `base` is reserved for conda's own tooling. Always prefer a named env; if none qualifies, create a new one.

If multiple envs qualify:
- choose the one that best matches the selected target

If none qualify:
- create a new conda env with an appropriate Python version
- record that the env was newly created

## Procedure

### Step 1 — Read State and User Target
- Read `PROJECT_STATE.md` if it exists; if not, stop and tell the user to run `/ars-01-coordinator` first
- Parse `target=` if provided; if none, use `auto`
- Read `~/.ars/config.yml` for `gpu_servers`; if file is missing, treat `gpu_servers` as empty and continue
- If `target=remote_gpu` but `gpu_servers` is empty, stop and tell the user to add a server to `~/.ars/config.yml`

### Step 2 — Detect Capabilities
Run non-destructive checks such as:
- `conda --version`
- `conda env list`
- `python --version`
- `pip --version`
- `ssh -V`
- `rsync --version`
- `screen --version` or `tmux -V`
- `nvidia-smi` for NVIDIA GPU presence
- `nvidia-smi --query-gpu=index,name,memory.total,memory.free --format=csv,noheader` for per-GPU details (only when `nvidia-smi` is present)
- Python checks for MPS capability when relevant

These checks are mandatory. Do not skip them and do not infer capabilities from documentation alone.

Recommended command patterns:

```bash
conda --version
conda env list
python --version
pip --version
ssh -V
rsync --version
screen --version || tmux -V
nvidia-smi
nvidia-smi --query-gpu=index,name,memory.total,memory.free --format=csv,noheader 2>/dev/null
python - <<'PY'
import platform
print(platform.platform())
try:
    import torch
    print('torch', torch.__version__)
    print('cuda', torch.cuda.is_available())
    print('mps', hasattr(torch.backends, 'mps') and torch.backends.mps.is_available())
except Exception as e:
    print('torch_check_error', e)
PY
```

### Step 2.5 — ARS Toolchain Check

Check all external tools and services that ARS skills depend on. This is mandatory
and must complete before environment selection. Failures are classified as either
**blocking** (pipeline cannot run at all) or **degraded** (specific skills will fall
back to reduced functionality).

#### 2.5.1 — Codex MCP Server

```bash
# Check if codex CLI is installed
codex --version 2>/dev/null || npx @openai/codex --version 2>/dev/null

# Check if codex mcp-server is registered in Claude MCP config
claude mcp list 2>/dev/null | grep -i codex
```

Interpret results:

| State | Classification | Effect |
|-------|---------------|--------|
| `codex` binary found AND listed in `claude mcp list` | **ready** | Full cross-model review available |
| `codex` binary found but NOT in `claude mcp list` | **misconfigured** | Run `claude mcp add codex -s user -- codex mcp-server` to fix |
| `codex` NOT found, `npm`/`npx` available | **installable** | Can fix with `npm install -g @openai/codex` |
| neither `codex` nor `npm`/`npx` found | **unavailable** | Degraded: mandatory gates C1/F1/G1 will use self-review with disclaimer |

Record the status as one of: `ready | misconfigured | installable | unavailable`

#### 2.5.2 — npm / npx (required to install Codex)

```bash
npm --version 2>/dev/null
npx --version 2>/dev/null
node --version 2>/dev/null
```

#### 2.5.3 — Web Search capability

```bash
# Claude Code's WebSearch tool is built-in; check if any proxy/firewall blocks it
# by attempting a minimal fetch (do NOT call WebSearch here — just note it as assumed available)
```

Record: `assumed_available` (WebSearch is a built-in Claude Code tool, cannot be
independently probed from shell; mark unavailable only if the user has reported
it failing in this session).

#### 2.5.4 — arXiv / Semantic Scholar reachability (optional)

```bash
curl -s --max-time 5 -o /dev/null -w "%{http_code}" https://arxiv.org/ 2>/dev/null
curl -s --max-time 5 -o /dev/null -w "%{http_code}" https://api.semanticscholar.org/ 2>/dev/null
```

HTTP 200 or 301/302 → reachable. Timeout / connection refused → blocked (lit review
will be limited to cached/offline sources).

#### 2.5.5 — Git (required for auto-loop branch isolation)

```bash
git --version
git -C . rev-parse --is-inside-work-tree 2>/dev/null
```

Git missing or not in a git repo → auto-loop branch isolation will be **skipped**
(loop still runs, but without `git reset` safety net; log this as a risk).

#### 2.5.6 — Produce Toolchain Summary

After all checks, produce a structured summary table:

```
ARS Toolchain Status:
  codex_mcp:      ready | misconfigured | installable | unavailable
  npm_npx:        yes | no
  web_search:     assumed_available | reported_unavailable
  arxiv:          reachable | unreachable
  semantic_scholar: reachable | unreachable
  git:            yes (in repo) | yes (no repo) | no
```

**Blocking conditions** (stop pipeline, do not proceed to Gate A0):
- None by default. Codex MCP unavailability is degraded, not blocking.

**Degraded conditions** (proceed but record limitations):
- `codex_mcp: misconfigured` → print fix instructions and ask user to resolve before running Gate C1/F1/G1
- `codex_mcp: installable` → print install command, offer to run it, continue if user declines
- `codex_mcp: unavailable` → mandatory review gates will use self-review fallback; log in `PROJECT_STATE.md`
- `arxiv: unreachable` → literature search limited to Semantic Scholar and WebSearch
- `git: yes (no repo)` → auto-loop will run without branch isolation; flag as risk

### Step 3 — Select Target
Apply this order unless user explicitly overrides:
1. `local_gpu`
2. `local_mps`
3. `remote_gpu`
4. `local_cpu`

Rules:
- `local_gpu` requires visible local NVIDIA GPU support
- `local_mps` requires Apple Silicon / MPS support
- `remote_gpu` requires configured `GPU_SERVERS` plus `ssh` and `rsync`
- `local_cpu` is always the final fallback

Selection algorithm:
- If user explicitly requests a target and it is available, select it
- If user explicitly requests a target and it is unavailable, record the failure and choose `local_cpu` only if the user requested fallback semantics; otherwise stop and report the blocker
- If target is `auto`, choose the first available target in the default priority order
- Always record both the requested target and the final selected mode

#### GPU Subset Selection (`gpu_ids`)

When target is `local_gpu`, also resolve the GPU subset:

1. Parse `gpu_ids` from user input. Accepted formats: `0`, `0,1`, `0,2,3`.
2. If `gpu_ids` is provided:
   - Validate each ID exists in `nvidia-smi` output. If any ID is invalid, stop and report the bad IDs plus the list of valid ones.
   - Set `GPU Subset` to the provided value (e.g. `0,2`).
   - Set `CUDA_VISIBLE_DEVICES` accordingly.
3. If `gpu_ids` is **not** provided:
   - Set `GPU Subset` to `all` (use all visible GPUs, no `CUDA_VISIBLE_DEVICES` override).
4. Record the available GPU list in `Hardware Summary` (index, name, VRAM) so future skills can reference it.

### Step 4 — Resolve Conda Environment
- If conda is unavailable: record that clearly, set `Selected Conda Env: none`, and continue only with documentation-level guidance
- If conda is available: inspect existing envs
- Reuse the first suitable env for the selected target
- Otherwise create a new env and record its name, Python version, and target

Conda resolution is mandatory for automatic execution. If conda exists but env inspection, reuse, or creation fails, stop and report the blocker instead of silently proceeding.

Reuse algorithm:
- Enumerate all conda envs, **skip `base`**
- For each env, test `conda run -n {env} python --version`
- Prefer env names containing the target hint (`gpu`, `mps`, `cpu`, `remote`)
- If no hinted env qualifies, reuse the first env that satisfies the usability rule

Creation algorithm:
- Choose Python 3.11 by default unless `PROJECT_STATE.md` already requires a different minimum
- Use target-specific names like `ars-local-gpu`
- After creation, verify with `conda run -n {env} python --version`

Recommended creation patterns:

```bash
conda create -y -n ars-local-gpu python=3.11
conda create -y -n ars-local-mps python=3.11
conda create -y -n ars-remote-gpu python=3.11
conda create -y -n ars-local-cpu python=3.11
```

Name suggestion rule:
- prefer `ars-{target}` style names
- if the name already exists, append a short suffix

### Step 5 — Write Runtime Environment
Replace the `Runtime Environment` section in `PROJECT_STATE.md` with detected facts. Do not append a second environment section.

Writeback is mandatory. If detection succeeded but `PROJECT_STATE.md` cannot be updated, treat the skill run as failed.

Update `PROJECT_STATE.md` with:

```markdown
## Runtime Environment
- **Target Preference**: {requested_target or auto}
- **Selected Mode**: {local_gpu | local_mps | remote_gpu | local_cpu}
- **GPU Subset**: {all | comma-separated indices e.g. 0,2 | N/A}
- **CUDA_VISIBLE_DEVICES**: {value to export, or "unset" if all GPUs are used}
- **Conda Strategy**: reuse_existing_first
- **Selected Conda Env**: {env_name | none}
- **Selected Python**: {python_version | unknown}
- **Environment Created**: {yes | no}
- **Hardware Summary**: {e.g. GPU 0: A100 80GB, GPU 1: A100 80GB | MPS available | CPU only}
- **Software Summary**: {e.g. conda yes, pip yes, ssh yes, rsync no}
- **Remote Summary**: {e.g. GPU_SERVERS configured | not configured}
- **Constraints**: {comma-separated blockers or limitations}
- **Implementation Guidance**: {short instruction for `/ars-04-experiment`}

## ARS Toolchain
- **Codex MCP**: {ready | misconfigured | installable | unavailable}
- **Codex MCP Note**: {fix instructions if not ready, else "none"}
- **npm/npx**: {yes | no}
- **WebSearch**: {assumed_available | reported_unavailable}
- **arXiv**: {reachable | unreachable}
- **Semantic Scholar**: {reachable | unreachable}
- **Git**: {yes_in_repo | yes_no_repo | no}
- **Cross-Model Review**: {full | degraded_self_review}
- **Toolchain Checked**: {date}
```

## Output

- Updated `PROJECT_STATE.md`
- Short summary of selected target and conda decision
- Clear blocker message if requested target is unavailable and fallback is not allowed

## Notes for implementation

When executing this skill, use this sequence:
1. Read `PROJECT_STATE.md`
2. Read `CLAUDE.md` to check `GPU_SERVERS`
3. Run capability checks (Step 2)
4. Run ARS toolchain checks (Step 2.5) — **must complete before Step 4**
5. Resolve selected target
6. Resolve/reuse/create conda env if possible
7. Update `PROJECT_STATE.md` (Runtime Environment + ARS Toolchain sections)
8. Print a concise summary like:

Automatic execution contract:
- A successful run must produce a populated `Runtime Environment` section in `PROJECT_STATE.md`
- A partial run that only prints observations is a failure
- If a requested target is unavailable, either fail explicitly or fall back only when the invocation asked for fallback/auto behavior

```markdown
Selected mode: local_mps
Conda env: ars-local-mps (reused)
Python: 3.11
GPU Subset: N/A (MPS target)
Constraints: no local NVIDIA GPU, remote GPU not configured

ARS Toolchain:
  codex_mcp:        ready
  npm/npx:          yes
  web_search:       assumed_available
  arxiv:            reachable
  semantic_scholar: reachable
  git:              yes (in repo)
  cross_model_review: full
```

## Boundaries

- **DO**: detect capabilities, choose target, reuse/create conda env, record facts
- **DO NOT**: write experiment code, install task-specific dependencies, interpret results
- If required tools are missing, record the limitation explicitly instead of guessing
