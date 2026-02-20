# Project Insights Dashboard

Claude Code plugin for project insights reporting. Synthesizes scope from Jira and Confluence, assesses execution risk, and generates executive-level HTML reports.

## Plugin Structure

```
.claude-plugin/plugin.json    - Plugin manifest
.mcp.json                     - MCP server configuration
commands/project-insights.md  - Slash command definition
agents/
  jira-discovery.md           - Jira querying agent (haiku)
  confluence-discovery.md     - Confluence querying agent (haiku)
  risk-assessment.md          - Risk scoring agent (sonnet)
skills/project-insights/
  SKILL.md                    - Main orchestration logic
  references/
    synthesis-model.md        - Scope item synthesis algorithm
    mcp-tool-reference.md     - Documented MCP tool names and parameters
    report-template.html      - HTML/CSS report template
  examples/
    sample-report.html        - Example generated report
reports/                      - Generated report output (gitignored)
```

## Model Specifications

```
Model assignments for agents and passes:
- jira-discovery agent:        haiku (fast, structured extraction)
- confluence-discovery agent:   haiku (fast, structured extraction)
- risk-assessment agent:        sonnet (reasoning about risk scoring)
- Pass 1b synthesis:            opus (complex cross-referencing and clustering)
- Pass 4 report generation:     sonnet (narrative writing with structure)
```

## MCP Authentication

This plugin requires the Atlassian MCP server for Jira and Confluence access.

To authenticate:
1. Run /mcp in a Claude Code session
2. The Atlassian server should appear in the list
3. If not authenticated, the OAuth flow will launch in your browser
4. Grant access to the required Jira and Confluence scopes
5. Return to Claude Code -- the session should now have MCP access

The MCP server is configured in `.mcp.json` with HTTP transport to `https://mcp.atlassian.com/v1/mcp`

Actual MCP tool names and parameters are documented in:
  `skills/project-insights/references/mcp-tool-reference.md`

## How to Test

```
/project-insights --jira PROJ --spaces TEAMSPACE --search "your feature"
```

The skill will:
1. Verify MCP authentication
2. Run parallel discovery agents against Jira and Confluence
3. Synthesize findings into scope items (many-to-many relationships)
4. Present scope items for your review (checkpoint)
5. Assess risk for each scope item using weighted scoring
6. Generate an HTML report in `reports/YYYY-MM-DD-PROJ-slug/`

## Key Design Decisions

- **File-backed architecture:** Agents write to JSON files, not context, to prevent context saturation
- **Many-to-many relationships:** Scope items can reference multiple Jira tickets AND multiple Confluence pages
- **Weighted risk scoring:** Numeric scores (0-100+) mapped to three ratings (on_track / at_risk / behind)
- **Single risk agent:** One agent processes all scope items in one pass with batch Jira queries
- **HTML output:** Self-contained HTML with embedded CSS and Roboto typography
- **Sub-agent type:** Agents must be dispatched as `general-purpose` (not Bash) to use the Write tool
- **No pagination offset in Confluence:** `confluence_search` has no offset parameter; agent uses multiple targeted CQL queries instead

## 4-Pass Architecture

```
Pass 1:  Discovery (parallel haiku agents) → raw/*.json
Pass 1b: Synthesis (opus, parent context) → synthesized/scope-items.json
         ↓ Checkpoint: user reviews scope items
Pass 2:  Refined Discovery (if needed) → synthesized/scope-items-refined.json
Pass 3:  Risk Assessment (single sonnet agent) → synthesized/risk-assessments.json + people.json
Pass 4:  Report Generation (sonnet, parent) → project-insights.html
```
