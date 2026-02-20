---
name: project-insights
description: >
  Generate executive-level project insights from Jira, Confluence, and Google Drive.
  Trigger phrases: project insights, project health, risk report, scope assessment
version: 0.1.0
---

# Project Insights Skill

Generate executive-level project insights by synthesizing scope from Jira, Confluence, and Google Drive, assessing execution risk, and producing a styled HTML report.

## Architecture Overview

This skill uses a **4-pass file-backed architecture**. All agent outputs persist to local JSON files in a per-run directory. No agent needs to hold all findings in context. Passes are decoupled -- each reads from the previous pass's files.

| Pass | Description | Model | Executor |
|------|-------------|-------|----------|
| Pass 1 | Discovery (Jira + Confluence + Drive) | haiku | Parallel sub-agents |
| Pass 1b | Synthesis | opus | Parent context |
| Checkpoint | User reviews scope items | — | Interactive |
| Pass 2 | Refined Discovery (if needed) | haiku | Sub-agents |
| Pass 3 | Risk Assessment | sonnet | Single sub-agent |
| Pass 4 | Report Generation | sonnet | Parent context |

## MCP Health Check (Hard Gate)

**This is a hard gate. Execute BEFORE any pass. If any required check fails, STOP the entire pipeline immediately. Do NOT proceed to Pass 1. Do NOT attempt automatic retries.**

### Atlassian Auth (always required)

1. Call `mcp__atlassian__getAccessibleAtlassianResources` (no parameters).
2. Verify the response contains cloud ID `0f350409-7469-4766-9759-84b56e79f2f5` for `cmtelematics.atlassian.net`.
3. **If the call succeeds** and the cloud ID is present:
   - Record auth as verified in `run-metadata.json`.
4. **If the call fails for ANY reason** (auth error, connection error, timeout, missing cloud ID):
   - Inform the user: "Atlassian MCP health check failed. Please run `/mcp` and complete the OAuth flow for the Atlassian server, then try again."
   - Set `run-metadata.json` status to `"error"` with `error_message: "Atlassian MCP health check failed"`.
   - **STOP. Do NOT proceed.**

### Google Drive Auth (only if `--drive-email` provided)

1. Call `mcp__google-workspace__search_drive_files` with `user_google_email` set to the `--drive-email` value, `query: "trashed = false"`, `page_size: 1`.
2. **If the call succeeds** (returns results or an empty list):
   - Record Drive auth as verified in `run-metadata.json`.
3. **If the call fails for ANY reason** (auth error, connection error, timeout):
   - Inform the user: "Google Workspace MCP health check failed. Please run `/mcp` and complete the OAuth flow for the google-workspace server, then try again."
   - Set `run-metadata.json` status to `"error"` with `error_message: "Google Workspace MCP health check failed"`.
   - **STOP. Do NOT proceed.**

### Both Checks Must Pass

Only after ALL required checks pass (Atlassian always; Google Workspace if `--drive-email` provided), proceed to Run Directory Setup and Pass 1.

## Run Directory Setup

Create a per-run directory for all output files.

**Directory naming:** `reports/YYYY-MM-DD-<first-jira-key>-<slug>/`
- `<slug>` = first search keyword, kebab-cased. If no search keywords, use "general".
- If the directory already exists, append `-2`, `-3`, etc.

**Directory structure:**
```
reports/YYYY-MM-DD-PROJ-pipeline/
├── run-metadata.json
├── raw/
│   ├── jira-epics.json
│   ├── jira-stories.json
│   ├── confluence-pages.json
│   └── google-drive-docs.json    (only if --drive-email provided)
└── synthesized/
    ├── scope-items.json
    ├── scope-items-refined.json
    ├── risk-assessments.json
    └── people.json
```

**Create directories:**
```
mkdir -p <run-dir>/raw/
mkdir -p <run-dir>/synthesized/
```

**Write initial `run-metadata.json`:**

