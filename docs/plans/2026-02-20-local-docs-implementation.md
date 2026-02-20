# Local File Discovery Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace Google Workspace MCP with local file discovery, adding a processing cache so documents aren't re-extracted on subsequent runs.

**Architecture:** A new `local-docs-discovery.md` haiku agent scans `external-docs/<subfolder>/` for `.docx`, `.pdf`, `.md`, `.txt` files, extracts content via shell tools (`pandoc`, `pdftotext`) or the Read tool, and writes `raw/local-docs.json`. A `.cache.json` file in the subfolder tracks processed documents by filename + mtime. The synthesis model replaces all `drive_sources` references with `local_doc_sources`.

**Tech Stack:** Markdown agent definitions, shell tools (pandoc, pdftotext), JSON file schemas.

---

### Task 1: Delete Google Drive files and remove MCP config

**Files:**
- Delete: `agents/google-drive-discovery.md`
- Delete: `skills/project-insights/references/google-drive-mcp-reference.md`
- Modify: `.mcp.json`

**Step 1: Delete the Google Drive discovery agent**

```bash
rm agents/google-drive-discovery.md
```

**Step 2: Delete the Google Drive MCP reference**

```bash
rm skills/project-insights/references/google-drive-mcp-reference.md
```

**Step 3: Remove google-workspace from .mcp.json**

Edit `.mcp.json` to remove the `google-workspace` entry. The file should become:

```json
{
  "mcpServers": {
    "atlassian": {
      "type": "http",
      "url": "https://mcp.atlassian.com/v1/mcp"
    }
  }
}
```

**Step 4: Commit**

```bash
git add -u agents/google-drive-discovery.md skills/project-insights/references/google-drive-mcp-reference.md .mcp.json
git commit -m "chore: remove Google Drive agent, MCP reference, and MCP config"
```

---

### Task 2: Create external-docs directory and .gitignore

**Files:**
- Create: `external-docs/.gitkeep`
- Create: `.gitignore`

**Step 1: Create the external-docs directory with .gitkeep**

```bash
mkdir -p external-docs
touch external-docs/.gitkeep
```

**Step 2: Create .gitignore**

Write `.gitignore` at the project root:

```
# Generated reports
reports/

# External documents for analysis (user-provided, not source code)
external-docs/*
!external-docs/.gitkeep

# Processing cache files
.cache.json
```

**Step 3: Commit**

```bash
git add .gitignore external-docs/.gitkeep
git commit -m "chore: add external-docs directory and .gitignore"
```

---

### Task 3: Create the local-docs-discovery agent

**Files:**
- Create: `agents/local-docs-discovery.md`

**Step 1: Write the agent definition**

Write `agents/local-docs-discovery.md` with the following content. This agent mirrors the structure of the existing `jira-discovery.md` and `confluence-discovery.md` agents — it receives inputs, executes extraction steps, and writes output to `raw/local-docs.json`.

The agent must:

1. Start with YAML frontmatter: `model: haiku`, `description: Discover local planning documents for project insights reporting`
2. Accept inputs: `local_docs_dir` (absolute path), `run_dir`, `cache_path`, `reprocess` (boolean), `search_keywords` (optional)
3. **Prerequisite check:** Verify `pandoc` and `pdftotext` are available via `which pandoc` and `which pdftotext`. If either is missing, report a clear error with install instructions (`brew install pandoc`, `brew install poppler`) and stop.
4. List files recursively in `local_docs_dir` matching extensions: `.docx`, `.pdf`, `.md`, `.txt` (use `find` via Bash)
5. Read `.cache.json` from `cache_path` if it exists (via Read tool)
6. For each file, get mtime via `stat -f '%m' <file>` (macOS) or `stat -c '%Y' <file>` (Linux). Compare against cache entry. If mtime matches and `reprocess` is false, use cached `extracted` data.
7. For new/changed files, extract text:
   - `.docx`: `pandoc -t plain <file>` via Bash
   - `.pdf`: `pdftotext <file> -` via Bash
   - `.md`, `.txt`: Read tool
