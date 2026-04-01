---
name: ars-08-figure-polish
description: Iteratively refine publication figures with critique loops.
---

# Skill: Figure Polish

> **Role**: Iteratively refine publication figures using VLM critique.
> Generate → render → VLM review → fix → re-render → repeat.
>
> Inspired by [SakanaAI/AI-Scientist-v2](https://github.com/SakanaAI/AI-Scientist-v2):
> figures are passed to a Vision-Language Model for critique on clarity,
> accuracy, and aesthetics, then iteratively refined until publication-ready.

## Trigger Keywords

figure polish, improve figures, fix plots, figure quality, publication figures,
subplot, layout, colorblind, legend, 优化图表, 图表美化, 论文配图

## Why This Matters

AI Scientist v1 had no vision — it couldn't see its own figures, so they were
often unreadable (overlapping labels, missing legends, bad colors). v2 added a
VLM feedback loop and the figure quality jumped dramatically. In peer review,
bad figures are a silent killer — reviewers form impressions before reading text.

## Inputs

- `figures/` directory with existing plots (PNG/PDF from `/ars-07-analysis`)
- `src/analysis/plot_*.py` — the code that generated each figure
- `results/` — raw data behind the figures
- (Optional) Venue style guide (NeurIPS, ICML, ICLR, etc.)

## Procedure

### Step 1 — Inventory Figures

List all figures in `figures/`. For each one, record:

```markdown
## Figure Inventory
| File | Type | Data Source | Referenced In |
|------|------|-------------|---------------|
| main_comparison.pdf | bar chart | results/exp1/metrics.json | paper/results.md |
| training_curves.pdf | line plot | results/exp1/train.log | paper/results.md |
| ablation.pdf | grouped bar | results/ablation/metrics.json | paper/results.md |
| architecture.pdf | diagram | manual | paper/method.md |
```

### Step 2 — VLM Critique (Per Figure)

For each figure, use vision capabilities to critique it.

If **Codex MCP is available** (preferred — cross-model critique):
```
Send figure image to Codex MCP with prompt:

"You are a reviewer at {venue}. Evaluate this figure for publication readiness.

Score each dimension 1-10:
1. READABILITY: Can you read all labels, ticks, legends at print size?
2. CLARITY: Is the message of the figure immediately obvious?
3. ACCURACY: Do the visual representations match the data correctly?
4. AESTHETICS: Professional, clean, no chartjunk, good whitespace?
5. ACCESSIBILITY: Colorblind-friendly? Works in grayscale?
6. LAYOUT: Good use of space? Subplots well-arranged?
7. LABELING: Axes labeled with units? Title informative? Legend clear?
8. CONSISTENCY: Same style/colors across all figures in the paper?

For each dimension scored < 7, provide a SPECIFIC fix.
List fixes in priority order (most impactful first)."
```

If **no VLM available** (fallback — code-level static checks):
```python
# Automated checks on plot_*.py code:
checks = {
    "font_size": "all text >= 10pt",
    "has_labels": "all axes have labels",
    "has_legend": "legend present if multiple series",
    "colorblind": "uses colorblind-friendly palette",
    "error_bars": "error bars present for comparative data",
    "dpi": "saved at >= 300 DPI",
    "format": "saved as PDF (vector) for paper",
    "tight_layout": "plt.tight_layout() or constrained_layout=True",
    "no_title": "no plt.title() (caption goes in LaTeX, not figure)",
}
```

### Step 3 — Fix Priority Queue

Collect all issues across all figures. Prioritize:

```
Priority 1 (BLOCKING — must fix):
  - Unreadable text (too small, overlapping)
  - Missing axis labels or units
  - Misleading visualization (wrong chart type, truncated axis)
  - Data mismatch (figure doesn't match reported numbers)

Priority 2 (MAJOR — should fix):
  - Not colorblind-friendly
  - Missing error bars on comparative data
  - Poor subplot arrangement (wasted space, unaligned)
  - Inconsistent style across figures
  - Low resolution (< 300 DPI)

Priority 3 (POLISH — nice to fix):
  - Aesthetic improvements (whitespace, grid style)
  - Consolidating related subplots into composite figure
  - Font consistency with paper body text
  - Color palette refinement
```

### Step 4 — Apply Fixes (Auto-Loop)

For each figure, starting with highest priority issues:

```
REPEAT (max 5 rounds per figure):
  a. Edit src/analysis/plot_{name}.py to fix the top issue
  b. Re-run: python src/analysis/plot_{name}.py
  c. VLM re-critique the regenerated figure
  d. If the specific issue is fixed → advance to next issue
  e. If fix introduced new problems → revert, try different approach
  f. Stop when all dimensions score >= 7, or max rounds reached
```

### Step 5 — Composite Figure Generation

After individual figures are polished, consider composition:

**Should subplots be merged?**
- Related comparisons (e.g., accuracy + loss) → side-by-side subplots
- Ablation variants → grouped in one figure with shared legend
- Training curves for different methods → overlaid on same axes

**Composition rules:**
```python
# Good composite: shared x-axis, related content
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
# ax1: accuracy comparison
# ax2: loss comparison
# shared legend at bottom

# Bad composite: unrelated content forced together
# Don't merge architecture diagram with bar chart
```

**Layout heuristics by venue:**
- 2-column papers (NeurIPS, ICML): figures should be column-width (3.25") or full-width (6.75")
- Single-column: figures up to 6" wide
- Avoid figures that are too tall — horizontal layouts preferred
- Maximum 6-8 figures per paper; consolidate if more

### Step 6 — Style Consistency Pass

After all individual fixes, do a cross-figure consistency check:

```python
STYLE_GUIDE = {
    "font_family": "serif",        # Match LaTeX body
    "font_size": 10,               # Minimum for readability
    "tick_size": 8,
    "legend_size": 9,
    "color_palette": "tab10",       # Or venue-specific
    "line_width": 1.5,
    "marker_size": 6,
    "grid": True,                   # Light grid for data plots
    "grid_alpha": 0.3,
    "spine_visible": ["bottom", "left"],  # Minimal spines
    "dpi": 300,
    "format": "pdf",
    "tight_layout": True,
}
```

Generate a shared style file `src/analysis/plot_style.py`:
```python
import matplotlib.pyplot as plt
import matplotlib

def apply_paper_style():
    plt.rcParams.update({
        'font.family': 'serif',
        'font.size': 10,
        'axes.labelsize': 10,
        'axes.titlesize': 10,
        'xtick.labelsize': 8,
        'ytick.labelsize': 8,
        'legend.fontsize': 9,
        'figure.dpi': 300,
        'savefig.dpi': 300,
        'savefig.bbox': 'tight',
        'axes.grid': True,
        'grid.alpha': 0.3,
        'axes.spines.top': False,
        'axes.spines.right': False,
    })

# Colorblind-friendly palette (Wong, 2011)
COLORS = ['#0072B2', '#E69F00', '#009E73', '#CC79A7',
          '#D55E00', '#56B4E9', '#F0E442']
```

Have all `plot_*.py` import and call `apply_paper_style()`.

### Step 7 — Final VLM Review (All Figures Together)

Send ALL figures to VLM in a single review:

```
"You are reviewing the complete figure set for a {venue} paper.
Evaluate:
1. Are the figures consistent in style (colors, fonts, layout)?
2. Do the figures tell a coherent visual story?
3. Could any figures be merged to save space?
4. Is the figure order logical?
5. Are all figures referenced in the text?
Overall figure quality score: 1-10"
```

### Step 8 — Register and Log

Update `EXPERIMENT_LOG.md`:
```markdown
## FIGURE-POLISH-{date}
- **Figures processed**: {n}
- **Total fix rounds**: {total across all figures}
- **Issues found**: {count by priority}
- **Issues fixed**: {count}
- **Final VLM score**: {score}/10
- **Style file**: src/analysis/plot_style.py
```

Update `figures/FIGURE_NOTES.md` (for paper writing):
```markdown
## Figure 1: main_comparison.pdf
- **Description**: Bar chart comparing our method vs 3 baselines on {metric}
- **Data source**: results/exp1/metrics.json
- **Key observation**: Our method outperforms all baselines by 2-5%
- **LaTeX caption draft**: "Comparison of {metric} across methods. Error bars
  show standard deviation over 3 seeds. Our method (blue) achieves..."

## Figure 2: ...
```

## Output

- Polished figures in `figures/` (overwriting originals, originals backed up in `figures/backup/`)
- Updated `src/analysis/plot_*.py` files
- `src/analysis/plot_style.py` shared style
- `figures/FIGURE_NOTES.md` with descriptions and caption drafts
- Log entry in `EXPERIMENT_LOG.md`

## Integration with Pipeline

In `/ars-12-pipeline`, figure polish runs after `/ars-07-analysis` (Phase 5) and before `/ars-09-writer` (Phase 6):

```
Phase 5: /ars-07-analysis → generates raw figures
Phase 5.5: /ars-08-figure-polish → iteratively refines figures via VLM
Phase 6: /ars-09-writer → uses polished figures + FIGURE_NOTES.md for drafting
```

The `/ars-09-writer` skill reads `figures/FIGURE_NOTES.md` for caption drafts and
figure descriptions, making the writing phase more accurate.

## Boundaries

- **DO**: Critique figures, edit plot code, re-render, apply consistent style, merge subplots
- **DO NOT**: Change the underlying data, modify experiment results, create misleading visualizations
- **DO NOT**: Remove error bars to make results "look cleaner"
- **DO NOT**: Truncate axes to exaggerate differences (unless explicitly noted in caption)
- If a figure reveals a problem with the data → flag for `/ars-07-analysis`, don't hide it
