# ARS — AI Research Synergy

> Governed autonomous research pipeline. Claude Code executes, external LLM reviews.
> Zero dependencies — everything is Markdown.

```
┌──────────────────────────────────────────────────┐
│ /ars-01-coordinator → /ars-03-literature → /ars-04-experiment          │
│      → /ars-07-analysis → /ars-09-writer → /ars-10-reviewer            │
│   optional: /ars-05-implementation-review before major runs       │
│              ↕ /ars-11-gate-check (7 gates)                       │
└──────────────────────────────────────────────────┘
```

## What Makes ARS Different

**Claim-evidence chain**: Every paper claim traces to evidence, which traces to
an artifact, which traces to an experiment. `/ars-11-gate-check` enforces 100% coverage
before the paper is considered ready.

**Cross-model adversarial review**: Claude Code does the work, an external LLM
(via Codex MCP) reviews it. No model grades its own homework.

**Selective collaboration**: External review is concentrated at high-risk reasoning
boundaries instead of every procedural step. Routine execution stays single-model;
critical gates and triggered escalations get cross-model scrutiny.

**Quality gates**: 7 mandatory checkpoints (A0→G1) that block progression until
conditions are met. No shortcuts.

**Pure Markdown**: No database, no Docker, no framework. State lives in `.md` files.
Fork it, modify it, use it with any LLM agent.

## Quick Start

### 1. Install Skills

```bash
git clone https://github.com/youruser/ai-research-synergy.git
cd ai-research-synergy
cp -r skills/* ~/.claude/skills/
```

### 2. Set Up Codex MCP (for cross-model review)

```bash
npm install -g @openai/codex
claude mcp add codex -s user -- codex mcp-server
```

### 3. Run

```bash
# Full pipeline (autonomous overnight)
claude> /ars-12-pipeline "your research topic"

# Or step by step
claude> /ars-01-coordinator "research brief"
claude> /ars-02-environment target=auto
claude> /ars-02-environment target=local_gpu gpu_ids=0      # single GPU
claude> /ars-02-environment target=local_gpu gpu_ids=0,2    # specific GPUs
claude> /ars-03-literature "topic keywords"
claude> /ars-04-experiment
claude> /ars-07-analysis
claude> /ars-09-writer
claude> /ars-10-reviewer
claude> /ars-11-gate-check G1

# Optional safeguard before major runs or custom eval changes
claude> /ars-05-implementation-review
# e.g. provide spec + core code paths + config + focus area
```

#### Example: Step-by-Step Run

```
/ars-01-coordinator "Few-shot data augmentation for time-series classification"
  → Creates PROJECT_STATE.md, defines 3 research questions, identifies baseline

/ars-02-environment target=auto
  → Selects local_gpu/local_mps/remote_gpu/local_cpu, reuses or creates conda env, updates PROJECT_STATE.md
  → Add gpu_ids=0,2 to restrict to specific GPUs (local_gpu only); CUDA_VISIBLE_DEVICES propagated automatically

/ars-03-literature "few-shot time-series augmentation"
  → Finds 12 papers, builds SOTA table, adds 8 evidence items to EVIDENCE.md

/ars-04-experiment
  → Writes src/train.py, runs 5 configs on 4 GPUs, logs to EXPERIMENT_LOG.md

/ars-05-implementation-review   (optional before major runs or custom eval changes)
  → Audits whether the code faithfully implements the idea and eval protocol

/ars-07-analysis
  → Computes stats, generates figures/, registers evidence in EVIDENCE.md

/ars-09-writer
  → Drafts paper/ with claim-evidence binding, no unsupported claims

/ars-10-reviewer
  → Codex MCP returns score 6.2/10, lists 5 weaknesses

/ars-11-gate-check G1
  → Final verification: either passes the full package or returns the remaining blockers
```

## Review Policy

ARS uses **selective cross-model collaboration**:
- **Default**: Claude handles routine execution alone
- **Mandatory external review**: Gates `C1`, `F1`, `G1`
- **Triggered external review**: novelty uncertainty, high-cost experiments, high/critical claims, fairness/evaluator risk, stalled review loops, or anomalous results

This keeps external review focused on high-risk reasoning boundaries instead of every procedural step.

### 4. Fully Automatic Mode

For unattended runs, `/ars-12-pipeline` should complete `/ars-02-environment` before Gate A0.

### 5. Overnight Mode

Add to `.claude/settings.local.json`:
```bash
mkdir .claude
nano .claude/settings.local.json
```

