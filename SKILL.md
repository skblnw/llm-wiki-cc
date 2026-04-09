---
name: llm-wiki
description: Build and maintain a personal knowledge base wiki from raw source documents. Use whenever the user wants to create a wiki, knowledge base, or organized research notes from documents — even if they don't say "wiki" explicitly. Triggers on document ingestion, knowledge organization, research synthesis, building interlinked notes, processing articles or papers into structured collections, or maintaining a growing body of knowledge. If the user is adding documents and wants them organized, cross-referenced, and queryable, this is the skill to use.
---

# LLM Wiki

Build a persistent, compounding knowledge base from raw source documents. You read sources, extract knowledge, and maintain a wiki of interlinked markdown pages. The human curates sources and asks questions; you handle the summarizing, cross-referencing, filing, and bookkeeping.

The wiki is not a one-shot artifact — it compounds. Every source ingested enriches existing pages. Every question answered can become a new page. Cross-references are maintained, contradictions are flagged, and the synthesis reflects everything ingested so far.

## Commands

```
/wiki                     — ingest all unprocessed sources in raw/
/wiki ingest              — same as above
/wiki ingest <file>       — ingest a specific source
/wiki query "<question>"  — answer a question from the wiki
/wiki lint                — health-check the wiki
/wiki status              — show wiki stats
```

## Architecture

Three layers, each with a clear owner:

| Layer | Location | Owner | Purpose |
|-------|----------|-------|---------|
| Raw sources | `raw/` | Human | Immutable source documents — articles, papers, PDFs, images |
| Wiki | `wiki/` | LLM | Interlinked markdown pages — summaries, entities, concepts, syntheses |
| Schema | This skill + CLAUDE.md | Both | Conventions and domain-specific rules, co-evolved over time |

## Directory Layout

```
project/
├── raw/                            # Source documents (human-curated, immutable)
│   ├── research_report.md          # Synthesis report (start here if present)
│   ├── 2601.05388v2.pdf            # Supporting paper
│   └── ...
├── wiki/                           # Knowledge base (LLM-maintained)
│   ├── index.md                    # Content catalog — the map of everything
│   ├── log.md                      # Chronological operation record
│   ├── overview.md                 # High-level synthesis across all sources
│   ├── sources/                    # One summary page per source document
│   ├── entities/                   # People, organizations, models, tools, datasets
│   ├── concepts/                   # Ideas, methods, theories, phenomena
│   └── analyses/                   # Your syntheses, comparisons, filed answers
```

---

## STEP 1: Detect and classify sources

Scan `raw/` recursively. Classify each file:

| Type | Extensions | Notes |
|------|-----------|-------|
| document | .md, .txt, .rst | Check if it reads like a research paper (arxiv IDs, "Abstract", citations) — if so, treat as paper |
| paper | .pdf | Extract text; note that extraction can be lossy |
| image | .png, .jpg, .jpeg, .webp, .svg | Describe visually; extract data from charts/diagrams |
| data | .csv, .json, .xlsx | Summarize structure and key statistics |

Skip dot-files, `wiki/`, and any `graphify-out/` or build directories.

**Check for prior work**: Read `wiki/log.md` if it exists. Compare the list of already-ingested source files against what's in `raw/`. Only process new or modified sources (compare by filename — if a file has been ingested before, skip it unless the user explicitly asks to re-ingest).

**Priority order**: If `raw/` contains a file named `research_report.md` (or similar synthesis document), ingest it first. Synthesis reports reference other sources and give you the best map of the domain — they help you identify entities and concepts to watch for in subsequent sources.

## STEP 2: Read and understand each source

For each unprocessed source, read it thoroughly and extract:

1. **Entities** — people, organizations, models, tools, datasets, proteins, chemicals, or whatever the domain's "nouns" are. Each entity gets a name and a one-line description of its role.
2. **Concepts** — ideas, methods, theories, techniques, phenomena. The domain's "verbs" and abstractions.
3. **Claims** — specific assertions the source makes, especially quantitative results, comparisons, or conclusions. Note confidence and caveats.
4. **Relationships** — how entities and concepts connect: "X uses Y", "A contradicts B", "C extends D".
5. **Questions** — what this source leaves unanswered or what new questions it raises.

For PDFs: read page ranges to extract the full text. If text extraction is garbled in places, flag those sections rather than guessing.

For images: describe what you see. If it's a chart, extract the key data points. If it's a diagram, describe the components and their relationships.

## STEP 3: Write source summary pages

For each source, create `wiki/sources/<slug>.md`:

