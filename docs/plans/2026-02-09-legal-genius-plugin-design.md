# Legal Genius Plugin Design

## Overview

A Claude Code plugin that performs automated legal contract gap analysis for Andersen Consulting. It compares target contracts (from integrating firms) against benchmark templates, producing:

1. An exhaustive gap analysis report (all gaps, no cap) as formatted .docx
2. A redlined version of the target contract with tracked changes showing recommended language

The plugin processes all target contracts for a firm in a single batch run.

## Plugin Structure

```
legal-genius-plugin/
├── .claude-plugin/plugin.json
├── CLAUDE.md
├── commands/
│   └── legal-genius.md              # Main orchestrator command
├── agents/
│   ├── contract-classifier.md       # Agent 1: classify target, select benchmark
│   ├── gap-analyzer.md              # Agent 2: exhaustive gap analysis
│   ├── audit-reviewer.md            # Agent 3: fact-check and revise draft
│   ├── target-to-docx.md            # Agent 4: convert target PDF to docx
│   ├── redline-editor.md            # Agent 5: apply tracked changes
│   └── report-formatter.md          # Agent 6: gap analysis markdown to docx
├── skills/
│   └── docx/                        # Bundled from anthropics/skills
│       ├── SKILL.md
│       ├── LICENSE.txt
│       └── scripts/
│           ├── __init__.py
│           ├── accept_changes.py
│           ├── comment.py
│           └── office/
│               ├── unpack.py
│               ├── pack.py
│               ├── validate.py
│               ├── soffice.py
│               ├── helpers/
│               ├── schemas/
│               └── validators/
├── benchmarks/                      # User places benchmark .docx files here
│   └── {ContractType}-{serial}.docx # e.g., MSA-001.docx
└── targets/                         # User places target PDFs here by firm
    └── {Firm_Name}/
        └── {contract}.pdf
```

## Naming Convention

### Universal File Naming

All files follow this format:

```
{YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-{DocType}[-redlined]-{serial}.{ext}
```

Where:
- `{Firm_Slug}` - Title_Case with underscores (e.g., `Andersen_Consulting`)
- `{Client_Slug}` - Title_Case with underscores (e.g., `Widgets_Corp`)
- `{ContractType}` - MSA, SOW, JAL, NDA, SUBK, AMEND
- `{DocType}` - `gap-analysis-draft`, `gap-analysis`, `target`, etc.
- `[-redlined]` - only present on the redlined docx
- `{serial}` - 6-character alphanumeric from UUID (e.g., `9e761a`)

### Output Directory

```
analysis/{Firm_Slug}/{YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-{serial}/
├── drafts/
│   └── {YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-gap-analysis-draft-{serial}.md
├── {YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-gap-analysis-{serial}.docx
└── {YYYY-MM-DD}-{Firm_Slug}-{Client_Slug}-{ContractType}-redlined-{serial}.docx
```

### Example

For Andersen Consulting analyzing an MSA from Widgets Corp:

```
analysis/Andersen_Consulting/2026-02-09-Andersen_Consulting-Widgets_Corp-MSA-9e761a/
├── drafts/
│   └── 2026-02-09-Andersen_Consulting-Widgets_Corp-MSA-gap-analysis-draft-9e761a.md
├── 2026-02-09-Andersen_Consulting-Widgets_Corp-MSA-gap-analysis-9e761a.docx
└── 2026-02-09-Andersen_Consulting-Widgets_Corp-MSA-redlined-9e761a.docx
```

## Pipeline Architecture

### Phase 0: Initialize (Orchestrator)

1. Ask user for the integrating firm name
2. Generate `{Firm_Slug}` (Title_Case, underscores for spaces)
3. List `targets/{Firm_Slug}/` to discover available contracts
4. List `benchmarks/` to discover available benchmark templates
5. Confirm batch processing of all discovered targets
6. Generate a 6-char UUID serial for each contract analysis

### Phase 1: Per-Contract Pipeline

For each target contract, the orchestrator dispatches agents sequentially:

```
Step 1 → contract-classifier     [reads target PDF directly]
Step 2 → gap-analyzer            [reads target PDF + benchmark docx directly]
Step 3 → audit-reviewer          [rereads both source docs, fact-checks draft]
       ┌──────────────────────────────┐
Step 4 → target-to-docx       [PARALLEL]  Step 6 → report-formatter
       └──────────────────────────────┘
Step 5 → redline-editor           [needs Steps 3 + 4 output]
```

**Parallelism:**
- Steps 4 (target-to-docx) and 6 (report-formatter) run in parallel
- Step 5 (redline-editor) blocks on both Steps 3 and 4
- Contracts are processed sequentially within the batch to manage context

### Phase 2: Batch Completion (Orchestrator)

- Print summary table of all contracts processed
- List all output files with full paths
- Note any contracts that failed or required user intervention

## Agent Specifications

