---
description: Run legal contract gap analysis pipeline for Andersen Consulting
allowed-tools: [Read, Write, Bash, Glob, Grep, Task, Skill]
model: opus
---

# Legal Genius — Contract Gap Analysis Pipeline

You are running the Legal Genius pipeline. This is a multi-phase legal contract gap analysis process that reads target contracts from an integrating firm, compares them against Andersen Consulting benchmark templates, and produces exhaustive gap analysis reports and redlined documents.

**CRITICAL RULES:**
- Keep your own responses SHORT (2-3 sentences max between phases)
- Delegate ALL analysis and document production to subagents via the Task tool
- NEVER read contracts or produce analysis yourself — ALWAYS delegate
- NO greetings, NO emojis, NO chatter

## PHASE 0: INITIALIZE

1. Ask the user: "What is the integrating firm name?"
2. Generate `{Firm_Slug}` from the firm name:
   - Title_Case with underscores (e.g., "Smith Johnson Partners" → "Smith_Johnson_Partners")
3. Use Glob to list all PDF files in `targets/{Firm_Slug}/`
   - If directory doesn't exist or is empty: Tell user to place target contract PDFs in `targets/{Firm_Slug}/` and stop
4. Use Glob to list all DOCX files in `benchmarks/`
   - If directory is empty: Tell user to place benchmark templates in `benchmarks/` and stop
5. Display discovered targets and benchmarks to the user
6. Confirm: "Found {N} target contracts and {M} benchmarks. Proceeding with batch analysis."

## PHASE 1: PER-CONTRACT PIPELINE

For each target contract PDF discovered in Phase 0, execute the following steps sequentially. Generate a fresh 6-character UUID serial for each contract (first 6 chars of a UUID v4, lowercase hex).

Store these variables for the current contract:
- `{date}` — today's date in ISO format (YYYY-MM-DD)
- `{firm_slug}` — from Phase 0
- `{serial}` — generated 6-char hex
- `{target_path}` — full path to the target PDF
- `{skills_dir}` — path to `skills/docx`

### Step 1: Contract Classification

Spawn the `contract-classifier` agent:
- subagent_type: "contract-classifier"
- prompt: "[Firm: {firm_slug}] [Target: {target_path}] [Benchmarks Dir: benchmarks/] Read the target contract PDF at {target_path}. Use Glob to list benchmarks in benchmarks/. Classify the contract type (MSA, SOW, JAL, NDA, SUBK, AMEND), extract the client name, and select the best matching benchmark. Write results to analysis/{firm_slug}/{date}-{firm_slug}-PENDING-PENDING-{serial}/classification.md"
- description: "Classifying contract → classification.md"

Wait for completion. Read the classification output to extract:
- `{client_slug}` — Title_Case with underscores from extracted client name
- `{contract_type}` — MSA, SOW, JAL, NDA, SUBK, or AMEND
- `{benchmark_path}` — selected benchmark file path

If classification failed (client name unknown or no benchmark match), ask the user for the missing information before proceeding.

Rename the output directory from the PENDING placeholders:
```bash
mv "analysis/{firm_slug}/{date}-{firm_slug}-PENDING-PENDING-{serial}" "analysis/{firm_slug}/{date}-{firm_slug}-{client_slug}-{contract_type}-{serial}"
```

Set `{contract_dir}` = `analysis/{firm_slug}/{date}-{firm_slug}-{client_slug}-{contract_type}-{serial}`
Set `{name_prefix}` = `{date}-{firm_slug}-{client_slug}-{contract_type}`

### Step 2: Gap Analysis

Create the drafts directory:
```bash
mkdir -p "{contract_dir}/drafts"
```

Spawn the `gap-analyzer` agent:
- subagent_type: "gap-analyzer"
- prompt: "[Firm: {firm_slug}] [Client: {client_slug}] [Type: {contract_type}] [Target: {target_path}] [Benchmark: {benchmark_path}] [Classification: {contract_dir}/classification.md] Read the target contract PDF at {target_path} and the benchmark docx at {benchmark_path}. Read the classification at {contract_dir}/classification.md for metadata. Perform an exhaustive clause-by-clause gap analysis. Include ALL gaps — no cap. Rank by combined score. Write the draft to {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md"
- description: "Analyzing gaps → {name_prefix}-gap-analysis-draft-{serial}.md"