```markdown
---
title: "<Source Title>"
source_file: "raw/<filename>"
date_ingested: YYYY-MM-DD
type: document | paper | image | data
authors: ["Author Name"]
year: YYYY
---

# <Source Title>

## Key Takeaways
- Most important findings or arguments (3-5 bullets)

## Summary
2-4 paragraphs capturing the core content. Write for someone who hasn't read
the source but needs to understand what it contributes.

## Entities
- [[entities/<slug>]] — role in this source

## Concepts
- [[concepts/<slug>]] — how this source treats the concept

## Notable Claims
- Specific claim (with caveats and confidence)

## Open Questions
- Questions this source raises for further investigation
```

**Slug convention**: lowercase, hyphen-separated, descriptive. `research-report.md` becomes `research-report`. A paper by Airas & Zhang becomes `airas-zhang-2026`. Keep slugs stable — they become the permanent identifier.

## STEP 4: Create or update entity pages

For each entity extracted in Step 2:

**If the page doesn't exist**, create `wiki/entities/<slug>.md`:

```markdown
---
title: "<Entity Name>"
type: entity
aliases: ["alternate names or abbreviations"]
first_seen: "sources/<source-slug>"
---

# <Entity Name>

1-2 paragraph description synthesizing everything the wiki knows about this
entity. Update this paragraph as new sources are ingested.

## Key Facts
- Fact with [[source attribution]]
- Another fact

## Connections
- Related to [[entities/<other>]] — nature of the relationship
- Used by [[concepts/<concept>]] — how

## Mentioned In
- [[sources/<source>]] — context of how it appears
```

**If the page exists**, update it:
- Add new facts from the new source
- Revise the description if the new source changes the picture
- Add the source to "Mentioned In"
- If the new source contradicts existing content, note the contradiction explicitly: *"[[sources/source-a]] claims X, but [[sources/source-b]] argues Y."*

## STEP 5: Create or update concept pages

Same pattern as entities, but for ideas and methods. Create `wiki/concepts/<slug>.md`:

```markdown
---
title: "<Concept Name>"
type: concept
related: ["other-concept-slug"]
first_seen: "sources/<source-slug>"
---

# <Concept Name>

Description synthesizing all sources. Start broad, add depth as more sources
arrive.

## How It Works
Technical explanation at appropriate depth for the domain.

## Why It Matters
Significance and implications.

## Related Concepts
- [[concepts/<other>]] — nature of the connection

## Sources
- [[sources/<source>]] — what this source contributes to understanding this concept
```

When updating an existing concept page, strengthen or challenge the existing description based on the new source. The concept page should always reflect the *current best understanding* across all ingested sources.

## STEP 6: Cross-reference pass

After writing all pages for a source, do a linking pass:

1. Every `[[wikilink]]` must point to a page that exists. If you link to `[[entities/foo]]` and `wiki/entities/foo.md` doesn't exist, either create it (if there's enough to say) or remove the link and add "foo" to the "gaps" section of your lint notes.
2. If two entity or concept pages clearly relate but don't link to each other, add the cross-reference to both.
3. If a concept is mentioned on an entity page (or vice versa), make sure there's a `[[link]]`.

The goal: a reader browsing in Obsidian should be able to follow links from any page to discover related knowledge without dead ends.

## STEP 7: Update index.md

`wiki/index.md` is the master catalog. Rebuild it after each ingest:

```markdown
# Index

> N sources ingested, N entity pages, N concept pages, N analyses

## Sources
- [[sources/research-report]] — Overview of evolutionary intelligence in protein solvation
- [[sources/airas-zhang-2026]] — Original Airas & Zhang paper on knowledge distillation for ISMs

## Entities
- [[entities/esm3]] — Multimodal protein language model (Meta)
- [[entities/schake-gnn]] — Hybrid GNN architecture combining SchNet + SAKE

## Concepts
- [[concepts/implicit-solvent-models]] — Computational models replacing explicit water molecules
- [[concepts/knowledge-distillation]] — Training compact models from larger teacher models

## Analyses
- [[analyses/ism-paradigm-comparison]] — Comparison of solvation modeling approaches
```

Each entry: one link + one-line description. Keep it scannable — this is how you (and the human) navigate the wiki.

## STEP 8: Append to log.md

Record what happened. Every ingest gets an entry:

```markdown
## [YYYY-MM-DD] ingest | <Source Title>
Source: raw/<filename>
Created: sources/<slug>, entities/x, entities/y, concepts/a, concepts/b
Updated: entities/z (added new findings from this source)
Pages touched: N
```

Use a consistent prefix format so the log is grep-able: `## [YYYY-MM-DD] verb | subject`.

Also log queries and lint passes:
```markdown
## [YYYY-MM-DD] query | "What is the Schake architecture?"
Answer filed to: analyses/schake-architecture-explained

## [YYYY-MM-DD] lint | Health check
Found: 2 orphan pages, 1 dead link, 3 gaps
Fixed: dead link in entities/esm3, created stub for concepts/born-energy
```