8. From extracted text, identify:
   - `content_summary`: First 500 chars of scope-relevant content
   - `dates_mentioned`: Use same date extraction patterns as confluence-discovery.md (ISO `\d{4}-\d{2}-\d{2}`, US formats, relative Q1/Q2 patterns). Record date, context, format_detected, confidence.
   - `deliverables_mentioned`: Key noun phrases from headings and bullet points (2-8 words each)
   - `jira_keys_mentioned`: Regex `[A-Z][A-Z0-9]+-\d+` for all unique Jira keys
9. Write updated `.cache.json` using the Write tool. Schema:
   ```json
   {
     "version": 1,
     "processed": {
       "<filename>": {
         "mtime": "<unix timestamp as string>",
         "size_bytes": <int>,
         "extracted": {
           "content_summary": "...",
           "dates_mentioned": [],
           "deliverables_mentioned": [],
           "jira_keys_mentioned": []
         }
       }
     }
   }
   ```
10. Write `raw/local-docs.json` using the Write tool. Schema:
    ```json
    {
      "source_dir": "external-docs/<subfolder>",
      "fetched_at": "<ISO 8601 timestamp>",
      "total_docs": <count>,
      "docs": [
        {
          "file_path": "<relative path from local_docs_dir>",
          "file_name": "<basename>",
          "file_type": "pdf|docx|md|txt",
          "last_modified": "<ISO 8601 from mtime>",
          "from_cache": true|false,
          "content_summary": "[max 500 chars]",
          "dates_mentioned": [...],
          "deliverables_mentioned": [...],
          "jira_keys_mentioned": [...]
        }
      ]
    }
    ```
11. Return only a brief status message. Example: `"Found 3 local docs (1 PDF, 1 DOCX, 1 MD). 1 from cache, 2 newly processed. 4 unique Jira keys referenced."`
12. Error handling: If pandoc/pdftotext fails on a specific file, log the error and skip that file. Always write both output files even if some extractions failed.

**Step 2: Commit**

```bash
git add agents/local-docs-discovery.md
git commit -m "feat: add local-docs-discovery agent with cache support"
```

---

### Task 4: Update the command definition

**Files:**
- Modify: `commands/project-insights.md`

**Step 1: Update the command arguments**

Edit `commands/project-insights.md` to:

1. Change the frontmatter `argument-hint` line from:
   ```
   argument-hint: --jira PROJECT_KEY --spaces SPACE_KEY [--drive-email EMAIL] [--drive-folders IDS] [--search "keywords"] [--target-date YYYY-MM-DD] [--stale-days N]
   ```
   to:
   ```
   argument-hint: --jira PROJECT_KEY --spaces SPACE_KEY [--local-docs SUBFOLDER] [--reprocess-docs] [--search "keywords"] [--target-date YYYY-MM-DD] [--stale-days N]
   ```

2. Replace the `--drive-email` and `--drive-folders` argument descriptions with:
   ```
   - `--local-docs` (optional): Subfolder name under `external-docs/`. Enables local document discovery. Files (.docx, .pdf, .md, .txt) in this directory are scanned for planning content.
   - `--reprocess-docs` (optional): Force re-extraction of all local docs, ignoring the processing cache.
   ```

3. Remove the `--drive-folders` without `--drive-email` validation paragraph.

4. Remove the entire "### 2. Google Workspace MCP" section from the MCP health check. Keep only "### 1. Atlassian MCP" and renumber "### 3. All checks passed" to "### 2. All checks passed". Update the "All checks passed" text to: `Only after the Atlassian MCP health check passes, proceed to follow the project-insights skill's 4-pass process exactly.`

5. Update the first line description from `Generate executive-level project insights from Jira, Confluence, and Google Drive` to `Generate executive-level project insights from Jira, Confluence, and local documents`.

**Step 2: Commit**

