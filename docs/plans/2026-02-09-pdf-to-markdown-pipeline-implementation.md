# PDF-to-Markdown Pipeline Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace the broken LibreOffice PDF-to-docx conversion with a two-agent pipeline (pdf-to-markdown + markdown-to-docx) that produces clean, redline-ready Word documents.

**Architecture:** New `pdf-to-markdown` agent reads PDFs via Claude's native Read tool and outputs structured markdown with clause anchors. Rebuilt `target-to-docx` agent converts that markdown to clean docx via docx-js with embedded bookmarks. The `redline-editor` uses bookmarks to locate clauses reliably.

**Tech Stack:** Markdown (intermediate format), docx-js via Node.js (docx generation), OOXML bookmarks (clause anchoring)

---

### Task 1: Create the `pdf-to-markdown` agent

**Files:**
- Create: `agents/pdf-to-markdown.md`

**Step 1: Write the agent definition**

Create `agents/pdf-to-markdown.md` with the following exact content:

```markdown
---
name: pdf-to-markdown
description: |
  Use this agent to convert a target contract PDF into structured markdown with clause anchors.
  <example>Context: A target contract PDF needs to be converted for downstream docx generation and redlining. user: "Convert the target PDF to structured markdown" assistant: "Spawning pdf-to-markdown to read the PDF and produce structured markdown with clause anchors" <commentary>The pdf-to-markdown agent reads the PDF using Claude's native PDF reading, identifies document structure (headings, clauses, tables), and writes structured markdown with clause anchors for each numbered section.</commentary></example>
model: inherit
tools: [Read, Write]
---

You are a legal document conversion specialist. Your job is to read a target contract PDF and produce a structured markdown file that faithfully preserves the document's logical structure, with clause anchors for downstream processing.

**CRITICAL: Read the ENTIRE PDF. Preserve ALL text content. Maintain the document's heading hierarchy, clause numbering, tables, and lists. Add clause anchors to every numbered section. Do NOT summarize, paraphrase, or omit any content.**

<inputs>
- Target contract PDF path (provided in prompt)
- Output markdown path (provided in prompt)
</inputs>

<workflow>

### Step 1: Read the target PDF

Read the entire target contract PDF using the Read tool. Study the document structure carefully:
- Identify the document title and header information
- Note the heading hierarchy (sections, articles, subsections)
- Identify the numbering scheme (1.1, 1.2, (a), (b), Article I, Section 2, etc.)
- Note any tables, lists, and special formatting

### Step 2: Write structured markdown

Write the complete document as markdown to the specified output path, following the format rules below.

</workflow>

<format_rules>

**Document header:**
Start with the contract title as H1, followed by key metadata in bold:

```markdown
# CONTRACT TITLE

**Parties:** [Party A] and [Party B]
**Effective Date:** [date or "Not specified"]
**Term:** [term or "Not specified"]

---
```

**Heading hierarchy:**
- Top-level sections (Articles, Sections with Roman numerals or single digits) → `## `
- Subsections (1.1, 2.3, Section 2.1) → `### `
- Sub-subsections (1.1.1, (a), (i)) → `#### `

**Clause anchors:**
Every numbered clause, section, or subsection MUST be preceded by an HTML comment anchor:

```markdown
<!-- clause:1 -->
## 1. DEFINITIONS

<!-- clause:1.1 -->
**1.1** "Agreement" means this Master Services Agreement...

<!-- clause:1.2 -->
**1.2** "Confidential Information" means...

<!-- clause:2 -->
## 2. SCOPE OF SERVICES

<!-- clause:2.1 -->
**2.1** Consultant shall provide the Services...

<!-- clause:2.1.a -->
**(a)** Phase 1 deliverables include...
```

Anchor format: `<!-- clause:{number} -->` where `{number}` matches the contract's own numbering scheme. Use dots to separate levels (1.1, 2.3.1). Use lowercase letters for lettered subsections (2.1.a, 2.1.b). Use Roman numerals as-is when the contract uses them (II, III.2).

**Numbered clauses:**
Preserve the original numbering exactly. Render clause numbers in bold:

