# Local File Discovery + Google Drive Removal

**Date:** 2026-02-20
**Status:** Approved

## Summary

Replace Google Workspace MCP (which requires OAuth credentials users may not have) with local file discovery. Users drop exported documents (.docx, .pdf, .md, .txt) into `external-docs/<subfolder>/` and the plugin processes them as a third source alongside Jira and Confluence. A processing cache prevents redundant re-extraction across runs.

## Requirements

1. **Local docs folder convention:** `external-docs/<name>/` in the project root. Gitignored. User specifies the subfolder via `--local-docs <name>`.
2. **Supported formats:** `.docx`, `.pdf`, `.md`, `.txt`. Extracted via `pandoc` (docx), `pdftotext` (pdf), or Read tool (md/txt).
3. **Processing cache:** `external-docs/<subfolder>/.cache.json` stores extracted content keyed by filename + mtime. Unchanged files are not reprocessed. `--reprocess-docs` flag forces full re-extraction.
4. **Google Drive removal:** Clean break. Delete agent, MCP reference, `.mcp.json` entry, all `--drive-*` CLI flags.

## Approach

Mirror the existing agent pattern. A new `local-docs-discovery.md` haiku agent replaces the deleted `google-drive-discovery.md`. It writes `raw/local-docs.json` in a schema parallel to the old `raw/google-drive-docs.json`. The synthesis model replaces `drive_sources` with `local_doc_sources`. The rest of the pipeline (risk, report) requires minimal changes.

## File Plan

### New files

| File | Purpose |
|------|---------|
| `agents/local-docs-discovery.md` | Haiku agent for local file extraction |
| `external-docs/.gitkeep` | Preserve directory in git |

### Deleted files

| File | Reason |
|------|--------|
| `agents/google-drive-discovery.md` | Google Drive no longer supported |
| `skills/project-insights/references/google-drive-mcp-reference.md` | Google Drive no longer supported |

### Modified files

| File | Changes |
|------|---------|
| `.mcp.json` | Remove `google-workspace` server entry |
| `.gitignore` | Add `external-docs/` (except `.gitkeep`) |
| `commands/project-insights.md` | Replace `--drive-email`/`--drive-folders` with `--local-docs`, add `--reprocess-docs` |
| `skills/project-insights/SKILL.md` | Replace Drive agent dispatch with local-docs agent; remove Drive MCP health check; add cache awareness |
| `skills/project-insights/references/synthesis-model.md` | Replace `drive_sources` with `local_doc_sources` throughout all 4 phases |
| `skills/project-insights/references/report-template.html` | Update source badges from "Google Drive" to "Local Doc" |
| `CLAUDE.md` | Update plugin structure, architecture docs, remove Google Workspace setup |

## Agent Contract: local-docs-discovery

**Model:** haiku
**Subagent type:** general-purpose (needs Write + Bash tools)

### Inputs

| Parameter | Type | Description |
|-----------|------|-------------|
| `local_docs_dir` | string | Absolute path to `external-docs/<subfolder>/` |
| `run_dir` | string | Report output directory |
| `cache_path` | string | Path to `.cache.json` |
| `reprocess` | boolean | Force re-extract all files |
| `search_keywords` | string | Optional search terms |

### Agent Logic

1. List files recursively: `.docx`, `.pdf`, `.md`, `.txt`
2. Read `.cache.json` if it exists
3. For each file: compare mtime. If unchanged and `reprocess` is false, use cached extraction.
4. For new/changed files:
   - `.docx` → `pandoc -t plain <file>` (shell)
   - `.pdf` → `pdftotext <file> -` (shell)
   - `.md`, `.txt` → Read tool
5. From extracted text, identify:
   - `content_summary`: first 500 chars of scope-relevant content
   - `dates_mentioned`: ISO dates, US dates, relative dates (same patterns as Confluence agent)
   - `deliverables_mentioned`: key nouns/phrases from headings and bullet points
   - `jira_keys_mentioned`: regex `[A-Z][A-Z0-9]+-\d+`
