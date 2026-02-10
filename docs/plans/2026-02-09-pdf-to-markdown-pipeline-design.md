# PDF-to-Markdown Pipeline Redesign

## Problem

LibreOffice PDF-to-docx conversion produces mangled output — both broken text ordering and lost formatting. The redline-editor cannot reliably find or edit clause text in the resulting garbled XML. This makes the entire redline path unreliable.

## Solution

Replace the single LibreOffice conversion step with a two-agent pipeline that uses Claude's native PDF reading and docx-js for clean document generation.

### Current Flow (broken)

```
PDF → LibreOffice → mangled docx → redline-editor fails
```

### New Flow

```
PDF → pdf-to-markdown (Claude reads PDF) → structured markdown with clause anchors
                                          ↓
                                    target-to-docx (docx-js) → clean docx with bookmarks
                                          ↓
                                    redline-editor → reliable tracked changes
```

## New Agent: `pdf-to-markdown`

**File:** `agents/pdf-to-markdown.md`
**Model:** inherit
**Tools:** Read, Write

### Purpose

Read a target contract PDF using Claude's native PDF reading capability and produce a structured markdown file that preserves the document's logical structure.

### What It Preserves

- Heading hierarchy (detected by font size, bold, ALL CAPS, numbering patterns)
- Section/clause numbering (1.1, 1.2, (a), (b), etc.)
- Paragraph boundaries
- Bold and italic emphasis where detectable
- Tables (as markdown tables)
- Lists (bulleted and numbered)

### What It Does NOT Preserve

- Exact font faces and sizes (the docx step applies professional formatting)
- Column layouts, headers/footers, watermarks
- Images, logos, signature blocks (noted as `[Signature Block]` placeholder)

### Clause Anchors

Each numbered clause gets a unique HTML comment anchor:

```markdown
<!-- clause:1.1 -->
**1.1** "Agreement" means this Master Services Agreement...

<!-- clause:2.3.a -->
**(a)** The Consultant shall provide...
```

Anchor format: `<!-- clause:{section_number} -->` where section_number uses dots and letters matching the contract's numbering scheme.

### Output Format

```markdown
# CONTRACT TITLE

**Parties:** [Party A] and [Party B]
**Effective Date:** [date]

---

<!-- clause:1 -->
## 1. DEFINITIONS

<!-- clause:1.1 -->
**1.1** "Agreement" means this Master Services Agreement...

<!-- clause:1.2 -->
**1.2** "Confidential Information" means...

<!-- clause:2 -->
## 2. SCOPE OF SERVICES

<!-- clause:2.1 -->
**2.1** Consultant shall provide the Services as described...

| Deliverable | Timeline | Fee |
|------------|----------|-----|
| Phase 1    | 30 days  | $X  |
```

## Rebuilt Agent: `target-to-docx`

**File:** `agents/target-to-docx.md` (rewritten)
**Model:** inherit
**Tools:** Read, Write, Bash, Skill

### Purpose

Convert structured markdown (with clause anchors) to a clean, professionally formatted Word document using docx-js.

### Key Changes from Current Version

- Drops LibreOffice dependency entirely
- Generates a Node.js script using docx-js (same approach as report-formatter)
- Produces clean, predictable OOXML with well-formed runs and paragraphs
- Converts `<!-- clause:X -->` anchors to `<w:bookmarkStart>` / `<w:bookmarkEnd>` pairs

### Formatting Spec

- Font: Times New Roman 11pt body
- Heading 1: Times New Roman 16pt bold
- Heading 2: Times New Roman 14pt bold
- Heading 3: Times New Roman 12pt bold
- Tables: borders, header row shading (D5E8F0), cell margins
- Page: US Letter, 1-inch margins
- Numbered clauses preserved as-is from markdown

### Bookmark Embedding

Each clause anchor becomes a Word bookmark:

```xml
<w:bookmarkStart w:id="100" w:name="clause_1_1"/>
<w:p>
  <w:r><w:rPr><w:b/></w:rPr><w:t>1.1</w:t></w:r>
  <w:r><w:t xml:space="preserve"> "Agreement" means...</w:t></w:r>
</w:p>
<w:bookmarkEnd w:id="100"/>
```

This gives the redline-editor a reliable way to locate clauses by bookmark name.

## Updated Agent: `redline-editor`

**File:** `agents/redline-editor.md` (minor update)

### Changes

Add bookmark-based clause location as the primary search strategy:

1. **Primary:** Search for `<w:bookmarkStart w:name="clause_{normalized}"/>` to locate the target paragraph
2. **Fallback:** Text search in surrounding `<w:r>` elements (existing behavior)
3. Everything else stays the same — tracked change markup, author, dates, XML rules

## Updated Orchestrator: `commands/legal-genius.md`

### Pipeline Change

Replace:
```
Step 4 (parallel with 6): target-to-docx (PDF → docx via LibreOffice)
```

With:
```
Step 4a (sequential): pdf-to-markdown (PDF → structured markdown)
Step 4b (parallel with 6): target-to-docx (markdown → clean docx)
```

### New Step 4a: PDF to Markdown

Runs after Step 3 (audit review), before the parallel fork.

- subagent_type: "pdf-to-markdown"
- Input: target contract PDF path
- Output: `{contract_dir}/drafts/{name_prefix}-target-{serial}.md`

### Modified Step 4b: Target to DOCX

Now takes markdown input instead of PDF input.

- subagent_type: "target-to-docx"
- Input: `{contract_dir}/drafts/{name_prefix}-target-{serial}.md`
- Output: `{contract_dir}/{name_prefix}-target-{serial}.docx`
- Runs in parallel with Step 6 (report-formatter)

### Modified Step 5: Redline Editing

Input docx path unchanged. The docx is now cleaner and has bookmarks.

## Dependency Changes

| Dependency | Before | After |
|-----------|--------|-------|
| LibreOffice | Required (not installed) | **Removed** |
| docx-js (npm `docx`) | Used by report-formatter | Also used by target-to-docx |
| Claude PDF reading | Used by classifier, gap-analyzer, auditor | Also used by pdf-to-markdown |

**Net effect:** One dependency removed, zero added.

## Files to Change

| File | Action |
|------|--------|
| `agents/pdf-to-markdown.md` | CREATE |
| `agents/target-to-docx.md` | REWRITE |
| `agents/redline-editor.md` | UPDATE (add bookmark search) |
| `commands/legal-genius.md` | UPDATE (insert Step 4a, adjust parallelism) |
| `CLAUDE.md` | UPDATE (document new agent, remove LibreOffice dep) |
| `.claude-plugin/plugin.json` | UPDATE (add new agent) |

## Files NOT Changed

- `agents/contract-classifier.md` — untouched
- `agents/gap-analyzer.md` — untouched
- `agents/audit-reviewer.md` — untouched
- `agents/report-formatter.md` — untouched
- `skills/docx/` — untouched
- File naming convention — unchanged