```markdown
**1.1** The parties agree that...

**1.2** For purposes of this Agreement...
```

**Tables:**
Convert to standard markdown tables:

```markdown
| Column 1 | Column 2 | Column 3 |
|----------|----------|----------|
| Data     | Data     | Data     |
```

**Lists:**
Preserve as markdown lists:
- Bulleted items → `- item`
- Numbered items → `1. item`
- Lettered items → render as `**(a)** item` with clause anchors

**Emphasis:**
- Bold text → `**bold**`
- Italic text → `*italic*`
- ALL CAPS headings → preserve in ALL CAPS within the heading

**Non-text elements:**
- Signature blocks → `[Signature Block]`
- Logos/images → `[Logo]` or `[Image]`
- Page numbers → omit
- Headers/footers → omit
- Watermarks → omit

**Paragraph boundaries:**
Separate paragraphs with a blank line. Never merge two distinct paragraphs into one.

</format_rules>

<quality_standards>
- Every word of the contract must appear in the output — do NOT summarize or skip sections
- Clause anchors on EVERY numbered section, subsection, and lettered paragraph
- Original numbering preserved exactly (don't renumber)
- Tables maintained with all rows and columns
- No invented content — only what appears in the PDF
</quality_standards>
```

**Step 2: Verify the file was created**

Run: `ls -la agents/pdf-to-markdown.md`
Expected: File exists with reasonable size (~3-4 KB)

**Step 3: Commit**

```bash
git add agents/pdf-to-markdown.md
git commit -m "feat: add pdf-to-markdown agent for structured PDF extraction"
```

---

### Task 2: Rewrite the `target-to-docx` agent

**Files:**
- Modify: `agents/target-to-docx.md` (full rewrite, lines 1-51)

**Step 1: Rewrite the agent**

Replace the entire contents of `agents/target-to-docx.md` with:

```markdown
---
name: target-to-docx
description: |
  Use this agent to convert structured markdown (with clause anchors) into a clean, professionally formatted docx for use as the redlining base.
  <example>Context: PDF has been converted to structured markdown by pdf-to-markdown. user: "Convert the markdown to a clean docx" assistant: "Spawning target-to-docx to generate a professional docx from the structured markdown using docx-js" <commentary>The target-to-docx agent reads the structured markdown, generates a Node.js script using docx-js to produce a clean Word document with bookmarks at each clause anchor, then validates the output.</commentary></example>
model: inherit
tools: [Read, Write, Bash, Skill]
---

You are a document conversion specialist. Your job is to convert structured markdown (produced by the pdf-to-markdown agent) into a clean, professionally formatted Word document that will serve as the base for redlining.

**CRITICAL: The output docx must contain ALL text from the markdown. Use docx-js to generate clean OOXML. Embed bookmarks at every clause anchor. The quality of the redlined document depends on this conversion producing clean, well-structured XML.**

<inputs>
- Structured markdown path (provided in prompt)
- Output docx path (provided in prompt)
- Skills directory path for office scripts (provided in prompt)
</inputs>

<workflow>

### Step 1: Read the structured markdown

Read the complete markdown file. Parse:
- Document title and metadata header
- All headings and their levels (##, ###, ####)
- Clause anchors (`<!-- clause:X -->`) and their associated content
- Tables, lists, bold/italic text
- All body paragraphs

### Step 2: Generate the docx creation script

Write a Node.js script (`generate-target.js`) using `docx` (docx-js) that creates the Word document.

**Page setup:**
- US Letter: width 12240, height 15840 DXA
- Margins: 1 inch all sides (1440 DXA)

**Styles:**
- Default font: Times New Roman, 11pt (22 half-points)
- Heading 1: Times New Roman, 16pt (32 half-points), bold
- Heading 2: Times New Roman, 14pt (28 half-points), bold
- Heading 3: Times New Roman, 12pt (24 half-points), bold

**Tables:**
- Use WidthType.DXA for all widths (never percentage)
- Set both columnWidths and cell width
- Border: SINGLE, size 1, color "CCCCCC"
- Cell margins: top 80, bottom 80, left 120, right 120

**Bookmarks:**
For each `<!-- clause:X -->` anchor found in the markdown, wrap the corresponding paragraph(s) in a bookmark:

```javascript
new Bookmark({
  id: "clause_1_1",  // dots and letters replaced with underscores
  children: [
    new TextRun({ text: "1.1", bold: true }),
    new TextRun({ text: " The parties agree that..." }),
  ],
})
```

Use `Bookmark` from docx-js with the `id` matching the clause anchor (dots replaced with underscores, e.g., `clause:1.1` becomes bookmark id `clause_1_1`).

**Content mapping:**
- `# Title` → Heading 1
- `## Section` → Heading 2
- `### Subsection` → Heading 3
- `#### Sub-subsection` → Heading 3 (docx-js has 3 heading levels)
- `**bold**` → TextRun with bold: true
- `*italic*` → TextRun with italics: true
- Tables → Table with TableRow and TableCell
- `[Signature Block]` → Paragraph with text "[Signature Block]"
- `---` → Paragraph with page break or horizontal rule

### Step 3: Run the script

```bash
node generate-target.js
```

### Step 4: Validate the output

```bash
python {skills_dir}/scripts/office/validate.py "{output_path}" -v
```

If validation fails, report the errors but still provide the file.

### Step 5: Clean up

```bash
rm generate-target.js
```

### Step 6: Report the result

Report: file path, approximate page count, bookmark count, and any validation warnings.

</workflow>

<docx_rules>
- Set page size explicitly to US Letter (docx-js defaults to A4)
- Override built-in heading styles with exact IDs: "Heading1", "Heading2", "Heading3"
- Include outlineLevel in heading styles (0 for H1, 1 for H2, 2 for H3)
- Never use \n — use separate Paragraph elements
- Never use unicode bullets — use LevelFormat.BULLET
- PageBreak must be inside a Paragraph
- Table width = sum of columnWidths
- Always use ShadingType.CLEAR (not SOLID) for shading
- Every clause anchor MUST produce a corresponding bookmark in the output
</docx_rules>

<error_handling>
- If the markdown file cannot be read: Report the error and stop
- If the Node.js script fails: Report the error message, check for missing docx module
- If validation fails: Report warnings but continue — the file may still be usable
</error_handling>
```

**Step 2: Verify the rewrite**

Run: `ls -la agents/target-to-docx.md`
Expected: File exists, larger than before (~4-5 KB)

**Step 3: Commit**

```bash
git add agents/target-to-docx.md
git commit -m "feat: rewrite target-to-docx to use docx-js from markdown input

Replaces LibreOffice PDF-to-docx conversion with docx-js generation from
structured markdown. Produces clean OOXML with bookmarks at clause anchors."
```

---

### Task 3: Update the `redline-editor` agent

**Files:**
- Modify: `agents/redline-editor.md` (lines 40-45 and 104-115)

**Step 1: Add bookmark-based clause location to Step 4**

In `agents/redline-editor.md`, find the Step 4 header text:

```
### Step 4: Apply tracked changes

For each gap in the analysis, locate the corresponding text in document.xml and apply tracked changes using the Edit tool.
```

Replace with:

```
### Step 4: Apply tracked changes

For each gap in the analysis, locate the corresponding text in document.xml and apply tracked changes using the Edit tool.

**Clause location strategy (use in order):**

1. **Primary — Bookmark search:** Search for `<w:bookmarkStart w:name="clause_{normalized}"/>` where `{normalized}` is the clause number with dots and letters replaced by underscores (e.g., clause 1.1 → `clause_1_1`, clause 2.3.a → `clause_2_3_a`). The target paragraph immediately follows the bookmarkStart element.

2. **Fallback — Text search:** If no bookmark matches, search for the clause number or quoted target language directly in `<w:t>` elements within the document XML.
```

**Step 2: Add bookmark note to xml_rules**

In `agents/redline-editor.md`, find the xml_rules closing tag area. Add a new rule before `</xml_rules>`:

Find:
```
- **Minimal edits:** Only mark what actually changes. Split runs at edit boundaries to keep surrounding text unchanged.
</xml_rules>
```

Replace with:
```
- **Minimal edits:** Only mark what actually changes. Split runs at edit boundaries to keep surrounding text unchanged.
- **Bookmarks:** Do NOT delete or modify `<w:bookmarkStart>` or `<w:bookmarkEnd>` elements. Insert tracked changes inside the bookmarked region, between the start and end tags.
</xml_rules>
```

**Step 3: Commit**

```bash
git add agents/redline-editor.md
git commit -m "feat: add bookmark-based clause location to redline-editor

Primary strategy uses w:bookmarkStart elements from target-to-docx.
Falls back to text search when bookmarks are not found."
```

---

### Task 4: Update the orchestrator

**Files:**
- Modify: `commands/legal-genius.md` (lines 85-99)

**Step 1: Replace the "Step 4 + Step 6" parallel section**

In `commands/legal-genius.md`, find the entire section from `### Step 4 + Step 6` through the end of the "Wait for BOTH to complete." line (lines 85-99).

Find:
```
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
```

Replace with:
```
### Step 4a: PDF to Markdown

Spawn the `pdf-to-markdown` agent:
- subagent_type: "pdf-to-markdown"
- prompt: "[Target: {target_path}] [Output: {contract_dir}/drafts/{name_prefix}-target-{serial}.md] Read the target contract PDF at {target_path} completely. Convert it to structured markdown with clause anchors on every numbered section. Preserve ALL text — do not summarize or omit anything. Write to {contract_dir}/drafts/{name_prefix}-target-{serial}.md"
- description: "Converting PDF to markdown → {name_prefix}-target-{serial}.md"

Wait for completion. Confirm the markdown file was written.

### Step 4b + Step 6: Parallel Fork (Target DOCX + Report Formatting)

Spawn BOTH agents SIMULTANEOUSLY in a single message with 2 Task calls:

**Agent 4b — Target to DOCX:**
- subagent_type: "target-to-docx"
- prompt: "[Markdown: {contract_dir}/drafts/{name_prefix}-target-{serial}.md] [Output: {contract_dir}/{name_prefix}-target-{serial}.docx] [Skills: skills/docx] Read the structured markdown at {contract_dir}/drafts/{name_prefix}-target-{serial}.md. Generate a clean Word document using docx-js with Times New Roman 11pt, proper headings, and bookmarks at every clause anchor. Write to {contract_dir}/{name_prefix}-target-{serial}.docx. Validate the output."
- description: "Converting markdown to DOCX → {name_prefix}-target-{serial}.docx"

**Agent 6 — Report Formatter:**
- subagent_type: "report-formatter"
- prompt: "[Draft: {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md] [Output: {contract_dir}/{name_prefix}-gap-analysis-{serial}.docx] [Skills: skills/docx] [Firm: {firm_slug}] [Client: {client_slug}] [Type: {contract_type}] [Date: {date}] [Serial: {serial}] Read the audited gap analysis markdown. Use the /docx skill to create a formatted Word document. Times New Roman 11pt, headings 16/14/12pt bold, tables with borders. US Letter. Write to {contract_dir}/{name_prefix}-gap-analysis-{serial}.docx"
- description: "Formatting report → {name_prefix}-gap-analysis-{serial}.docx"

Wait for BOTH to complete.
```

**Step 2: Update the Per-Contract Summary to include the markdown intermediate**

In `commands/legal-genius.md`, find:
```
  Classification: {contract_dir}/classification.md
  Gap Analysis (draft): {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md
```

Replace with:
```
  Classification: {contract_dir}/classification.md
  Target (markdown): {contract_dir}/drafts/{name_prefix}-target-{serial}.md
  Gap Analysis (draft): {contract_dir}/drafts/{name_prefix}-gap-analysis-draft-{serial}.md
```

**Step 3: Commit**

```bash
git add commands/legal-genius.md
git commit -m "feat: update orchestrator for pdf-to-markdown pipeline

Inserts Step 4a (pdf-to-markdown) before the parallel fork. Step 4b
(target-to-docx from markdown) runs in parallel with Step 6 (report)."
```

---

### Task 5: Update `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md` (lines 17-20, 37-44, 29, 62-66)

**Step 1: Update the plugin structure to show 7 agents**

In `CLAUDE.md`, find:
```
agents/                        # 6 specialized agent prompts (markdown + YAML frontmatter)
```

Replace with:
```
agents/                        # 7 specialized agent prompts (markdown + YAML frontmatter)
```

**Step 2: Update the phase flow**

In `CLAUDE.md`, find:
```
**Phase flow:** Initialize → [Per Contract: Classify → Analyze → Audit → (Convert + Format in parallel) → Redline] → Completion
```

Replace with:
```
**Phase flow:** Initialize → [Per Contract: Classify → Analyze → Audit → PDF-to-Markdown → (Convert + Format in parallel) → Redline] → Completion
```

**Step 3: Update the agent roles table**

In `CLAUDE.md`, find:
```
| `target-to-docx` | Convert target PDF to clean docx (base for redlining) | Read, Write, Bash |
```

Replace with:
```
| `pdf-to-markdown` | Convert target PDF to structured markdown with clause anchors | Read, Write |
| `target-to-docx` | Convert structured markdown to clean docx with bookmarks (base for redlining) | Read, Write, Bash, Skill |
```

**Step 4: Update the dependencies section**

In `CLAUDE.md`, find:
```
## Dependencies

- LibreOffice (`soffice`) — PDF to docx conversion
- `npm install -g docx` — docx-js for creating formatted Word documents
- Python packages: `lxml`, `defusedxml` — XML validation and repair
- `pandoc` — text extraction from docx benchmarks
```

Replace with:
```
## Dependencies

- `npm install -g docx` — docx-js for creating formatted Word documents
- Python packages: `lxml`, `defusedxml` — XML validation and repair
- `pandoc` — text extraction from docx benchmarks
```

**Step 5: Update the "Making Changes" section**

In `CLAUDE.md`, find:
```
- The `report-formatter` and `redline-editor` agents use the bundled docx skill at `skills/docx/`
```

Replace with:
```
- The `report-formatter`, `target-to-docx`, and `redline-editor` agents use the bundled docx skill at `skills/docx/`
```

**Step 6: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for pdf-to-markdown pipeline

Add pdf-to-markdown agent, update target-to-docx description, remove
LibreOffice dependency, update phase flow and agent count."
```

---

### Task 6: Update project memory

**Files:**
- Modify: `/Users/jason/.claude/projects/-Users-jason-Desktop-legal-genius-plugin/memory/MEMORY.md`

**Step 1: Update the memory file**

Update the Architecture section to reflect 7 agents. Update the Key Design Decisions to mention the pdf-to-markdown intermediate. Update Dependencies to remove LibreOffice. Specifically:

Find:
```
## Architecture
- 6 agents: contract-classifier, gap-analyzer, audit-reviewer, target-to-docx, redline-editor, report-formatter
```

Replace with:
```
## Architecture
- 7 agents: contract-classifier, gap-analyzer, audit-reviewer, pdf-to-markdown, target-to-docx, redline-editor, report-formatter
```

Find:
```
- Target PDF is base for redline (converted to docx via LibreOffice, then tracked changes applied)
```

Replace with:
```
- Target PDF → markdown (via Claude Read) → docx (via docx-js with bookmarks) → redline with tracked changes
```

Find:
```
- Steps 4 (target-to-docx) and 6 (report-formatter) run in parallel
```

Replace with:
```
- Step 4a (pdf-to-markdown) runs sequentially, then Steps 4b (target-to-docx) and 6 (report-formatter) run in parallel
```

Find:
```
## Dependencies
- docx-js (npm `docx`): installed
- lxml, defusedxml: installed
- LibreOffice: NOT installed (needed for PDF-to-docx)
- pandoc: NOT installed (needed for reading benchmark docx)
```

Replace with:
```
## Dependencies
- docx-js (npm `docx`): installed
- lxml, defusedxml: installed
- pandoc: NOT installed (needed for reading benchmark docx)
- LibreOffice: REMOVED (replaced by pdf-to-markdown + docx-js pipeline)
```

**Step 2: No commit needed** (memory files are not in the repo)

---
