# Project Insights Dashboard

A Claude Code plugin that generates executive-level project health reports by synthesizing data from Jira, Confluence, and local documents. It discovers scope, cross-references planning documents with execution tickets, assesses risk using weighted scoring, and produces a styled HTML report.

## Quick Start

### 1. Install the Plugin

Copy or clone this directory into your project, or install it as a Claude Code plugin. The plugin is recognized by the `.claude-plugin/plugin.json` manifest.

### 2. Configure Atlassian MCP

The plugin needs access to your Jira and Confluence instances through the Atlassian MCP server.

**Option A: Use the bundled config**

The included `.mcp.json` points to the Atlassian remote MCP server:

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

**Option B: Use the community MCP server (mcp-atlassian)**

If you prefer the open-source [mcp-atlassian](https://github.com/sooperset/mcp-atlassian) server (which provides more tools), install it and update `.mcp.json`:

```json
{
  "mcpServers": {
    "atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian"]
    }
  }
}
```

Configure authentication via environment variables (API token is simplest):

```bash
export JIRA_URL=https://your-company.atlassian.net
export JIRA_USERNAME=your.email@example.com
export JIRA_API_TOKEN=your_api_token
export CONFLUENCE_URL=https://your-company.atlassian.net/wiki
export CONFLUENCE_USERNAME=your.email@example.com
export CONFLUENCE_API_TOKEN=your_api_token
```

### 3. Set Up Local Documents (Optional)

To also discover planning documents from local files (PRDs, specs, roadmaps exported as .docx, .pdf, .md, .txt), place them in a subfolder under `external-docs/`.

Prerequisites:
- `pandoc` for .docx extraction: `brew install pandoc`
- `pdftotext` for .pdf extraction: `brew install poppler`

Setup:
1. Create a subfolder: `mkdir -p external-docs/my-project`
2. Copy your planning documents into the subfolder
3. Run the plugin with `--local-docs my-project`

Processed documents are cached in `external-docs/<subfolder>/.cache.json` to avoid redundant re-extraction on subsequent runs.

### 4. Authenticate

Start a Claude Code session and run `/mcp` to verify the Atlassian server is connected. If prompted, complete the OAuth flow in your browser.

### 5. Generate a Report

```
/project-insights --jira PROJ --spaces TEAMSPACE
```

That's it. The skill walks you through the rest interactively.

## Usage

### Basic Command

```
/project-insights --jira <PROJECT_KEY> --spaces <SPACE_KEY>
```

### Full Options

```
/project-insights --jira <keys> --spaces <keys> [--local-docs SUBFOLDER] [--reprocess-docs] [--search "keywords"] [--target-date YYYY-MM-DD] [--stale-days N]
```

| Argument | Required | Description |
|----------|----------|-------------|
| `--jira` | Yes | Comma-separated Jira project key(s). Example: `PROJ` or `PROJ,ENG,INFRA` |
| `--spaces` | Yes | Comma-separated Confluence space key(s). Example: `TEAMSPACE` or `TEAM,DOCS` |
| `--local-docs` | No | Subfolder name under `external-docs/`. Enables local document discovery. |
| `--reprocess-docs` | No | Force re-extraction of all local docs, ignoring the processing cache. |
| `--search` | No | Free-text keywords to narrow discovery. Example: `"data pipeline migration"` |
| `--target-date` | No | Overall project deadline in `YYYY-MM-DD` format. Used for time-based risk scoring. |
| `--stale-days` | No | Days of inactivity before a ticket is flagged as stale. Default: `14`. |

### Examples

**Single project, single space:**
```
/project-insights --jira ACME --spaces ACMEENG
```

**Multiple projects and spaces with keyword focus:**
```
/project-insights --jira ACME,INFRA --spaces ACMEENG,PLATFORM --search "data pipeline"
```

**With a deadline and custom staleness threshold:**
```
/project-insights --jira ACME --spaces ACMEENG --target-date 2026-06-30 --stale-days 7
```

**With local document discovery:**
```
/project-insights --jira ACME --spaces ACMEENG --local-docs my-project --search "data pipeline"
```

**Force re-extraction of local docs:**
```
/project-insights --jira ACME --spaces ACMEENG --local-docs my-project --reprocess-docs
```

**Broad discovery (no keyword filter):**
```
/project-insights --jira ACME --spaces ACMEENG
```
This discovers all epics/initiatives in the project and all planning documents (PRDs, RFCs, specs, roadmaps) in the space and optionally local docs.

## What Happens When You Run It

The skill executes a 4-pass pipeline:

### Pass 1: Discovery
Up to three agents run in parallel:
- **Jira agent** queries your project for epics, initiatives, and stories. It auto-discovers issue types (doesn't assume "Epic" exists) and paginates through all results.
- **Confluence agent** searches your space for planning documents by label, title, and keywords. It extracts dates, deliverables, and Jira key mentions from page content.
- **Local docs agent** (if `--local-docs` provided) scans the specified subfolder for planning documents (.docx, .pdf, .md, .txt). It extracts dates, deliverables, Jira key mentions, and Confluence references using pandoc and pdftotext.

### Pass 1b: Synthesis
The system cross-references Jira, Confluence, and local doc data to produce **scope items** -- logical groupings of related work. A scope item like "Data Pipeline V2 Migration" might link to 1 Confluence PRD, 1 local doc spec, 1 Jira epic, and 13 child stories. The synthesis uses keyword matching, explicit cross-references (Jira keys in Confluence/local doc pages), and clustering to build these relationships. When all three sources corroborate a scope item, confidence scores increase.

### Checkpoint: Your Review
You're shown the discovered scope items with confidence scores and asked to review:
- Remove incorrect items
- Add missing scope items
- Merge or split items
- Provide additional context (dates, priorities)

This is where your domain knowledge improves the output.

### Pass 3: Risk Assessment
A single agent batch-queries Jira for current ticket statuses and calculates a weighted risk score for each scope item:

| Signal | Impact |
|--------|--------|
| Low progress (< 25% active) | +40 |
| Stale tickets (no updates in N days) | +5 each |
| Blocked tickets | +15 each |
| Untracked scope (0 Jira tickets) | +50 |
| Behind schedule (time vs. progress gap) | +20 to +40 |
| Missed deadline | +30 |

Scores map to ratings: **On Track** (0-15), **At Risk** (16-39), **Behind** (40+).

### Pass 4: Report
A styled HTML report is generated with:
- Executive summary
- Scope item cards with progress bars and risk badges
- Risk dashboard table
- Items requiring attention (blocked, stale, untracked)
- Key people with risk involvement
- Prioritized, actionable recommendations

## Output

Reports are saved to `reports/YYYY-MM-DD-<PROJECT>-<slug>/`:

```
reports/2026-02-19-ACME-pipeline/
├── run-metadata.json           # Run parameters and pass status tracking
├── raw/
│   ├── jira-epics.json         # Raw Jira discovery data
│   ├── jira-stories.json       # Child stories per epic
│   ├── confluence-pages.json   # Confluence planning documents
│   └── local-docs.json         # Local planning documents (if --local-docs used)
├── synthesized/
│   ├── scope-items.json        # Initial synthesis output
│   ├── scope-items-refined.json # After your review
│   ├── risk-assessments.json   # Risk scores and details
│   └── people.json             # People-to-risk mapping
└── project-insights.html       # The final HTML report
```

The intermediate JSON files are preserved so you can inspect exactly what was discovered and how it was synthesized. This is useful for debugging or feeding into other tools.

## Ad Hoc and Custom Reports

### Re-running with Different Parameters

Each run creates a new directory, so you can generate multiple reports:

```
/project-insights --jira ACME --spaces ACMEENG --search "authentication"
/project-insights --jira ACME --spaces ACMEENG --search "data pipeline"
/project-insights --jira ACME,INFRA --spaces ACMEENG,PLATFORM
```

### Focusing on a Specific Area

Use `--search` to narrow the scope to a specific feature or workstream:

```
/project-insights --jira ACME --spaces ACMEENG --search "monitoring observability"
```

The agents will prioritize tickets and pages matching those keywords, giving you a focused report on that area.

### Adjusting Sensitivity

Lower `--stale-days` for fast-moving sprints, raise it for longer cycles:

```
# Aggressive: flag anything untouched for a week
/project-insights --jira ACME --spaces ACMEENG --stale-days 7

# Relaxed: only flag items dormant for a month
/project-insights --jira ACME --spaces ACMEENG --stale-days 30
```

### Resuming a Failed Run

If a run fails mid-way (e.g., MCP auth expires), the next run can resume from where it left off. The `run-metadata.json` file tracks which passes completed.

### Using the Raw Data

The JSON files in `raw/` and `synthesized/` are structured and documented. You can use them for:
- Feeding into dashboards or spreadsheets
- Custom analysis scripts
- Comparing scope drift across multiple report runs
- Audit trails of what was discovered and how items were linked

## Customizing the Report

### Modifying the HTML Template

Edit `skills/project-insights/references/report-template.html` to change:
- Colors, fonts, or layout
- Add or remove sections
- Adjust the risk badge styling
- Add your company logo or branding

The template uses standard HTML/CSS with Google Fonts. No JavaScript dependencies.

### Adjusting Risk Scoring

The risk heuristic weights are defined in two places:
- `agents/risk-assessment.md` -- the agent's scoring logic (change the weights here)
- `skills/project-insights/SKILL.md` -- the reference table (update to match)

### Changing Model Assignments

Model assignments are in `skills/project-insights/SKILL.md` and each agent's frontmatter:
- Discovery agents (Jira, Confluence, Local Docs) use **haiku** (fast, cheap) -- suitable for structured extraction
- Risk assessment uses **sonnet** -- needs reasoning about scoring
- Synthesis uses **opus** -- complex cross-referencing and clustering

You can change these if you want to trade speed for quality or vice versa.

## Example Report

Open `skills/project-insights/examples/sample-report.html` in a browser to see what a generated report looks like. It shows a fictional project with 6 scope items at varying risk levels.

## Troubleshooting

**"Atlassian MCP authentication failed"**
Run `/mcp` in Claude Code and complete the OAuth flow. The skill will not proceed without valid authentication.

**Empty results from Jira or Confluence**
- Verify your project key and space key are correct (they're case-sensitive)
- Check that your MCP credentials have read access to the project/space
- Try a broader search without `--search` keywords

**Scope items don't look right at checkpoint**
This is exactly what the checkpoint is for. Remove, merge, split, or add items based on your knowledge. The system learns from your input in Pass 2.

**Report is missing some tickets**
The Jira agent paginates through results but searches by epic/initiative type first. Flat stories without parent epics may be missed unless they match keyword searches or have scope-related labels (roadmap, milestone, scope).

**Local docs extraction fails**
- Ensure `pandoc` is installed: `brew install pandoc`
- Ensure `pdftotext` is installed: `brew install poppler`
- Check that documents are not corrupted or password-protected
- Use `--reprocess-docs` to force re-extraction and see fresh error messages

**Local docs finds no documents**
- Verify the subfolder exists under `external-docs/` and contains supported files (.docx, .pdf, .md, .txt)
- Check the subfolder name matches what you passed to `--local-docs`

## Architecture

See `CLAUDE.md` for detailed architecture documentation, model specifications, and design decisions.