```json
{
  "permissions": {
    "allow": [
      "mcp__codex__codex",
      "Write",
      "Edit",
      "WebSearch",
      "Skill(arxiv)",
      "Bash(python:*)",
      "Bash(python3:*)",
      "Bash(pip:*)",
      "Bash(conda:*)",
      "Bash(ssh:*)",
      "Bash(rsync:*)",
      "Bash(screen:*)",
      "Bash(tmux:*)",
      "Bash(nvidia-smi:*)"
    ]
  }
}
```

Then: `claude --dangerously-skip-permissions`

### Mode-specific permissions

- `local_gpu`: needs `Bash(conda:*)`, `Bash(python:*)`, `Bash(python3:*)`, `Bash(pip:*)`, `Bash(nvidia-smi:*)`
- `local_mps`: needs `Bash(conda:*)`, `Bash(python:*)`, `Bash(python3:*)`, `Bash(pip:*)`
- `remote_gpu`: needs `Bash(conda:*)`, `Bash(ssh:*)`, `Bash(rsync:*)`, `Bash(screen:*)` or `Bash(tmux:*)`
- `local_cpu`: needs `Bash(conda:*)`, `Bash(python:*)`, `Bash(python3:*)`, `Bash(pip:*)`

## Skills

| # | Skill | Role | Needs Codex MCP? |
|---|-------|------|:---:|
| 01 | `/ars-01-coordinator` | Plan, decompose, prioritize, pivot | No |
| 02 | `/ars-02-environment` | Detect/select runtime target and conda env | No |
| 03 | `/ars-03-literature` | Search papers, SOTA table, evidence extraction | No |
| 04 | `/ars-04-experiment` | Write code, run experiments, collect results | No |
| 05 | `/ars-05-implementation-review` | Audit spec-to-code correctness and eval integrity | Recommended |
| 06 | `/ars-06-auto-loop` | Karpathy-style iterate-measure-keep/discard loop | Depends on mode |
| 07 | `/ars-07-analysis` | Statistics, figures, evidence registration | No |
| 08 | `/ars-08-figure-polish` | VLM-driven iterative figure refinement | Recommended |
| 09 | `/ars-09-writer` | Draft paper with claim-evidence binding | No |
| 10 | `/ars-10-reviewer` | Cross-model adversarial project/paper review | **Yes** |
| 11 | `/ars-11-gate-check` | Verify quality gate conditions | No |
| 12 | `/ars-12-pipeline` | Orchestrate full workflow | Yes (for review) |

### Auto-Loop Modes

Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch).
The core pattern: modify → run (fixed budget) → measure → keep/discard → repeat.

| Mode | What It Iterates | Metric | Needs Codex? |
|------|-----------------|--------|:---:|
| `experiment` | Model/training code | val_loss, accuracy | No |
| `hyperparam` | Config values | val_loss, accuracy | No |
| `review` | Paper draft (via reviewer) | Review score (1-10) | **Yes** |
| `writing_polish` | Prose quality | Composite quality score | No |

## State Files

All state lives in Markdown at the project root:

| File | What's In It |
|------|-------------|
| `PROJECT_STATE.md` | Phase, milestones, task backlog, config |
| `CLAIMS.md` | All claims with evidence links and status |
| `EVIDENCE.md` | Evidence registry (lit, experiments, stats) |
| `REVIEW_LOG.md` | Review rounds, scores, action items |
| `DECISIONS.md` | Pivot and scope change records |
| `EXPERIMENT_LOG.md` | All experiment runs and results |
| `NARRATIVE.md` | Running lab notebook |

Templates are in `templates/`. `/ars-01-coordinator` initializes them automatically.

## Quality Gates

| Gate | Name | Key Checks |
|------|------|-----------|
| A0 | Problem Clarity | Questions verifiable, scope bounded, baseline identified, environment selected |
| B1 | Literature Complete | SOTA table, ≥5 evidence items, gap analysis, novelty check |
| C1 | Experiment Plan | Hypotheses, metrics, baselines, statistical plan, reviewer ≥6 |
| D1 | Experiments Complete | All runs done, ≥3 seeds, failures classified |
| E1 | Analysis Complete | Stats computed, figures exist, negatives reported |
| F1 | Draft Complete | 100% claim coverage (results+conclusions), no fake citations |
| G1 | Final Review | All gates passed, reviewer ≥7, full evidence chain intact |

## Configuration

In `PROJECT_STATE.md` Config section:

```
AUTO_PROCEED: true          # Full autopilot vs pause at gates
MAX_REVIEW_ROUNDS: 6        # Max review-fix cycles
MAX_EXPERIMENT_HOURS: 8     # Skip long experiments
POSITIVE_THRESHOLD: 7       # Score to stop review loop
```

GPU servers in `PROJECT_STATE.md` Config section (filled in by `/ars-02-environment`).

## License

MIT