```bash
git add commands/project-insights.md
git commit -m "feat: update command to use --local-docs instead of --drive-email"
```

---

### Task 5: Update the orchestration skill (SKILL.md)

**Files:**
- Modify: `skills/project-insights/SKILL.md`

This is the largest single change. Work through the file top-to-bottom:

**Step 1: Update the header and description**

- Line 1-2: Change `description` in frontmatter from mentioning "Google Drive" to "local documents"
- Line 9-10: Change opening paragraph from "Jira, Confluence, and Google Drive" to "Jira, Confluence, and local documents"

**Step 2: Update the architecture table**

- Line 19 (Pass 1 description): Change `Discovery (Jira + Confluence + Drive)` to `Discovery (Jira + Confluence + Local Docs)`

**Step 3: Remove Google Drive MCP health check**

Remove the entire "### Google Drive Auth" section (lines 41-49, the section starting with `### Google Drive Auth (only if --drive-email provided)`).

Update "### Both Checks Must Pass" (line 53) to: `Only after the Atlassian check passes, proceed to Run Directory Setup and Pass 1.`

**Step 4: Update run directory structure**

- Line 71: Change `google-drive-docs.json    (only if --drive-email provided)` to `local-docs.json          (only if --local-docs provided)`
- Lines 92-99 (run-metadata.json parameters): Replace `drive_email` and `drive_folder_ids` with:
  ```json
  "local_docs_subfolder": "<subfolder name or null>",
  "reprocess_docs": false,
  ```
- Lines 109-119 (mcp_auth section): Remove the entire `google_drive` block. The `mcp_auth` section should only contain `atlassian`.

**Step 5: Update Pass 1 (Discovery)**

- Line 155 (Google Drive agent bullet): Replace the entire Google Drive agent dispatch instruction with:
  ```
  - **Local docs agent** (`subagent_type: general-purpose`, `model: haiku`): Pass the absolute path to `external-docs/<subfolder>/`, the cache path `external-docs/<subfolder>/.cache.json`, the reprocess flag, search keywords, and run directory path. The agent will scan local files, extract content via pandoc/pdftotext/Read, and write to `raw/local-docs.json`. **Only dispatch if `--local-docs` was provided.**
  ```
- Line 164: Change `raw/google-drive-docs.json` verification to `raw/local-docs.json` with condition `(only if local docs agent was dispatched)`

**Step 6: Update Pass 1b (Synthesis)**

- Line 181: Change `raw/google-drive-docs.json (if it exists)` to `raw/local-docs.json (if it exists)`
- Line 184: Replace "Google Drive docs" references with "local docs" in the cross-reference extraction description. Specifically change: `Scan Drive docs for Jira key mentions and Confluence URL references` to `Scan local docs for Jira key mentions and Confluence URL references`. Change `If Drive data is present, also build Drive↔Jira and Drive↔Confluence cross-reference maps` to `If local docs data is present, also build LocalDoc↔Jira and LocalDoc↔Confluence cross-reference maps`.
- Line 185: Change `drive-anchored` to `local-doc-anchored` in clustering description

**Step 7: Update Checkpoint display**

- Lines 204-231: Replace all "Drive docs" references with "Local docs". Change `P Drive docs` to `P local docs`. Change `Drive:` to `Local docs:` in the unmatched items display.

**Step 8: Update Pass 2 (Refined Discovery)**

- Lines 251-256: Replace Google Drive references. Change `Search Google Drive for matching docs by keyword (if Drive is enabled)` to `Search local docs for matching files by keyword (if local docs enabled)`.

**Step 9: Update Pass 4 (Report Generation)**

- Line 325: Change `P Drive docs` to `P Local docs` in source count description.

**Step 10: Update Reference Documents section**

- Line 402: Remove the Google Drive MCP reference line: `- **Google Drive MCP tool names and parameters:** skills/project-insights/references/google-drive-mcp-reference.md`

**Step 11: Commit**

