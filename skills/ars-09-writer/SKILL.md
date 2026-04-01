---
name: ars-09-writer
description: Draft paper sections with strict claim-evidence binding.
---

# Skill: Writer

> **Role**: Draft paper sections with every claim bound to evidence.
> Generate LaTeX or Markdown. Never fabricate citations or data.

## Trigger Keywords

write, draft, paper, section, abstract, introduction, method, results,
conclusion, limitations, related work, 写论文, 写作, 撰写

## Inputs

- `PROJECT_STATE.md` — project goal, scope
- `CLAIMS.md` — approved claims with evidence links
- `EVIDENCE.md` — all evidence items
- `NARRATIVE.md` — analysis summaries, gap analysis
- `results/sota_table.md` — SOTA comparison
- `figures/` — generated figures
- `figures/FIGURE_NOTES.md` — figure descriptions and caption drafts, if available
- (Optional) Venue target for formatting

## Procedure

### Step 1 — Generate Paper Plan

Before writing, create `paper/PAPER_PLAN.md`:

```markdown
# Paper Plan

## Title
{working title}

## Target Venue
{venue} — {page limit} pages, {format}

## Section Outline
1. Abstract (~200 words)
2. Introduction (~1.5 pages)
   - Motivation: [CLM-xxx]
   - Problem statement: [CLM-xxx]
   - Key contribution: [CLM-xxx]
3. Related Work (~1 page)
   - {subtopic 1}: [P-xxx], [P-xxx]
   - {subtopic 2}: [P-xxx], [P-xxx]
4. Method (~2 pages)
   - Overview: [CLM-xxx]
   - Component A: [CLM-xxx]
   - Component B: [CLM-xxx]
5. Experiments (~2 pages)
   - Setup: [CLM-xxx]
   - Main results: [CLM-xxx], Figure {x}, Table {x}
   - Ablation: [CLM-xxx], Figure {x}
6. Analysis & Discussion (~0.5 page)
   - Key finding: [CLM-xxx]
   - Negative result: [CLM-xxx]
7. Limitations (~0.3 page) [MANDATORY]
8. Conclusion (~0.3 page)

## Claim-Section Mapping
{every claim in CLAIMS.md must appear in at least one section}

## Uncovered Claims
{list any claims without assigned sections — these MUST be resolved}
```

### Step 2 — Draft Each Section

Write sections in order. For each section:

**Claim binding rule:** Every key assertion must reference a claim ID inline:

```markdown
Our method achieves a 3.2% improvement over the strongest baseline [CLM-001],
while requiring 40% fewer parameters [CLM-005]. This improvement is statistically
significant (p = 0.003, paired t-test) [CLM-001].
```

The claim IDs will be stripped in the final paper but are essential during drafting
for traceability.

**Section-specific rules:**

**Abstract:**
- Must mention the core contribution (1 claim)
- Must include at least one quantitative result
- Must mention a limitation or scope constraint
- ≤ 250 words (or venue-specific limit)

**Introduction:**
- Motivate the problem with literature evidence [EVD-xxx]
- State the gap clearly
- List contributions (each backed by a claim)
- Do NOT overclaim: "state-of-the-art" only if SOTA table proves it

**Related Work:**
- Cite real papers only — use papers from `/ars-03-literature` output
- Compare approaches, don't just list papers
- Position our work relative to prior work

**Method:**
- Describe what we actually built, not what we wish we built
- Include enough detail to reproduce (or reference code artifact)
- Use figures for architecture if available

**Experiments:**
- Setup: datasets, baselines, metrics, hyperparameters, compute
- Results: reference figures and tables by name
- Every number must trace to `results/` artifacts
- **Never round results to make them look better**

**Limitations (MANDATORY):**
- At least 3 limitations
- Include: statistical limitations (small seed count), scope limitations,
  known failure cases, generalization concerns
- **DO NOT write a limitations section that reads as humble-brag**

**Conclusion:**
- Summarize findings — reference only claims from Results section
- No new claims in conclusion
- Mention future work

### Step 3 — Anti-Hallucination Citation Check

Before finalizing, verify every citation:

1. List all references mentioned in the draft
2. Cross-check each against papers found by `/ars-03-literature`
3. For each reference, ensure:
   - Author names are correct
   - Year is correct
   - Venue is correct
   - The cited claim actually appears in that paper
4. **REMOVE any reference you cannot verify** — better to have fewer citations
   than fabricated ones

If a citation is needed but not in our literature collection:
→ Add a `TODO: verify citation` marker
→ Do NOT invent a plausible-looking reference

### Step 4 — Build Reference List

Generate `paper/references.bib` using only verified citations.
Prefer fetching BibTeX from DBLP or Semantic Scholar over writing it manually.

### Step 5 — Register Claims

For any new claims introduced during writing, add to `CLAIMS.md`:

```markdown
- [CLM-{seq}] "{exact claim statement}"
  - type: novel_contribution | comparative_result | methodological | limitation | background_fact
  - severity_if_wrong: low | medium | high | critical
  - status: drafted
  - evidence: [EVD-xxx], [EVD-xxx]
  - section: {which section it appears in}
```

**Claims without evidence get status `needs_evidence`** — they MUST be resolved
before Gate F1 can pass.

### Step 6 — Self-Check Before Output

Run through this checklist:
- [ ] Every claim in Results/Conclusion links to evidence
- [ ] No fabricated citations
- [ ] Limitations section exists and is substantive
- [ ] Negative results are reported honestly
- [ ] Figures/tables referenced actually exist in `figures/`
- [ ] Numbers in text match numbers in `results/`
- [ ] No "delve", "pivotal", "landscape", "groundbreaking" (AI writing tells)

## Output

- Paper sections in `paper/` (as `.md` or `.tex` files)
- `paper/PAPER_PLAN.md`
- `paper/references.bib`
- Updated `CLAIMS.md` with new claims
- Updated section-claim mapping

## Boundaries

- **DO**: Draft sections, bind claims to evidence, cite verified papers, report honestly
- **DO NOT**: Fabricate citations, make unsupported claims, hide negative results,
  run experiments, modify figures
- If evidence is missing for a claim → mark it `needs_evidence`, don't fabricate support