6. Write updated `.cache.json`
7. Write `raw/local-docs.json`

### Cache Format

```json
{
  "version": 1,
  "processed": {
    "prd.pdf": {
      "mtime": "2026-02-19T14:30:00Z",
      "size_bytes": 245000,
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

### Output Schema: `raw/local-docs.json`

```json
{
  "source_dir": "external-docs/data-pipeline",
  "total_docs": 3,
  "docs": [
    {
      "file_path": "prd.pdf",
      "file_name": "prd.pdf",
      "file_type": "pdf",
      "last_modified": "2026-02-19T14:30:00Z",
      "from_cache": false,
      "content_summary": "[extracted scope - max 500 chars]",
      "dates_mentioned": [
        {
          "date": "2026-04-15",
          "context": "target launch date",
          "format_detected": "iso",
          "confidence": "high"
        }
      ],
      "deliverables_mentioned": ["data pipeline v2", "schema validation"],
      "jira_keys_mentioned": ["PROJ-101"]
    }
  ]
}
```

## Synthesis Model Changes

### Phase 1: Cross-Reference Extraction

- Remove: Drive → Jira, Drive → Confluence maps
- Add: LocalDoc → Jira (via `jira_keys_mentioned`), LocalDoc → Confluence (via URL pattern matching in extracted text)
- Identifier: `file_path` (local docs have no stable external IDs)

### Phase 2: Initial Clustering

Replace "Drive-anchored" with "LocalDoc-anchored" at the same priority position (after epic-anchored and confluence-anchored):

1. Epic-anchored (highest priority)
2. Confluence-anchored
3. **LocalDoc-anchored** (replaces Drive-anchored)
4. Keyword-overlap (fallback)
5. Orphans

### Phase 3: Confidence Scoring

- Three-source corroboration: Jira + Confluence + LocalDoc → +0.15 boost (same as Drive)
- "LocalDoc-only scope" → low confidence (same as Drive-only was)

### Phase 4: Scope Item Naming

- Add: LocalDoc title (filename without extension, cleaned up) as naming candidate
- Priority: Confluence PRD/RFC title > LocalDoc title > Jira epic summary > synthesized keywords

### Schema Changes

Replace `drive_sources` with `local_doc_sources` in scope items:

```json
"local_doc_sources": [
  {
    "file_path": "prd.pdf",
    "file_name": "prd.pdf",
    "file_type": "pdf",
    "match_type": "explicit_reference",
    "dates_mentioned": [],
    "deliverables_mentioned": []
  }
]
```

Replace `unmatched_drive` with `unmatched_local_docs` in orphans.

## Orchestration Changes (SKILL.md)

- **MCP health gate:** Remove Google Workspace check. Only Atlassian remains.
- **Pass 1:** Dispatch local-docs agent in parallel with Jira + Confluence (only if `--local-docs` provided).
- **Pass 1 verification:** Check `raw/local-docs.json` (only if agent was dispatched).
- **Graceful degradation:** If `--local-docs` not provided, 2-source mode (Jira + Confluence only). No penalty.

## CLI Changes

### Removed flags

- `--drive-email`
- `--drive-folders`

### New flags

| Flag | Required | Description |
|------|----------|-------------|
| `--local-docs <subfolder>` | No | Subfolder name under `external-docs/`. Enables local doc discovery. |
| `--reprocess-docs` | No | Force re-extraction of all local docs (ignore cache). |

### Updated invocation

```
/project-insights --jira PROJ --spaces TEAMSPACE --local-docs data-pipeline --search "data pipeline"
/project-insights --jira PROJ --spaces TEAMSPACE --local-docs data-pipeline --reprocess-docs
```

## Report Template Changes

- Source badge: "Google Drive" → "Local Doc"
- Source count: "1 Local doc" instead of "1 Drive doc"
- No structural HTML/CSS changes needed

## Prerequisites

Users must have installed:
- `pandoc` (for .docx extraction): `brew install pandoc`
- `pdftotext` (for .pdf extraction): `brew install poppler`

The agent should check for these tools at the start and report a clear error if missing.