### Agent 1: contract-classifier

**Purpose:** Read target PDF, classify contract type, select matching benchmark.

**Tools:** `Read, Write, Glob`

**Inputs:**
- Target contract PDF path
- Benchmarks directory listing

**Workflow:**
1. Read the target contract PDF
2. Classify contract type (MSA, SOW, JAL, NDA, SUBK, AMEND)
3. List all available benchmarks in `benchmarks/`
4. Select the most appropriate benchmark based on contract type and subject matter
5. State selection and rationale

**Output:** Classification result written to orchestrator context:
- Contract type
- Client name (extracted from contract)
- Selected benchmark file path
- Rationale for benchmark selection

**Error handling:**
- If contract type cannot be determined: ask user
- If no appropriate benchmark exists: halt and notify user
- If client name cannot be extracted: ask user

### Agent 2: gap-analyzer

**Purpose:** Perform exhaustive clause-by-clause gap analysis between target and benchmark.

**Tools:** `Read, Write`

**Inputs:**
- Target contract PDF path
- Selected benchmark docx path
- Classification result from Agent 1

**Workflow:**
1. Read the target contract PDF completely
2. Read the selected benchmark docx completely
3. Compare clause by clause
4. Identify ALL gaps (no cap) — every deviation, omission, or weakness
5. Score each gap on Legal Risk (1-5) and Business Risk (1-5)
6. Rank all gaps by combined score (high to low)
7. Assess consulting relevance of each benchmark clause
8. Write draft gap analysis

**Output:** `{contract-slug}/drafts/{naming}-gap-analysis-draft-{serial}.md`

**Gap Analysis Sections:**

#### Section A: Executive Summary
- Target contract identification (parties, effective date, term)
- Benchmark used for comparison and rationale
- Overall risk assessment (LOW/MEDIUM/HIGH/CRITICAL) with detailed legal and business rationale
- Specific remediation recommendations

#### Section B: Gap Analysis Table (ALL gaps, ranked by combined score)

| Clause | Benchmark Summary | Target Language | Consulting Relevance | Legal Risk (1-5) | Business Risk (1-5) | Recommended Action |
|--------|-------------------|-----------------|---------------------|------------------|--------------------|--------------------|
| *clause* | *summary* | *quote or "ABSENT"* | HIGH/MEDIUM/LOW/N/A | 1-5 | 1-5 | *specific action* |

#### Section C: Risk Scoring Rubric

| Score | Legal Risk | Business Risk |
|-------|-----------|---------------|
| 5 | Material liability, litigation exposure, or regulatory violation | Significant financial loss (>$500K) or client relationship damage |
| 4 | Enforceable obligations materially unfavorable to firm | Moderate financial exposure ($100K-$500K) or operational constraints |
| 3 | Ambiguous language creating potential disputes | Minor financial risk (<$100K) or resource inefficiency |
| 2 | Suboptimal but low probability of adverse outcome | Inconvenience or minor friction |
| 1 | Technical gap with negligible practical impact | No meaningful business impact |

#### Section D: Relevance Assessment
- Flag benchmark clauses addressing tax-specific scenarios
- Flag regulated tax practice requirements
- Flag provisions inapplicable to consulting services
- Mark as "NOT APPLICABLE TO CONSULTING" with brief explanation

### Agent 3: audit-reviewer

**Purpose:** Fact-check the draft gap analysis against source documents.

**Tools:** `Read, Write`

**Inputs:**
- Draft gap analysis markdown
- Target contract PDF path
- Selected benchmark docx path

**Workflow:**
1. Read the draft gap analysis
2. Reread the target contract PDF completely
3. Reread the benchmark docx completely
4. Verify every gap claim is factually accurate:
   - Quoted target language matches actual contract text
   - Benchmark summaries accurately reflect benchmark provisions
   - Clause references are correct
   - Risk scores are justified
   - "ABSENT" designations are confirmed absent
5. Fix any discrepancies:
   - Misquotes or paraphrasing errors
   - Wrong clause references
   - Scoring inconsistencies
   - Missing gaps discovered on reread
6. Ensure all recommendations are from Andersen Consulting's perspective
7. Overwrite the draft with the revised version

**Output:** Revised `{contract-slug}/drafts/{naming}-gap-analysis-draft-{serial}.md`

### Agent 4: target-to-docx

**Purpose:** Convert target PDF to clean docx as the base for redlining.

**Tools:** `Read, Write, Bash, Skill` (docx skill)

**Inputs:**
- Target contract PDF path

**Workflow:**
1. Convert PDF to docx using LibreOffice:
   ```bash
   python skills/docx/scripts/office/soffice.py --headless --convert-to docx target.pdf
   ```
2. Validate the converted docx:
   ```bash
   python skills/docx/scripts/office/validate.py converted.docx
   ```
3. Write the clean docx to the contract output directory