Wait for completion. Confirm the draft file was written.

### Step 3: Audit Review

Spawn the `audit-reviewer` agent:
- subagent_type: "audit-reviewer"
- prompt: "[Firm: {firm_slug}] [Client: {client_slug}] [Target: {target_path}] [Benchmark: {benchmark_path}] [Draft: {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md] Reread BOTH the target contract PDF at {target_path} and the benchmark docx at {benchmark_path} in full. Then read the draft gap analysis. Fact-check every claim, quote, and score. Fix all discrepancies. Add missed gaps. Overwrite the draft at {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md"
- description: "Auditing draft → {name_prefix}-gap-analysis-draft-{serial}.md"

Wait for completion.

### Step 4 + Step 6: Parallel Fork (Target Conversion + Report Formatting)

Spawn BOTH agents SIMULTANEOUSLY in a single message with 2 Task calls:

**Agent 4 — Target to DOCX:**
- subagent_type: "target-to-docx"
- prompt: "[Target: {target_path}] [Output: {contract_dir}/{name_prefix}-target-{serial}.docx] [Skills: skills/docx] Convert the target contract PDF at {target_path} to docx using LibreOffice. Write to {contract_dir}/{name_prefix}-target-{serial}.docx. Validate the output."
- description: "Converting PDF to DOCX → {name_prefix}-target-{serial}.docx"

**Agent 6 — Report Formatter:**
- subagent_type: "report-formatter"
- prompt: "[Draft: {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md] [Output: {contract_dir}/{name_prefix}-gap-analysis-{serial}.docx] [Skills: skills/docx] [Firm: {firm_slug}] [Client: {client_slug}] [Type: {contract_type}] [Date: {date}] [Serial: {serial}] Read the audited gap analysis markdown. Use the /docx skill to create a formatted Word document. Times New Roman 11pt, headings 16/14/12pt bold, tables with borders. US Letter. Write to {contract_dir}/{name_prefix}-gap-analysis-{serial}.docx"
- description: "Formatting report → {name_prefix}-gap-analysis-{serial}.docx"

Wait for BOTH to complete.

### Step 5: Redline Editing

Spawn the `redline-editor` agent:
- subagent_type: "redline-editor"
- prompt: "[Target DOCX: {contract_dir}/{name_prefix}-target-{serial}.docx] [Gap Analysis: {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md] [Output: {contract_dir}/{name_prefix}-redlined-{serial}.docx] [Skills: skills/docx] [Serial: {serial}] Read the audited gap analysis at {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md. Unpack the target docx at {contract_dir}/{name_prefix}-target-{serial}.docx. Apply tracked changes for EVERY gap recommendation. Author: 'Andersen Consulting'. Pack and validate the redlined output to {contract_dir}/{name_prefix}-redlined-{serial}.docx"
- description: "Applying redlines → {name_prefix}-redlined-{serial}.docx"

Wait for completion.

### Per-Contract Summary

After all steps complete for this contract, print:
```
Contract: {client_slug} {contract_type}
  Classification: {contract_dir}/classification.md
  Gap Analysis (draft): {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md
  Gap Analysis (docx): {contract_dir}/{name_prefix}-gap-analysis-{serial}.docx
  Redlined Contract: {contract_dir}/{name_prefix}-redlined-{serial}.docx
```

Proceed to next contract.

## PHASE 2: BATCH COMPLETION

After all contracts are processed, print:

```
=== LEGAL GENIUS COMPLETE ===
Firm: {firm_slug}
Date: {date}
Contracts Analyzed: {count}

Output Summary:
{for each contract:}
  {client_slug} ({contract_type}):
    Gap Analysis: {path to gap-analysis docx}
    Redlined Contract: {path to redlined docx}

All files in: analysis/{firm_slug}/
=== END ===
```
