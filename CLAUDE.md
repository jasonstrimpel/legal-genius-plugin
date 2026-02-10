# CLAUDE.md

This file provides guidance to Claude Code when working with this plugin.

## What This Is

A Claude Code plugin (v1.0.0) that provides the `/legal-genius` command — a multi-phase legal contract gap analysis pipeline. It reads target contracts (PDFs) from integrating firms, compares them clause-by-clause against Andersen Consulting benchmark templates (DOCX), and produces:

1. An exhaustive gap analysis report as a formatted Word document
2. A redlined version of the target contract with tracked changes showing recommended language

There is no build system, package manager, or test suite. The codebase is markdown agent definitions, a command orchestrator, and the bundled docx skill (Python scripts + XSD schemas) for Word document manipulation.

## Plugin Structure

```
.claude-plugin/plugin.json    # Plugin manifest
commands/legal-genius.md       # Main orchestrator — the /legal-genius command
agents/                        # 7 specialized agent prompts (markdown + YAML frontmatter)
skills/docx/                   # Bundled docx skill from anthropics/skills
benchmarks/                    # User places Andersen benchmark .docx templates here
targets/                       # User places target contract PDFs here, organized by firm
```

## Architecture

The orchestrator (`commands/legal-genius.md`) drives a per-contract pipeline that processes all target contracts for a firm in batch. It never reads contracts or writes analysis itself — it delegates everything to agents via the Task tool.

**Phase flow:** Initialize → [Per Contract: Classify → Analyze → Audit → PDF-to-Markdown → (Convert + Format in parallel) → Redline] → Completion

**Coordination model:** File-based. Each agent reads source documents and writes output files. The orchestrator passes file paths and metadata between agents via Task prompts.

**Output directory:** `analysis/{Firm_Slug}/{contract-slug}/` where contract-slug is `{YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-{serial}`.

### Agent Roles

| Agent | Purpose | Tools |
|-------|---------|-------|
| `contract-classifier` | Read target PDF, classify contract type, select benchmark | Read, Write, Glob |
| `gap-analyzer` | Exhaustive clause-by-clause gap analysis (all gaps, no cap) | Read, Write |
| `audit-reviewer` | Reread source docs, fact-check and revise draft analysis | Read, Write |
| `pdf-to-markdown` | Convert target PDF to structured markdown with clause anchors | Read, Write |
| `target-to-docx` | Convert structured markdown to clean docx with bookmarks (base for redlining) | Read, Write, Bash, Skill |
| `redline-editor` | Apply tracked changes to target docx per gap recommendations | Read, Write, Bash, Edit |
| `report-formatter` | Convert gap analysis markdown to formatted Times Roman docx | Read, Write, Bash, Skill |

### Naming Convention

All files: `{YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-{DocType}[-redlined]-{serial}.{ext}`

- Firm_Slug and Client_Slug use Title_Case with underscores
- Serial is a 6-character alphanumeric UUID fragment
- ContractType: MSA, SOW, JAL, NDA, SUBK, AMEND

## Making Changes

- Each agent's YAML frontmatter defines its `tools` — only listed tools are available
- Agent input/output file paths are defined in both the agent and the orchestrator; update both if changing
- The orchestrator references agents by filename (minus `.md`); renaming requires updating `commands/legal-genius.md`
- The `report-formatter`, `target-to-docx`, and `redline-editor` agents use the bundled docx skill at `skills/docx/`

## Dependencies

- `npm install -g docx` — docx-js for creating formatted Word documents
- Python packages: `lxml`, `defusedxml` — XML validation and repair
- `pandoc` — text extraction from docx benchmarks
