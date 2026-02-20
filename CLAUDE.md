# Project Insights Dashboard

Claude Code plugin for project insights reporting. Synthesizes scope from Jira, Confluence, and Google Drive, assesses execution risk, and generates executive-level HTML reports.

## Plugin Structure

```
.claude-plugin/plugin.json    - Plugin manifest
.mcp.json                     - MCP server configuration (Atlassian + Google Workspace)
commands/project-insights.md  - Slash command definition
agents/
  jira-discovery.md           - Jira querying agent (haiku)
  confluence-discovery.md     - Confluence querying agent (haiku)
  google-drive-discovery.md   - Google Drive querying agent (haiku)
  risk-assessment.md          - Risk scoring agent (sonnet)
skills/project-insights/
  SKILL.md                    - Main orchestration logic
  references/
    synthesis-model.md        - Scope item synthesis algorithm
    mcp-tool-reference.md     - Atlassian MCP tool names and parameters
    google-drive-mcp-reference.md - Google Drive MCP tool names and parameters
    report-template.html      - HTML/CSS report template
  examples/
    sample-report.html        - Example generated report
reports/                      - Generated report output (gitignored)
```

## Model Specifications

```
Model assignments for agents and passes:
- jira-discovery agent:          haiku (fast, structured extraction)
- confluence-discovery agent:    haiku (fast, structured extraction)
- google-drive-discovery agent:  haiku (fast, structured extraction)
- risk-assessment agent:         sonnet (reasoning about risk scoring)
- Pass 1b synthesis:             opus (complex cross-referencing and clustering)
- Pass 4 report generation:      sonnet (narrative writing with structure)
```

## MCP Authentication

### Atlassian (required)

This plugin requires the Atlassian MCP server for Jira and Confluence access.

To authenticate:
1. Run /mcp in a Claude Code session
2. The Atlassian server should appear in the list
3. If not authenticated, the OAuth flow will launch in your browser
4. Grant access to the required Jira and Confluence scopes
5. Return to Claude Code -- the session should now have MCP access

The MCP server is configured in `.mcp.json` with HTTP transport to `https://mcp.atlassian.com/v1/mcp`

Atlassian MCP tool names and parameters are documented in:
  `skills/project-insights/references/mcp-tool-reference.md`

### Google Drive (optional)

Google Drive discovery is optional. When `--drive-email` is provided, the plugin searches Google Drive for planning documents (PRDs, specs, roadmaps in Google Docs).

To set up:
1. Install the Google Workspace MCP server: `pip install workspace-mcp` (or use `uvx workspace-mcp`)
2. Create a Google Cloud project and enable the Drive API and Docs API
3. Create OAuth 2.0 credentials (Desktop application type)
4. Set environment variables:
   ```bash
   export GOOGLE_OAUTH_CLIENT_ID=your_client_id
   export GOOGLE_OAUTH_CLIENT_SECRET=your_client_secret
   ```
5. Run /mcp in a Claude Code session -- the google-workspace server should appear
6. On first use, complete the OAuth consent flow in your browser

Google Drive MCP tool names and parameters are documented in:
  `skills/project-insights/references/google-drive-mcp-reference.md`

## How to Test

```
/project-insights --jira PROJ --spaces TEAMSPACE --search "your feature"
/project-insights --jira PROJ --spaces TEAMSPACE --drive-email user@example.com --search "your feature"
```

The skill will:
1. Verify MCP authentication (Atlassian always, Google Drive if `--drive-email` provided)
2. Run parallel discovery agents against Jira, Confluence, and optionally Google Drive
3. Synthesize findings into scope items (many-to-many relationships across all sources)
4. Present scope items for your review (checkpoint)
5. Assess risk for each scope item using weighted scoring
6. Generate an HTML report in `reports/YYYY-MM-DD-PROJ-slug/`

## Key Design Decisions

- **File-backed architecture:** Agents write to JSON files, not context, to prevent context saturation
- **Many-to-many relationships:** Scope items can reference multiple Jira tickets, Confluence pages, AND Google Drive docs
- **Weighted risk scoring:** Numeric scores (0-100+) mapped to three ratings (on_track / at_risk / behind)
- **Single risk agent:** One agent processes all scope items in one pass with batch Jira queries
- **HTML output:** Self-contained HTML with embedded CSS and Roboto typography
- **Sub-agent type:** Agents must be dispatched as `general-purpose` (not Bash) to use the Write tool
- **No pagination offset in Confluence:** `confluence_search` has no offset parameter; agent uses multiple targeted CQL queries instead
- **No cursor pagination in Drive:** `search_drive_files` uses `page_size` only; agent uses multiple targeted queries instead
- **Optional Google Drive:** Drive discovery is enabled by `--drive-email`; synthesis model gracefully degrades to 2-source when Drive is absent

## 4-Pass Architecture

```
Pass 1:  Discovery (parallel haiku agents: Jira + Confluence + Drive) → raw/*.json
Pass 1b: Synthesis (opus, parent context) → synthesized/scope-items.json
         ↓ Checkpoint: user reviews scope items
Pass 2:  Refined Discovery (if needed) → synthesized/scope-items-refined.json
Pass 3:  Risk Assessment (single sonnet agent) → synthesized/risk-assessments.json + people.json
Pass 4:  Report Generation (sonnet, parent) → project-insights.html
```