```bash
git add skills/project-insights/SKILL.md
git commit -m "feat: update orchestration to use local-docs agent instead of Google Drive"
```

---

### Task 6: Update the synthesis model

**Files:**
- Modify: `skills/project-insights/references/synthesis-model.md`

This file has extensive Drive references throughout. Work through each section:

**Step 1: Update Section 1 (Purpose and Context)**

- Line 1-9: Replace "Google Drive" with "local documents" in the opening paragraph. Change "Jira, Confluence, and Google Drive data" to "Jira, Confluence, and local document data". Change the note about `raw/google-drive-docs.json` to reference `raw/local-docs.json`.

**Step 2: Update Phase 1 (Cross-Reference Extraction)**

- Line 17: Change "Google Drive docs" to "local docs" in the bidirectional map description.
- Steps 1d and 1e: Replace "Drive" with "LocalDoc" throughout:
  - Step 1d title: `LocalDoc → Jira references (if local docs data exists)`
  - Step 1d body: `Read raw/local-docs.json`. Change `drive_file_id` to `file_path`. Change map key from `{drive_file_id: [jira_keys]}` to `{file_path: [jira_keys]}`.
  - Step 1e title: `LocalDoc → Confluence references (if local docs data exists)`
  - Step 1e body: Change `drive_file_id` to `file_path`. Change `confluence_refs_mentioned` to matching Confluence URLs found in extracted text content.
- Step 1f: Replace "Drive" with "LocalDoc" in cross-reference classification. Change `Jira↔Drive` to `Jira↔LocalDoc`, `Confluence↔Drive` to `Confluence↔LocalDoc`.

**Step 3: Update Phase 2 (Initial Clustering)**

- Section 2a: Change "Drive docs" to "local docs" in the attach steps. Replace `from Phase 1d` references.
- Section 2c title: Change to `LocalDoc-anchored clusters (if local docs data exists)`. Replace all "Drive" with "LocalDoc" in the body. Change `Drive→Confluence cross-references (Phase 1e)` to `LocalDoc→Confluence cross-references (Phase 1e)`.
- Section 2d: Replace "Drive doc titles" with "local doc titles" in keyword overlap.
- Section 2e: Change `unmatched_drive` to `unmatched_local_docs`.

**Step 4: Update Phase 3 (Confidence Scoring)**

- High section: Replace "Drive" with "LocalDoc" in three-source corroboration description.
- Medium section: Replace "Jira+Drive, Confluence+Drive" with "Jira+LocalDoc, Confluence+LocalDoc".
- Low section: Replace "Drive-only scope item" with "LocalDoc-only scope item".

**Step 5: Update Phase 4 (Scope Item Naming)**

- Priority item 2: Change "Google Drive doc title" to "Local doc title (filename without extension, cleaned up)".
- Stop words in 2d: Remove "google, drive" from the common project terms list. Add "local, file, document" if not already excluded (they may already be covered by "doc, document").

**Step 6: Update Section 3 (Output Schema)**

- Remove `"drive_email"` field from the top-level schema.
- Add `"local_docs_subfolder": "data-pipeline"` (or null) field.
- Replace `"drive_sources"` array with `"local_doc_sources"` array. Update the inner object to use `file_path` and `file_name` instead of `file_id`, `url`, and `title`. Remove `url` field. Schema:
  ```json
  "local_doc_sources": [
    {
      "file_path": "prd.pdf",
      "file_name": "prd.pdf",
      "file_type": "pdf",
      "match_type": "explicit_reference",
      "dates_mentioned": [...],
      "deliverables_mentioned": [...]
    }
  ]
  ```
- In the orphans section: Replace `"unmatched_drive"` with `"unmatched_local_docs"`. Update inner object from `file_id`/`title` to `file_path`/`file_name`.
- In the cross_references section: Replace `"drive_file_id"` with `"local_doc_path"`. Update evidence text examples from "Drive doc" to "Local doc".

**Step 7: Update Section 4 (Worked Examples)**

