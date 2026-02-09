# Legal Genius

AI-powered legal contract gap analysis plugin for [Claude Code](https://claude.ai/code).

Analyzes target contracts from integrating firms against Andersen Consulting benchmark templates, producing exhaustive gap analysis reports and redlined documents with tracked changes.

## Features

- **Batch processing** — Analyzes all target contracts for a firm in a single run
- **Exhaustive gap analysis** — Identifies ALL contract gaps (no cap), scored on legal and business risk
- **Redlined output** — Produces a redlined version of each target contract with tracked changes showing recommended language
- **Formatted reports** — Generates Word documents with professional formatting (Times New Roman, structured tables)
- **Audit verification** — Every draft analysis is fact-checked against source documents before finalization

## Installation

Install the plugin in your Claude Code project:

```
/install-plugin legal-genius-plugin
```

### Dependencies

| Dependency | Purpose | Install |
|-----------|---------|---------|
| LibreOffice | PDF to docx conversion | `brew install --cask libreoffice` |
| pandoc | Benchmark docx text extraction | `brew install pandoc` |
| docx (npm) | Word document creation | `npm install -g docx` |
| lxml, defusedxml | XML validation and repair | `pip install lxml defusedxml` |

## Usage

1. Place benchmark templates in `benchmarks/` (e.g., `MSA-001.docx`)
2. Place target contract PDFs in `targets/{Firm_Name}/`
3. Run `/legal-genius`
4. Enter the integrating firm name when prompted

The plugin will process each target contract through a 6-step pipeline:

1. **Classify** — Identify contract type and select matching benchmark
2. **Analyze** — Exhaustive clause-by-clause gap analysis
3. **Audit** — Independent fact-check of every claim and score
4. **Convert** — Transform target PDF to docx (parallel)
5. **Format** — Generate gap analysis report as Word document (parallel)
6. **Redline** — Apply tracked changes to target contract

## Output

For each contract analyzed:

```
analysis/{Firm_Slug}/{YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-{serial}/
├── drafts/
│   └── ...-gap-analysis-draft-{serial}.md    # Audited markdown draft
├── ...-gap-analysis-{serial}.docx            # Formatted Word report
└── ...-redlined-{serial}.docx                # Target with tracked changes
```

### Naming Convention

All files follow: `{YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-{DocType}[-redlined]-{serial}.{ext}`

- **Firm_Slug / Client_Slug** — Title_Case with underscores (e.g., `Andersen_Consulting`)
- **ContractType** — MSA, SOW, JAL, NDA, SUBK, AMEND
- **serial** — 6-character alphanumeric from UUID

## Architecture

```
commands/legal-genius.md       # Orchestrator — dispatches agents, manages pipeline
agents/
  contract-classifier.md       # Reads PDF, classifies type, selects benchmark
  gap-analyzer.md              # Exhaustive clause-by-clause analysis
  audit-reviewer.md            # Fact-checks draft against source documents
  target-to-docx.md            # Converts PDF to docx via LibreOffice
  redline-editor.md            # Applies tracked changes to target docx
  report-formatter.md          # Converts markdown to formatted Word document
skills/docx/                   # Bundled docx skill (scripts, schemas, validators)
```

## License

MIT
