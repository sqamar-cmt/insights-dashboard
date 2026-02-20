# Project Insights Dashboard

Claude Code plugin for project insights reporting. Synthesizes scope from Jira, Confluence, and local documents, assesses execution risk, and generates executive-level HTML reports.

## Plugin Structure

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

## Model Specifications

```
Model assignments for agents and passes:
- jira-discovery agent:          haiku (fast, structured extraction)
- confluence-discovery agent:    haiku (fast, structured extraction)
- local-docs-discovery agent:    haiku (fast, structured extraction)
- risk-assessment agent:         sonnet (reasoning about risk scoring)
- Pass 1b synthesis:             opus (complex cross-referencing and clustering)
- Pass 4 report generation:      sonnet (narrative writing with structure)
```

## MCP Authentication

### Session Startup: Re-authentication Required

**At the start of every new Claude Code session**, run `/mcp` to reconnect and re-authenticate the Atlassian MCP server. Previous OAuth tokens may have expired between sessions. Do not assume MCP access carries over from a prior session.

Validation steps after reconnecting:
1. Run `/mcp` to reconnect the `atlassian` server
2. Test Atlassian by calling `getAccessibleAtlassianResources` -- confirm the cloud ID `0f350409-7469-4766-9759-84b56e79f2f5` is returned for `cmtelematics.atlassian.net`
3. If any server fails to authenticate, re-run `/mcp` and complete the OAuth flow in the browser

### Atlassian (required)

This plugin requires the Atlassian MCP server for Jira and Confluence access.

**Cloud ID:** `0f350409-7469-4766-9759-84b56e79f2f5` (site: `cmtelematics.atlassian.net`)

To authenticate:
1. Run /mcp in a Claude Code session
2. The Atlassian server should appear in the list
3. If not authenticated, the OAuth flow will launch in your browser
4. Grant access to the required Jira and Confluence scopes
5. Return to Claude Code -- the session should now have MCP access

The MCP server is configured in `.mcp.json` with HTTP transport to `https://mcp.atlassian.com/v1/mcp`

Atlassian MCP tool names and parameters are documented in:
  `skills/project-insights/references/mcp-tool-reference.md`

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

## How to Test

```
/project-insights --jira PROJ --spaces TEAMSPACE --search "your feature"
/project-insights --jira PROJ --spaces TEAMSPACE --local-docs my-project --search "your feature"
```

The skill will:
1. Verify MCP authentication (Atlassian)
2. Run parallel discovery agents against Jira, Confluence, and optionally local documents
3. Synthesize findings into scope items (many-to-many relationships across all sources)
4. Present scope items for your review (checkpoint)
5. Assess risk for each scope item using weighted scoring
6. Generate an HTML report in `reports/YYYY-MM-DD-PROJ-slug/`

## Key Design Decisions

- **File-backed architecture:** Agents write to JSON files, not context, to prevent context saturation
- **Many-to-many relationships:** Scope items can reference multiple Jira tickets, Confluence pages, AND local documents
- **Weighted risk scoring:** Numeric scores (0-100+) mapped to three ratings (on_track / at_risk / behind)
- **Single risk agent:** One agent processes all scope items in one pass with batch Jira queries
- **HTML output:** Self-contained HTML with embedded CSS and Roboto typography
- **Sub-agent type:** Agents must be dispatched as `general-purpose` (not Bash) to use the Write tool
- **No pagination offset in Confluence:** `confluence_search` has no offset parameter; agent uses multiple targeted CQL queries instead
- **Optional local documents:** Local doc discovery is enabled by `--local-docs`; synthesis model gracefully degrades to 2-source when local docs are absent
- **Processing cache:** Local docs are cached after first extraction to avoid redundant reprocessing

## 4-Pass Architecture

```
Pass 1:  Discovery (parallel haiku agents: Jira + Confluence + Local Docs) → raw/*.json
Pass 1b: Synthesis (opus, parent context) → synthesized/scope-items.json
         ↓ Checkpoint: user reviews scope items
Pass 2:  Refined Discovery (if needed) → synthesized/scope-items-refined.json
Pass 3:  Risk Assessment (single sonnet agent) → synthesized/risk-assessments.json + people.json
Pass 4:  Report Generation (sonnet, parent) → project-insights.html
```