- Example 4 (Three-Source Corroboration): Change "Google Drive doc" to "Local doc". Replace `file_id` with `file_path`. Remove `url` references. Change the Drive-specific query language to local file context.

**Step 8: Update Section 5 (Edge Cases)**

- "Overly Broad Documents": Replace "Drive doc" with "local doc".
- "Empty Drive Results": Rename to "Empty Local Docs Results". Change `raw/google-drive-docs.json` to `raw/local-docs.json`. Update description.
- "Drive-Only Scope Items": Rename to "LocalDoc-Only Scope Items". Replace "Drive doc" with "local doc". Change `unmatched_drive` to `unmatched_local_docs`.
- "Conflicting Dates": Replace `drive` source type with `local_doc`. Remove "(or Drive)" references, replace with "(or local doc)".

**Step 9: Commit**

```bash
git add skills/project-insights/references/synthesis-model.md
git commit -m "feat: replace drive_sources with local_doc_sources in synthesis model"
```

---

### Task 7: Update the report template

**Files:**
- Modify: `skills/project-insights/references/report-template.html`

**Step 1: Update source count references in scope cards**

The template uses comment markers for source counts. The current card-meta section (around line 489) shows:

```html
<span><!-- SCOPE_ITEM_CONFLUENCE_COUNT -->4<!-- /SCOPE_ITEM_CONFLUENCE_COUNT --> Confluence pages</span>
<span><!-- SCOPE_ITEM_JIRA_COUNT -->22<!-- /SCOPE_ITEM_JIRA_COUNT --> Jira tickets</span>
```

There is no Drive count in the template currently (it was covered by the report generation pass). No changes are needed to the HTML template structure — the report generation pass (Pass 4 in SKILL.md) already handles populating the cards dynamically. The instruction to show "P Local docs" instead of "P Drive docs" is in SKILL.md (updated in Task 5).

Verify: Search the template for any "Drive" or "Google" text references. If found, replace with "Local doc" equivalents. Based on the current file, there are no hardcoded Drive references in the HTML — it uses comment markers that the generation pass fills in.

**Step 2: Commit (only if changes were made)**

```bash
git add skills/project-insights/references/report-template.html
git commit -m "feat: update report template for local doc sources"
```

---

### Task 8: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Update Plugin Structure section**

Replace the file tree (lines 7-25). Changes:
- Remove `google-drive-discovery.md` line
- Add `local-docs-discovery.md` line
- Remove `google-drive-mcp-reference.md` line
- Change `.mcp.json` description from `(Atlassian + Google Workspace)` to `(Atlassian)`
- Add `external-docs/` to the tree

New tree:
```
.claude-plugin/plugin.json    - Plugin manifest
.mcp.json                     - MCP server configuration (Atlassian)
commands/project-insights.md  - Slash command definition
agents/
  jira-discovery.md           - Jira querying agent (haiku)
  confluence-discovery.md     - Confluence querying agent (haiku)
  local-docs-discovery.md     - Local document discovery agent (haiku)
  risk-assessment.md          - Risk scoring agent (sonnet)
skills/project-insights/
  SKILL.md                    - Main orchestration logic
  references/
    synthesis-model.md        - Scope item synthesis algorithm
    mcp-tool-reference.md     - Atlassian MCP tool names and parameters
    report-template.html      - HTML/CSS report template
  examples/
    sample-report.html        - Example generated report
external-docs/                - Local documents for analysis (gitignored)
reports/                      - Generated report output (gitignored)
```

**Step 2: Update Model Specifications**

Replace `google-drive-discovery agent:  haiku (fast, structured extraction)` with:
```
local-docs-discovery agent:    haiku (fast, structured extraction)
```

**Step 3: Update MCP Authentication section**

- Update "Session Startup" text: Remove references to `google-workspace` server. Remove step 3 about testing Google Drive. Remove step about reconnecting "both" servers (now just Atlassian).
- Remove the entire "### Google Drive (optional)" section (lines 70-87).
- Update the description line at the top: Change "Synthesizes scope from Jira, Confluence, and Google Drive" to "Synthesizes scope from Jira, Confluence, and local documents".