**Output:** Clean docx version of the target contract (intermediate file used by Agent 5)

### Agent 5: redline-editor

**Purpose:** Apply tracked changes to the target docx based on gap analysis recommendations.

**Tools:** `Read, Write, Bash, Edit` (docx skill)

**Inputs:**
- Target docx (from Agent 4)
- Audited gap analysis markdown (from Agent 3)

**Workflow:**
1. Read the audited gap analysis to get the full gap table with recommended actions
2. Unpack the target docx:
   ```bash
   python skills/docx/scripts/office/unpack.py target.docx unpacked/
   ```
3. For each gap in the analysis:
   - Locate the corresponding clause in `unpacked/word/document.xml`
   - Apply tracked changes using OOXML tracked change markup:
     - `<w:del>` for text to be removed (author="Andersen Consulting")
     - `<w:ins>` for new recommended language (author="Andersen Consulting")
     - Preserve `<w:rPr>` formatting from original runs
     - Use smart quotes for new content (`&#x201C;`, `&#x201D;`, `&#x2019;`)
   - For ABSENT clauses: insert new paragraphs with `<w:ins>` tracked changes
4. Validate and pack:
   ```bash
   python skills/docx/scripts/office/pack.py unpacked/ output.docx --original target.docx
   ```

**Output:** `{naming}-redlined-{serial}.docx`

**Critical considerations:**
- Author for all tracked changes: "Andersen Consulting"
- Use `<w:delText>` (not `<w:t>`) inside `<w:del>` elements
- Use `<w:delInstrText>` (not `<w:instrText>`) inside `<w:del>` elements
- Add `xml:space="preserve"` to `<w:t>` elements with leading/trailing whitespace
- Mark entire paragraph deletions with `<w:del/>` in `<w:pPr><w:rPr>`
- IDs must be unique across all tracked change elements

### Agent 6: report-formatter

**Purpose:** Convert gap analysis markdown to formatted docx report.

**Tools:** `Read, Write, Bash, Skill` (docx skill)

**Inputs:**
- Audited gap analysis markdown (from Agent 3)
- Contract metadata (firm, client, type, serial, date)

**Workflow:**
1. Read the audited gap analysis markdown
2. Generate docx using docx-js (npm `docx` package):
   - Page size: US Letter (12240 x 15840 DXA)
   - Font: Times New Roman, size 11pt (22 half-points) body
   - Headings: H1 = 16pt bold, H2 = 14pt bold, H3 = 12pt bold
   - Tables: bordered, DXA widths, cell padding
   - Minimal formatting — no headers, footers, or page numbers unless specified
3. Validate:
   ```bash
   python skills/docx/scripts/office/validate.py output.docx
   ```

**Output:** `{naming}-gap-analysis-{serial}.docx`

**Formatting rules:**
- Override built-in heading styles with Times New Roman
- Use `WidthType.DXA` for all table widths (never percentage)
- Tables need dual widths (columnWidths + cell width)
- Use `LevelFormat.BULLET` for lists (never unicode bullets)
- Set page size explicitly to US Letter

## Orchestrator Command Design

The orchestrator (`commands/legal-genius.md`) follows the lead-genius pattern:

### YAML Frontmatter
```yaml
---
description: Run legal contract gap analysis pipeline for Andersen Consulting
allowed-tools: [Read, Write, Bash, Glob, Grep, Task, Skill]
model: opus
---
```

### Orchestrator Rules
- Keep responses SHORT (2-3 sentences between phases)
- Delegate ALL analysis and document production to subagents via Task tool
- NEVER read contracts or produce analysis itself
- Generate UUID serial at the start of each contract's pipeline

### Agent Dispatch Pattern

For each agent, the orchestrator:
1. Constructs the prompt with all required context (paths, metadata, prior results)
2. Dispatches via Task tool with the appropriate `subagent_type`
3. Waits for completion
4. Confirms output file was written
5. Proceeds to next step

For the parallel fork (Steps 4 + 6):
- Dispatches both agents in a single message with 2 Task calls
- Waits for both to complete before proceeding to Step 5

## Dependencies

- **pandoc**: Text extraction from docx (reading benchmarks)
- **docx** (npm): `npm install -g docx` — creating new docx files (Agent 6)
- **LibreOffice**: PDF to docx conversion (Agent 4), PDF rendering
- **Python packages**: `lxml`, `defusedxml` — XML validation and repair
- **Claude Code**: Read tool supports PDF natively (Agents 1, 2, 3)

## Constraints

- Always read the target contract before selecting a benchmark
- If no appropriate benchmark exists, notify the user and request guidance
- If client name cannot be determined from the contract, ask the user
- All recommendations written from Andersen Consulting's perspective
- Benchmark clauses inherited from Andersen Tax that are inapplicable to consulting must be flagged
- Gap analysis table has NO cap — include every gap found
- Every gap with a recommended action must have a corresponding tracked change in the redlined document