## STEP 9: Write or update overview.md

After all sources for this session are ingested, write or update `wiki/overview.md`:

```markdown
---
title: "Overview"
type: overview
last_updated: YYYY-MM-DD
source_count: N
---

# Overview

High-level synthesis of what the wiki covers: the main domain, key themes,
central findings, and open questions. Write this as an executive summary —
someone reading only this page should understand the shape of the knowledge base.

## Main Themes
- Theme 1 — brief description
- Theme 2 — brief description

## Key Findings
- Finding with [[source attribution]]

## Open Questions
- Unresolved questions across sources

## Recent Activity
- Last ingest: [date] — [source title]
```

This page evolves: each ingest pass should refine it so it always reflects the current state of the wiki.

---

## Query Workflow

When the user asks a question:

1. Read `wiki/index.md` to find relevant pages
2. Read those pages (source summaries, entity pages, concept pages)
3. Synthesize an answer using `[[wikilinks]]` as citations
4. If the answer is substantial and reusable, offer to file it as `wiki/analyses/<slug>.md`
5. If you file it, update `index.md` and append to `log.md`

Good answers go beyond what any single page says — they synthesize across pages. That's the wiki's compounding value.

## Lint Workflow

When the user asks for a health check:

1. **Orphan pages** — pages with no inbound links from other wiki pages
2. **Dead links** — `[[wikilinks]]` pointing to pages that don't exist
3. **Stale claims** — assertions from early sources that later sources have superseded
4. **Contradictions** — places where sources disagree (flag with both sides, don't resolve)
5. **Gaps** — important concepts or entities mentioned but lacking their own page
6. **Missing cross-references** — pages that should link to each other but don't
7. **Thin pages** — pages with very little content that could be expanded with available information

Present findings as a checklist. Offer to fix the mechanical issues (dead links, orphans, missing cross-refs). Flag contradictions and gaps for the human to decide on.

---

## Conventions

**Wikilinks**: Use `[[relative-path]]` from wiki root. Example: `[[entities/esm3]]`, `[[concepts/knowledge-distillation]]`, `[[sources/research-report]]`. Obsidian resolves these as links between notes.

**Filenames**: Lowercase, hyphen-separated. `implicit-solvent-models.md`, `airas-zhang-2026.md`. Keep stable — renaming breaks links.

**Frontmatter**: Every wiki page gets YAML frontmatter with at least `title` and `type`. This enables Obsidian Dataview queries and helps the LLM classify pages quickly.

**Tone**: Clear, concise, reference-style. Prefer bullet points for facts, short paragraphs for context. Always attribute claims to sources. The wiki is a reference, not a narrative.

**Contradictions**: When sources disagree, present both views with attributions. Never silently pick a winner. The human decides what to believe.

**Equations**: Use LaTeX (`$...$` inline, `$$...$$` block) when the domain involves math. Obsidian renders these with MathJax.

## Multi-Source Parallel Ingest

For large batches (5+ unprocessed sources), use subagents for efficiency:

1. Assign each subagent a subset of sources
2. Each subagent runs Steps 2-3 (read source, write source summary) and returns the list of entities and concepts it identified
3. After all subagents complete, do a **merge pass**: deduplicate entities (different sources may name the same thing differently), resolve naming conflicts, then run Steps 4-6 (entity/concept pages, cross-references) on the merged set
4. Finish with Steps 7-9 (index, log, overview)

For 1-4 sources, process sequentially — simpler and the coordination overhead isn't worth it.

## Tips

- **Start with the synthesis document.** If `raw/` contains `research_report.md` or a similar overview, ingest it first. It maps the domain and tells you what entities and concepts to expect in supporting documents.
- **Entity vs. concept is fuzzy.** ESM3 is both a specific model (entity) and an instance of protein language models (concept). Use your judgment — if it has a proper name, it's probably an entity. If it's a general idea, it's a concept. Some things warrant both a page in `entities/` and a mention on a broader concept page.
- **Don't over-extract.** Not every noun needs an entity page. Create pages for things that are (a) mentioned in multiple sources, (b) central to the domain, or (c) have enough substance for a useful page. Minor mentions can live as bullets on other pages.
- **Filed answers compound.** When a query answer is filed as an analysis page, it becomes part of the wiki's knowledge. Future queries and ingests can build on it. Encourage the human to file good answers.
- **The wiki is the interface.** The human browses it in Obsidian (or any markdown viewer). Optimize for browsability: clear headings, scannable bullets, meaningful links. Graph view in Obsidian will show the structure — aim for a connected graph with no large orphan clusters.
