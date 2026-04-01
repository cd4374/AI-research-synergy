---
name: ars-03-literature
description: Search, extract, and organize research literature for ARS.
---

# Skill: Literature

> **Role**: Search, read, extract, and organize research literature.
> Build the evidence foundation that all claims rest on.

## Trigger Keywords

literature, papers, search papers, related work, SOTA, state of the art,
survey, arXiv, scholar, citations, 文献, 论文搜索, 相关工作

## Inputs

- Research topic / search query — from user message or `PROJECT_STATE.md`
- (Optional) Specific paper titles or arXiv IDs to analyze
- (Optional) `EVIDENCE.md` to avoid duplicate evidence

## Procedure

### Step 1 — Generate Search Strategy

From the research topic, generate 3–5 search queries:

```
Queries:
1. "{core method} {domain}" — main topic
2. "{alternative approach} {domain}" — competing methods
3. "{core method} limitations" — known weaknesses
4. "{domain} benchmark comparison" — baselines & metrics
5. "{core method} recent 2025 2026" — latest work
```

### Step 2 — Search Multiple Sources

Search using available tools in this priority order:

1. **Semantic Scholar API** (via web search): `site:semanticscholar.org {query}`
2. **arXiv** (via web search): `site:arxiv.org {query}`
3. **Google Scholar** (via web search): `{query} research paper`

For each source, collect top 5–10 results. Target: **15–30 papers total** before dedup.

### Step 3 — Deduplicate and Filter

Remove duplicates (same title or arXiv ID). Filter by:
- Relevance to research questions (high/medium/low → keep high+medium)
- Recency (prefer last 3 years, but include seminal older work)
- Venue quality (prefer top venues: NeurIPS, ICML, ICLR, ACL, CVPR, AAAI, etc.)

Target: **10–20 papers** after filtering.

### Step 4 — Extract Per-Paper Summary

For each selected paper, extract:

```markdown
### [P-{seq}] {Title}
- **Authors**: {first author et al.}
- **Venue**: {venue, year}
- **arXiv**: {link if available}
- **Core idea**: {1–2 sentences}
- **Method**: {key technical approach}
- **Results**: {main quantitative results}
- **Limitations**: {stated or inferred limitations}
- **Relevance**: {how it relates to our work}
```

### Step 5 — Build SOTA Comparison Table

Create `results/sota_table.md`:

```markdown
# State-of-the-Art Comparison

| Method | Venue | Metric1 | Metric2 | Key Difference from Ours |
|--------|-------|---------|---------|--------------------------|
| {name} | {venue year} | {value} | {value} | {difference} |
```

Include at least 3 baselines and the current SOTA.

### Step 6 — Register Evidence

For each paper that provides citable facts, add to `EVIDENCE.md`:

```markdown
- [EVD-{seq}] {claimable fact from the paper}
  - source: literature
  - paper: [P-{seq}] {title}
  - artifact: results/sota_table.md
  - task: {current task ID}
  - strength: direct | indirect
  - validation: pending_review
```

Evidence types to extract:
- **Quantitative results** others achieved (for comparison)
- **Known limitations** of existing methods (motivation for our work)
- **Methodological facts** (standard practices in the field)
- **Background facts** (established knowledge)

### Step 7 — Identify Gaps

Write a "Research Gap Analysis" section in `NARRATIVE.md`:

```markdown
## Literature Gap Analysis ({date})

### What exists
{summary of current SOTA and approaches}

### What's missing
{specific gaps our work addresses}

### Our positioning
{how our proposed approach fills the gap}

### Risks
{potential overlap with concurrent/unpublished work}
```

### Step 8 — Novelty Pre-Check

Before concluding, do a quick novelty sanity check:
- Is there a paper doing **exactly** what we propose? If yes → flag in NARRATIVE.md
- Is there a paper that's **very close**? If yes → note differentiation needed
- Add a `NOVELTY_STATUS` field to `PROJECT_STATE.md`: `clear` | `caution` | `conflict`

## Output

- Paper summaries (in `NARRATIVE.md` or standalone `lit_review.md`)
- `results/sota_table.md`
- New entries in `EVIDENCE.md`
- Gap analysis in `NARRATIVE.md`
- Novelty status update in `PROJECT_STATE.md`

## Boundaries

- **DO**: Search, read, extract, summarize, compare, register evidence
- **DO NOT**: Run experiments, make novel claims, write paper sections, modify code
- If a paper needs deeper analysis (e.g., reproducing their code), create a task for `/ars-04-experiment`