**Step 4: Add Local Documents section**

Add a new section after "### Atlassian (required)" called "### Local Documents (optional)":

```markdown
### Local Documents (optional)

Local document discovery is optional. When `--local-docs <subfolder>` is provided, the plugin scans `external-docs/<subfolder>/` for planning documents (.docx, .pdf, .md, .txt).

Prerequisites:
- `pandoc` for .docx extraction: `brew install pandoc`
- `pdftotext` for .pdf extraction: `brew install poppler`

Usage:
1. Create a subfolder under `external-docs/` (e.g., `external-docs/data-pipeline/`)
2. Place exported planning documents in the subfolder
3. Run the plugin with `--local-docs data-pipeline`

Processed documents are cached in `external-docs/<subfolder>/.cache.json`. Use `--reprocess-docs` to force re-extraction.
```

**Step 5: Update "How to Test" section**

Replace:
```
/project-insights --jira PROJ --spaces TEAMSPACE --drive-email user@example.com --search "your feature"
```
with:
```
/project-insights --jira PROJ --spaces TEAMSPACE --local-docs my-project --search "your feature"
```

Update the skill steps list to replace "Google Drive" references with "local documents".

**Step 6: Update "Key Design Decisions"**

- Change "Many-to-many relationships" bullet: Replace "AND Google Drive docs" with "AND local documents"
- Remove "No cursor pagination in Drive" bullet
- Change "Optional Google Drive" to: `**Optional local documents:** Local doc discovery is enabled by --local-docs; synthesis model gracefully degrades to 2-source when local docs are absent`
- Add: `**Processing cache:** Local docs are cached after first extraction to avoid redundant reprocessing`

**Step 7: Update "4-Pass Architecture"**

Change `Jira + Confluence + Drive` to `Jira + Confluence + Local Docs`.

**Step 8: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for local docs, remove Google Drive references"
```

---

### Task 9: Verify all Drive references are removed

**Step 1: Search for remaining Drive references**

Run grep across the entire codebase for any remaining Google Drive references:

```bash
grep -ri "google.drive\|drive.email\|drive.folder\|google.workspace\|workspace.mcp\|drive_sources\|unmatched_drive\|drive_file_id\|google-drive\|drive-discovery\|google_drive\|--drive" --include="*.md" --include="*.json" --include="*.html" .
```

Expected: No results (all references removed). If any remain, fix them.

**Step 2: Search for any leftover MCP references**

```bash
grep -ri "google-workspace\|workspace-mcp\|search_drive_files\|get_doc_content\|get_drive_file_content\|list_drive_items" --include="*.md" --include="*.json" .
```

Expected: No results. If any remain, fix them.

**Step 3: Verify file structure**

```bash
ls agents/
# Expected: confluence-discovery.md  jira-discovery.md  local-docs-discovery.md  risk-assessment.md

ls skills/project-insights/references/
# Expected: mcp-tool-reference.md  report-template.html  synthesis-model.md

cat .mcp.json
# Expected: only atlassian server, no google-workspace

ls external-docs/
# Expected: .gitkeep
```

**Step 4: Commit any fixes**

If any remaining references were found and fixed:

```bash
git add -A
git commit -m "chore: clean up remaining Google Drive references"
```

---

### Task 10: Final review commit

**Step 1: Review all changes**

```bash
git log --oneline HEAD~10..HEAD
git diff main..HEAD --stat
```

Verify the change set matches the design document expectations.

**Step 2: Verify no regressions**

- Confirm `agents/jira-discovery.md` and `agents/confluence-discovery.md` are unchanged
- Confirm `agents/risk-assessment.md` is unchanged
- Confirm `skills/project-insights/references/mcp-tool-reference.md` is unchanged
- Confirm `skills/project-insights/examples/sample-report.html` is unchanged (it's example output, not a template)