```json
{
  "created_at": "<ISO timestamp>",
  "status": "in_progress",
  "current_pass": "setup",
  "parameters": {
    "jira_projects": ["PROJ"],
    "confluence_spaces": ["TEAMSPACE"],
    "drive_email": "user@example.com or null",
    "drive_folder_ids": ["folder_id_1"] ,
    "search_terms": ["data pipeline"],
    "target_date": "2026-04-15",
    "stale_days": 14
  },
  "passes": {
    "discovery": {"status": "pending", "started_at": null, "completed_at": null},
    "synthesis": {"status": "pending", "started_at": null, "completed_at": null},
    "checkpoint": {"status": "pending", "started_at": null, "completed_at": null},
    "refined_discovery": {"status": "pending", "started_at": null, "completed_at": null},
    "risk_assessment": {"status": "pending", "started_at": null, "completed_at": null},
    "report_generation": {"status": "pending", "started_at": null, "completed_at": null}
  },
  "mcp_auth": {
    "atlassian": {
      "verified": true,
      "verified_at": "<ISO timestamp>"
    },
    "google_drive": {
      "verified": true,
      "verified_at": "<ISO timestamp>",
      "note": "Only present if --drive-email provided"
    }
  }
}
```

## Error Handling and Resume

**Before each pass:**
- Update `run-metadata.json`: set the pass status to `"in_progress"` and record `started_at`

**After each pass completes:**
- Update `run-metadata.json`: set the pass status to `"completed"` and record `completed_at`

**If any pass fails:**
- Set `run-metadata.json` status to `"error"` with an `error_message` field
- Inform the user of what failed and suggest next steps

**Atomic write pattern:**
1. Write content to `<filename>.tmp`
2. Read back `<filename>.tmp` to verify it is valid JSON
3. Rename `<filename>.tmp` to `<filename>` (use Bash `mv`)
4. If rename fails, log error and do not delete the `.tmp` file

**On resume (`--resume`):**
- Read `run-metadata.json` from the most recent run directory
- Skip passes with status `"completed"`
- Restart from the first non-completed pass

---

## Pass 1: Discovery

**Model:** haiku (for all discovery agents)

1. Dispatch discovery agents **IN PARALLEL** using the Task tool. Always dispatch Jira and Confluence agents. If `--drive-email` was provided, also dispatch the Google Drive agent:
   - **Jira agent** (`subagent_type: general-purpose`, `model: haiku`): Pass the project key(s), search keywords, and run directory path. The agent will use `jira_search` and `jira_get_issue` MCP tools to find epics, initiatives, and child stories. It writes to `raw/jira-epics.json` and `raw/jira-stories.json`.
   - **Confluence agent** (`subagent_type: general-purpose`, `model: haiku`): Pass the space key(s), search keywords, and run directory path. The agent will use `confluence_search` and `confluence_get_page` MCP tools to find planning documents. It writes to `raw/confluence-pages.json`.
   - **Google Drive agent** (`subagent_type: general-purpose`, `model: haiku`): Pass the drive email, optional folder IDs, search keywords, and run directory path. The agent will use `search_drive_files` and `get_doc_content` MCP tools to find planning documents. It writes to `raw/google-drive-docs.json`. **Only dispatch if `--drive-email` was provided.**
   - Each agent returns only a brief status message (1-2 sentences).

2. Wait for all agents to complete.

3. Verify output files exist:
   - `raw/jira-epics.json` must exist and contain valid JSON
   - `raw/jira-stories.json` must exist and contain valid JSON
   - `raw/confluence-pages.json` must exist and contain valid JSON
   - `raw/google-drive-docs.json` must exist and contain valid JSON **(only if Drive agent was dispatched)**
   - If any required file is missing or invalid, report the error and stop

4. Update `run-metadata.json`: discovery pass status = `"completed"`

---

## Pass 1b: Synthesis

**Model:** opus (parent context)

This is the intellectual core of the skill. Follow the 4-phase algorithm described in `references/synthesis-model.md` **EXACTLY**.

1. Read all `raw/*.json` files into context:
   - `raw/jira-epics.json`
   - `raw/jira-stories.json`
   - `raw/confluence-pages.json`
   - `raw/google-drive-docs.json` (if it exists)

2. Execute the synthesis algorithm:
   a. **Cross-Reference Extraction:** Build the cross-reference map between Jira keys, Confluence pages, and Google Drive docs. Scan Confluence pages for Jira key mentions. Scan Drive docs for Jira key mentions and Confluence URL references. Scan Jira epics for Confluence URLs/titles. Build the epic-to-story hierarchy. If Drive data is present, also build Drive↔Jira and Drive↔Confluence cross-reference maps.
   b. **Initial Clustering:** Create epic-anchored, confluence-anchored, drive-anchored, keyword-overlap, and orphan clusters. Each item belongs to at most one cluster. Priority: epic-anchored > confluence-anchored > drive-anchored > keyword-overlap > orphan.
   c. **Confidence Scoring:** Score each cluster. High (0.8-1.0): explicit cross-references (boost when 3 sources corroborate). Medium (0.5-0.79): keyword overlap with 3+ terms. Low (0.2-0.49): single match or orphan.
   d. **Scope Item Naming:** Name each scope item using the priority hierarchy: Confluence PRD/RFC title > Drive doc title > Jira epic summary > synthesized keywords. Names should be 3-8 words, title case, no Jira keys.

3. Write `synthesized/scope-items.json` following the exact schema in `references/synthesis-model.md`.

4. Include orphans (unmatched Jira and Confluence items) in the output for user review.

5. Include the `cross_references` section for audit trail.

6. Update `run-metadata.json`: synthesis pass status = `"completed"`

---

## Checkpoint: User Refinement

Read `synthesized/scope-items.json` and present to the user.

**Display format:**
- List each scope item with: name, confidence score, source count (N Confluence pages, M Jira tickets, P Drive docs)
- Flag low-confidence items with `[NEEDS REVIEW]`
- Flag orphans separately under "Unmatched Items"
- Show target dates if available

**Prompt the user:**

```
I found [N] scope items from [X] Jira tickets, [Y] Confluence pages, and [Z] Drive docs.

[For each scope item:]
  [confidence emoji] [Name] (confidence: [score])
     Sources: [M] Jira tickets, [N] Confluence pages, [P] Drive docs
     Target date: [date or "none"]

[If orphans exist:]
Unmatched Items:
  Jira: [list unmatched Jira tickets]
  Confluence: [list unmatched Confluence pages]
  Drive: [list unmatched Drive docs]

Please review and let me know:
1. Are any scope items incorrect or should be removed?
2. Are there missing scope items I should add?
3. Should any items be merged or split?
4. Any additional context (dates, priorities, dependencies)?
5. Should I search deeper on any specific topic?
```

Use confidence indicators:
- 0.8-1.0: `[HIGH]`
- 0.5-0.79: `[MED]`
- 0.2-0.49: `[NEEDS REVIEW]`

**Handle user responses:**
- **Removals:** Mark scope items as excluded in `scope-items.json`
- **Additions:** Note user-added items for Pass 2 deeper search
- **Merges:** Combine scope items, union their sources
- **Splits:** Create new scope items with subsets of sources
- **Context:** Add user-provided dates/priorities to the relevant scope items

Update `run-metadata.json`: checkpoint pass status = `"completed"`

---

## Pass 2: Refined Discovery (if needed)

Only execute if the user added new items, requested deeper search, or provided information that requires additional Jira/Confluence/Drive queries.

1. For user-added scope items without system links:
   - Search Jira for matching epics/stories by keyword
   - Search Confluence for matching pages by keyword
   - Search Google Drive for matching docs by keyword (if Drive is enabled)
   - Add any matches to the scope item's sources

2. For deeper search requests:
   - Run additional JQL/CQL/Drive queries as specified by user
   - Append new findings to `raw/*.json` files

3. Write `synthesized/scope-items-refined.json`:
   - Start from `scope-items.json`
   - Apply all user modifications (removals, additions, merges, splits)
   - Incorporate any new discoveries from steps 1-2
   - Recalculate confidence scores where sources changed

**If no refinement is needed,** copy `scope-items.json` to `scope-items-refined.json` unchanged.

Update `run-metadata.json`: refined_discovery pass status = `"completed"`

---

## Pass 3: Risk Assessment

**Model:** sonnet

Dispatch a **SINGLE** `risk-assessment` agent (NOT one per scope item).

1. Dispatch the risk-assessment agent (`subagent_type: general-purpose`, `model: sonnet`) with:
   - Run directory path
   - Stale-days threshold (from parameters)

2. The agent:
   - Reads `synthesized/scope-items-refined.json`
   - Batch-queries Jira for current ticket statuses (50 keys per query)
   - Calculates weighted risk scores for each scope item
   - Writes `synthesized/risk-assessments.json` and `synthesized/people.json`
   - Returns a brief summary of risk ratings

3. Wait for the agent to complete.

4. Verify output files exist and contain valid JSON:
   - `synthesized/risk-assessments.json`
   - `synthesized/people.json`

5. Update `run-metadata.json`: risk_assessment pass status = `"completed"`

---

## Pass 4: Report Generation

**Model:** sonnet (parent context)

1. Read all synthesized files:
   - `synthesized/scope-items-refined.json` (scope and sources)
   - `synthesized/risk-assessments.json` (risk ratings and details)
   - `synthesized/people.json` (people mapping)

2. Read the HTML template from `skills/project-insights/references/report-template.html`

3. Generate the complete HTML report by populating the template:

   **Executive Summary** (3-5 sentences):
   - Overall project health assessment
   - Count of scope items by risk rating
   - Key risks and blockers
   - Timeline assessment if target dates are available

   **Stated Scope** section:
   - Each scope item as a card with:
     - Progress bar (visual bar with percentage, calculated from ticket counts)
     - Risk badge (On Track / At Risk / Behind) with color coding
     - Source count (N Confluence pages, M Jira tickets, P Drive docs)
     - Target date if available
   - Border-left color matches risk rating

   **Risk Dashboard** section:
   - Summary table sorted by risk rating: Behind > At Risk > On Track
   - Columns: Scope Item, Risk Rating, Score, Progress, Stale, Blocked

   **Items Requiring Attention** section (priority ordered):
   - Blocked items first (highest urgency)
   - Stale items second
   - Untracked scope items third (no Jira tickets)

   **Key People** section:
   - Lean table showing ONLY people with risk-involved tickets
   - Do not list everyone -- only those with stale, blocked, or at-risk tickets
   - Columns: Name, Risk Tickets, Stale, Blocked, Scope Items

   **Recommendations** section:
   - Prioritized numbered list of specific, actionable steps
   - Examples: "Assign owner to PROJ-205 (blocked, no assignee)"
   - At least 3, up to 7 recommendations
   - Focus on unblocking, unstaling, and tracking gaps

4. Write the report as HTML to `<run-dir>/project-insights.html`

5. Inform the user where the report was saved and offer to open it.

6. Update `run-metadata.json`:
   - report_generation pass status = `"completed"`
   - overall status = `"completed"`
   - `completed_at` timestamp

---

## Risk Heuristic Decision Table

The risk-assessment agent uses weighted scoring. For quick human reference:

| Signal | Score Impact | Example |
|--------|-------------|---------|
| >=75% tickets done/in-progress | +0 | 9 of 12 tickets active |
| 50-74% done/in-progress | +10 | 7 of 12 tickets active |
| 25-49% done/in-progress | +25 | 4 of 12 tickets active |
| <25% done/in-progress | +40 | 2 of 12 tickets active |
| Each stale ticket | +5 | Ticket not updated in >14 days |
| Each stale + assigned + not started | +10 | Assigned but untouched >14 days |
| Each blocked ticket | +15 | Ticket has blocker link |
| Each blocked + unassigned | +25 | Blocked and no one owns it |
| Untracked scope (0 Jira tickets) | +50 | Confluence scope with no Jira |
| Behind schedule (>20% gap) | +20 | 50% done but 70% time elapsed |
| Critically behind (>40% gap) | +40 total | 30% done but 70% time elapsed |
| Target date passed | +30 | Deadline was last week |

**Final Rating:**
- 0-15 points: **ON TRACK** (green)
- 16-39 points: **AT RISK** (amber)
- 40+ points: **BEHIND** (red)

---

## Context Management Rules

- Agents write to files and return ONLY brief status messages (1-2 sentences)
- Parent reads from files, NOT from agent response content
- No single pass holds all project data in context simultaneously
- If context is getting large, summarize and refer to files
- Each pass's instructions reference specific file paths to read from and write to
- Sub-agents must be dispatched with `subagent_type: general-purpose` (not Bash) to use the Write tool

---

## Reference Documents

- **Synthesis algorithm details:** `skills/project-insights/references/synthesis-model.md`
- **Atlassian MCP tool names and parameters:** `skills/project-insights/references/mcp-tool-reference.md`
- **Google Drive MCP tool names and parameters:** `skills/project-insights/references/google-drive-mcp-reference.md`
- **HTML report template:** `skills/project-insights/references/report-template.html`
- **Example report output:** `skills/project-insights/examples/sample-report.html`
