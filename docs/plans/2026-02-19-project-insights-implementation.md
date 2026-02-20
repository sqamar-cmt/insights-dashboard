# Project Insights Skill — Implementation Plan (Revised)

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a reusable `/project-insights` Claude Code plugin that dispatches parallel agents to query Jira and Confluence via Atlassian MCP, synthesizes scope with many-to-many relationships, assesses risk, and generates executive-level HTML reports.

**Architecture:** Claude Code plugin with a slash command (`/project-insights`), a SKILL.md for orchestration logic, agent definitions for each pass, and reference docs for the synthesis model and report template. The 4-pass architecture (Discovery + Synthesis --> User Checkpoint --> Risk Assessment --> Report) is encoded in the skill's instructions. All agent outputs are persisted to local JSON files in a per-run directory (`reports/YYYY-MM-DD-PROJ-slug/`), preventing context saturation and enabling incremental re-runs. Cross-referencing between Jira and Confluence is handled in the synthesis step (Pass 1b), not by a dedicated agent.

**Tech Stack:** Claude Code plugin system (SKILL.md, commands/, agents/), Atlassian MCP (Jira + Confluence), JSON file-backed knowledge base, HTML report output.

**Design Doc:** `docs/plans/2026-02-19-project-insights-skill-design.md`

**Model Assignments:**

| Pass | Agent / Context | Model |
|------|-----------------|-------|
| Pass 1: Jira Discovery | `jira-discovery` agent | **haiku** |
| Pass 1: Confluence Discovery | `confluence-discovery` agent | **haiku** |
| Pass 1b: Synthesis | parent context | **opus** |
| Pass 3: Risk Assessment | `risk-assessment` agent | **sonnet** |
| Pass 4: Report Generation | parent context | **sonnet** |

---

## Task 0: MCP Capability Validation

**Purpose:** Before writing any agent code, confirm what Atlassian MCP tools actually exist, what parameters they accept, and what they return. This prevents building agents against assumed APIs.

**This task MUST complete before Tasks 4, 5, 6, and 8.**

**Step 1: Start a Claude Code session and check MCP status**

Run `/mcp` to see all configured MCP servers. Verify that the `atlassian` server appears and shows as connected.

**Step 2: Authenticate with Atlassian OAuth**

If the Atlassian MCP server requires OAuth, complete the browser-based authentication flow. Record:
- Whether authentication is per-session or persists across sessions
- What scopes were granted (read:jira-work, read:confluence-content:summary, etc.)
- Any token refresh behavior observed

**Step 3: List all available MCP tools**

After authentication, list every tool the Atlassian MCP server exposes. For each tool, record:
- Tool name (exact string used to invoke it)
- Parameters (name, type, required/optional, allowed values)
- Return type and structure (JSON schema or representative example)
- Any pagination parameters (startAt, maxResults, limit, start)
- Any rate limiting or error responses observed

**Step 4: Test a simple Jira query**

Run a basic Jira query to validate tool behavior:
- List all projects accessible to the authenticated user
- Query epics in a known project: attempt JQL `project = PROJ AND issuetype = Epic`
- Record the exact tool invocation syntax and full response shape
- Note which fields are returned by default vs. require explicit field selection
- Test what happens with an invalid project key (error format)

**Step 5: Test a simple Confluence query**

Run a basic Confluence query to validate tool behavior:
- List all spaces accessible to the authenticated user
- Search for pages in a known space: attempt CQL `space = SPACE AND type = page`
- Record the exact tool invocation syntax and full response shape
- Note whether page content (body) is returned or requires a separate fetch
- Test what happens with an invalid space key (error format)

**Step 6: Test sub-agent MCP access**

Dispatch a minimal sub-agent (using the Task tool) that attempts to call an Atlassian MCP tool. Verify:
- Sub-agents can access MCP tools (not just the parent context)
- The authentication context is inherited by sub-agents
- The response format is identical to parent-context calls

**Step 7: Document findings**

Write all findings to `skills/project-insights/references/mcp-tool-reference.md` with the following structure:

```markdown
# Atlassian MCP Tool Reference

## Authentication
- Auth method: [OAuth / API key / etc.]
- Persistence: [per-session / persisted]
- Required scopes: [list]

## Jira Tools

### [tool-name-1]
- **Description:** [what it does]
- **Parameters:**
  - `param1` (string, required): [description]
  - `param2` (number, optional, default: 50): [description]
- **Returns:** [JSON schema or example]
- **Pagination:** [how to paginate, if applicable]
- **Notes:** [any quirks, limitations, or gotchas]

### [tool-name-2]
...

## Confluence Tools

### [tool-name-1]
...

## Error Handling
- Invalid project/space: [error shape]
- Auth expired: [error shape]
- Rate limiting: [observed behavior]

## Sub-Agent Access
- Can sub-agents call MCP tools: [yes/no]
- Auth inheritance: [yes/no]
- Any differences from parent-context calls: [details]
```

**Step 8: Commit**

```bash
git add skills/project-insights/references/mcp-tool-reference.md
git commit -m "docs: add MCP tool reference from capability validation"
```

---

## Task 0b: Agent File Writing Test

**Purpose:** Verify that sub-agents dispatched via the Task tool can use the Write tool to create files on disk. If they cannot, the entire file-backed architecture must be redesigned so agents return JSON in their response body and the parent context writes files.

**This task MUST complete before Tasks 4, 5, 6, and 8.**

**Step 1: Dispatch a minimal test agent**

Create a test by dispatching a sub-agent (via Task tool) with these instructions:
- Create a directory `test-output/`
- Write a JSON file to `test-output/agent-write-test.json` containing:

```json
{
  "test": "agent-file-write",
  "timestamp": "[current ISO timestamp]",
  "success": true
}
```

- Return a status message: "Write test complete"

**Step 2: Verify the file was written**

After the agent completes, check:
- Does `test-output/agent-write-test.json` exist?
- Is the JSON valid and complete?
- Were the file permissions correct?

**Step 3: Determine architecture outcome**

**If agents CAN write files:**
- Proceed with the file-backed architecture as designed
- Agents write to `<run-dir>/raw/*.json` directly
- Parent reads from files after agents complete

**If agents CANNOT write files:**
- Redesign agent instructions: agents return structured JSON in their response body
- Parent context receives agent responses and writes all files
- Update all subsequent agent task descriptions to specify "return JSON" instead of "write to file"
- Add a note to SKILL.md explaining the write delegation pattern

**Step 4: Clean up and commit**

```bash
rm -rf test-output/
git add -A && git commit -m "test: validate sub-agent file writing capability"
```

---

## Task 1: Create Plugin Scaffold

**Files:**
- Create: `.claude-plugin/plugin.json`

**Step 1: Create the plugin manifest**

```json
{
  "name": "insights-dashboard",
  "description": "Project insights reporting: synthesize scope from Jira and Confluence, assess execution coverage, and generate executive-level risk reports",
  "version": "0.1.0",
  "keywords": ["insights", "reporting", "project-health", "jira", "confluence"]
}
```

**Step 2: Verify plugin is recognized**

Run: `ls -la .claude-plugin/plugin.json`
Expected: File exists with correct contents.

**Step 3: Commit**

```bash
git init && git add .claude-plugin/plugin.json docs/plans/
git commit -m "chore: initialize insights-dashboard plugin scaffold with design docs"
```

---

## Task 2: Create the `/project-insights` Slash Command

**Files:**
- Create: `commands/project-insights.md`

**Step 1: Write the command file**

```markdown
---
description: Generate executive-level project insights from Jira and Confluence
argument-hint: --jira PROJECT_KEY --spaces SPACE_KEY [--search "keywords"] [--target-date YYYY-MM-DD] [--stale-days N]
---

You are generating a project insights report. Use the project-insights skill for full guidance.

Parse the user's arguments:
- `--jira` (required): Comma-separated Jira project key(s)
- `--spaces` (required): Comma-separated Confluence space key(s)
- `--search` (optional): Free-text search keywords
- `--target-date` (optional): User-supplied deadline in YYYY-MM-DD format
- `--stale-days` (optional, default 14): Days of inactivity before flagging as stale

If required arguments are missing, ask the user for them before proceeding.

Then follow the project-insights skill's 4-pass process exactly.
```

**Step 2: Commit**

```bash
git add commands/project-insights.md
git commit -m "feat: add /project-insights slash command with argument parsing"
```

---

## Task 3: Create the MCP Configuration

**Files:**
- Create: `.mcp.json`

**Step 1: Write the MCP server config**

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

**Step 2: Commit**

```bash
git add .mcp.json
git commit -m "feat: add Atlassian MCP server configuration for Jira and Confluence"
```

---

## Task 4: Create the Jira Discovery Agent

**Model:** haiku

**Files:**
- Create: `agents/jira-discovery.md`

**Step 1: Write the agent definition**

The agent must:
- Accept project key(s), optional search keywords, and a **run directory path** as input
- Use Atlassian MCP tools to query Jira (use exact tool names from `references/mcp-tool-reference.md`)
- **Auto-discover issue types first:** Query the project's issue type scheme to find the actual names for epics, initiatives, and high-level types. Do NOT hardcode `"Epic"` -- the project may use `"Initiative"`, `"Feature"`, `"Theme"`, or custom types
- Search for epics, initiatives, and high-level stories using the discovered type names
- Extract: summary, description, status, assignee, fix version, sprint, due date, linked issues
- **Handle pagination:** Loop with `startAt` parameter, fetching 100 results per page, until total results are exhausted. Pseudocode:

```
startAt = 0
maxResults = 100
allResults = []
loop:
    response = jira_search(jql, startAt=startAt, maxResults=maxResults)
    allResults.append(response.issues)
    if startAt + maxResults >= response.total:
        break
    startAt += maxResults
```

- **Write structured findings to `<run-dir>/raw/jira-epics.json` and `<run-dir>/raw/jira-stories.json`**
- **Return only a brief status message** ("Found 12 epics, 47 stories in PROJ") -- NOT the full data

Write the agent with JQL query patterns (using discovered type names):

1. First, query project issue types:
   - Use the MCP tool to list issue types for the project
   - Identify which types represent high-level work items (epics, initiatives, features, themes)
   - Store the discovered type names for use in subsequent queries

2. Then, run queries using discovered types:
   - `project = PROJ AND issuetype = "<discovered-epic-type>" ORDER BY rank ASC`
   - `project = PROJ AND issuetype = "<discovered-initiative-type>" ORDER BY rank ASC` (if initiative type exists)
   - `project = PROJ AND labels in (roadmap, milestone, scope) ORDER BY rank ASC`
   - If search keywords provided: `project = PROJ AND text ~ "keywords" ORDER BY rank ASC`

3. For each epic/initiative found, query child issues:
   - `"Epic Link" = PROJ-101 ORDER BY rank ASC` or equivalent parent link query
   - Extract child issue status rollup (done/in-progress/todo counts)

The JSON schema for the top-level file (`raw/jira-epics.json`):

```json
{
  "project_key": "PROJ",
  "discovered_types": {
    "epic": "Epic",
    "initiative": "Initiative"
  },
  "fetched_at": "2026-02-19T14:30:00Z",
  "total_epics": 12,
  "epics": [
    {
      "key": "PROJ-101",
      "type": "Epic",
      "summary": "[title]",
      "description_snippet": "[first 200 chars]",
      "status": "[status]",
      "assignee": "[name]",
      "fix_version": "[version or null]",
      "due_date": "[date or null]",
      "sprint": "[sprint name or null]",
      "child_issues": {
        "total": 5,
        "done": 2,
        "in_progress": 2,
        "todo": 1
      },
      "linked_issues": ["PROJ-102", "PROJ-103"],
      "last_updated": "[ISO date]",
      "labels": ["roadmap", "q1"]
    }
  ]
}
```

The JSON schema for child issues (`raw/jira-stories.json`):

```json
{
  "project_key": "PROJ",
  "fetched_at": "2026-02-19T14:30:00Z",
  "total_stories": 47,
  "stories": [
    {
      "key": "PROJ-201",
      "type": "Story",
      "summary": "[title]",
      "status": "[status]",
      "assignee": "[name or null]",
      "parent_key": "PROJ-101",
      "sprint": "[sprint name or null]",
      "story_points": "[number or null]",
      "last_updated": "[ISO date]",
      "blockers": [
        {
          "key": "PROJ-150",
          "summary": "[blocker summary]",
          "status": "[blocker status]"
        }
      ]
    }
  ]
}
```

**Step 2: Commit**

```bash
git add agents/jira-discovery.md
git commit -m "feat: add Jira discovery agent with type auto-discovery and pagination"
```

---

## Task 5: Create the Confluence Discovery Agent

**Model:** haiku

**Files:**
- Create: `agents/confluence-discovery.md`

**Step 1: Write the agent definition**

The agent must:
- Accept space key(s), optional search keywords, and a **run directory path** as input
- Use Atlassian MCP tools to search Confluence (use exact tool names from `references/mcp-tool-reference.md`)
- Search for planning documents: PRDs, RFCs, tech specs, SOWs, decision logs, roadmaps
- Extract: page title, space, excerpt/content summary, labels, last modified date, author
- **Extract dates from page content** with confidence scoring:
  - Look for ISO format dates: `YYYY-MM-DD` (high confidence)
  - Look for US format dates: `MM/DD/YYYY`, `Month DD, YYYY` (medium confidence)
  - Look for EU format dates: `DD/MM/YYYY`, `DD Month YYYY` (medium confidence)
  - Look for relative dates: "end of Q1", "by March", "next sprint" (low confidence, convert to approximate ISO date)
  - For each date found, record the surrounding context (sentence or phrase) and a confidence score
  - Regex patterns to use:
    - ISO: `\d{4}-\d{2}-\d{2}`
    - US: `\d{1,2}/\d{1,2}/\d{4}` and `(?:January|February|March|April|May|June|July|August|September|October|November|December)\s+\d{1,2},?\s+\d{4}`
    - Relative: `(?:end of |by |before |after )?Q[1-4]\s*\d{4}`, `(?:end of |by |before |after )?(?:January|February|March|April|May|June|July|August|September|October|November|December)\s*\d{4}?`
- **Handle pagination:** Loop with `start` parameter, fetching 25 results per page, until all results are retrieved. Pseudocode:

```
start = 0
limit = 25
allResults = []
loop:
    response = confluence_search(cql, start=start, limit=limit)
    allResults.append(response.results)
    if start + limit >= response.totalSize:
        break
    start += limit
```

- **Write structured findings to `<run-dir>/raw/confluence-pages.json`**
- **Return only a brief status message** ("Found 8 planning docs in TEAMSPACE") -- NOT the full data

CQL query patterns:
- `space = TEAMSPACE AND type = page AND label in (prd, rfc, spec, sow, roadmap, decision, design, architecture, requirements) ORDER BY lastModified DESC`
- If search keywords provided: `space = TEAMSPACE AND type = page AND text ~ "keywords" ORDER BY lastModified DESC`
- Fallback broader search: `space = TEAMSPACE AND type = page AND title ~ "keywords" ORDER BY lastModified DESC`

The JSON schema for each page written to file (`raw/confluence-pages.json`):

```json
{
  "spaces_searched": ["TEAMSPACE"],
  "fetched_at": "2026-02-19T14:30:00Z",
  "total_pages": 8,
  "pages": [
    {
      "page_id": "[id]",
      "title": "[title]",
      "space": "[space key]",
      "url": "[page URL]",
      "labels": ["prd", "roadmap"],
      "author": "[name]",
      "last_modified": "[ISO date]",
      "content_summary": "[extracted scope items, deliverables, dates, milestones -- max 500 chars]",
      "dates_mentioned": [
        {
          "date": "2026-04-15",
          "context": "target launch date for pipeline v2",
          "format_detected": "iso",
          "confidence": "high"
        },
        {
          "date": "2026-03-31",
          "context": "end of Q1 milestone for schema validation",
          "format_detected": "relative",
          "confidence": "low"
        }
      ],
      "deliverables_mentioned": ["data pipeline v2", "schema validation", "monitoring dashboards"],
      "jira_keys_mentioned": ["PROJ-101", "PROJ-205"]
    }
  ]
}
```

**Step 2: Commit**

```bash
git add agents/confluence-discovery.md
git commit -m "feat: add Confluence discovery agent with pagination and date extraction"
```

---

## Task 6: Create the Risk Assessment Agent

**Model:** sonnet

**Files:**
- Create: `agents/risk-assessment.md`

**Step 1: Write the agent definition**

The agent operates as a **single agent** processing ALL scope items in one pass. It does NOT dispatch per-scope-item sub-agents.

The agent must:
- Accept the **run directory path** and the **stale-days threshold** as input
- Read `<run-dir>/synthesized/scope-items-refined.json` to get all scope items with their linked Jira tickets
- **Batch-query Jira** for ticket statuses: gather all unique Jira keys across all scope items and query them in batches (using JQL `key in (PROJ-101, PROJ-102, ...)` with up to 50 keys per query) to minimize MCP calls
- For each scope item, calculate aggregate metrics:
  - Total tickets, done count, in-progress count, not-started count
  - Stale count (no updates in >N days, where N is the stale-days parameter)
  - Blocked count (tickets with blocker links or "Blocked" status)
- Apply **weighted risk scoring** (not binary thresholds):

```
Base score: 0 (lower is better)

Progress scoring:
  + 0  if >=75% done/in-progress
  + 10 if 50-74% done/in-progress
  + 25 if 25-49% done/in-progress
  + 40 if <25% done/in-progress

Staleness scoring:
  + 5  per stale ticket (>N days inactive)
  + 10 per stale ticket that is assigned but not started

Blocker scoring:
  + 15 per blocked ticket
  + 25 per blocked ticket with no assignee

Tracking scoring:
  + 50 if scope item has 0 Jira tickets (untracked)

Time scoring (if target date available):
  time_ratio = elapsed_days / total_days
  progress_ratio = done_tickets / total_tickets
  if time_ratio > progress_ratio + 0.2:
    + 20 (significantly behind schedule)
  if time_ratio > progress_ratio + 0.4:
    + 20 more (critically behind schedule)
  if target_date has passed:
    + 30

Final rating:
  0-15:  "on_track"
  16-39: "at_risk"
  40+:   "behind"
```

- Build a people mapping: for each person found across all scope items, aggregate their ticket count and risk-ticket count
- **Write results to `<run-dir>/synthesized/risk-assessments.json` and `<run-dir>/synthesized/people.json`**
- **Return only a brief status summary** listing each scope item and its risk rating

The JSON schema for risk assessments (`synthesized/risk-assessments.json`):

```json
{
  "assessed_at": "2026-02-19T14:45:00Z",
  "stale_days_threshold": 14,
  "assessments": [
    {
      "scope_item": "[name]",
      "risk_rating": "at_risk",
      "risk_score": 27,
      "risk_reasons": [
        "2 tickets stale >14 days (+10)",
        "1 blocker unresolved (+15)",
        "Progress ratio 50% vs time ratio 70% (+0, within tolerance)"
      ],
      "progress": {
        "total": 12,
        "done": 6,
        "in_progress": 3,
        "not_started": 2,
        "stale": 2,
        "blocked": 1
      },
      "stale_items": [
        {
          "key": "PROJ-310",
          "summary": "[ticket title]",
          "days_since_update": 22,
          "assignee": "Bob",
          "status": "In Progress"
        }
      ],
      "blocked_items": [
        {
          "key": "PROJ-205",
          "summary": "[ticket title]",
          "blocker_reason": "Waiting on infra team",
          "blocker_key": "INFRA-88",
          "assignee": "Carol"
        }
      ],
      "target_date": {
        "date": "2026-04-15",
        "source": "confluence",
        "confidence": "high"
      },
      "time_assessment": {
        "pct_complete": 50,
        "pct_time_elapsed": 70,
        "days_remaining": 55,
        "total_days": 180
      }
    }
  ]
}
```

The JSON schema for people mapping (`synthesized/people.json`):

```json
{
  "assessed_at": "2026-02-19T14:45:00Z",
  "people": [
    {
      "name": "Alice",
      "total_tickets": 4,
      "risk_tickets": 0,
      "stale_tickets": 0,
      "blocked_tickets": 0,
      "scope_items": ["Data Pipeline Migration", "Schema Validation"]
    },
    {
      "name": "Bob",
      "total_tickets": 3,
      "risk_tickets": 2,
      "stale_tickets": 2,
      "blocked_tickets": 0,
      "scope_items": ["Data Pipeline Migration"]
    }
  ]
}
```

**Step 2: Commit**

```bash
git add agents/risk-assessment.md
git commit -m "feat: add risk assessment agent with weighted scoring and batch queries"
```

---

## Task 7: Create Reference -- Synthesis Model

**Files:**
- Create: `skills/project-insights/references/synthesis-model.md`

**Step 1: Write the synthesis model reference**

This is the most critical reference document. It drives the Pass 1b synthesis step, which is the intellectual core of the skill. The document must contain:

**Section 1: Purpose and Context**

Explain that synthesis takes raw Jira and Confluence data and produces logical scope items -- the unit of analysis for the rest of the pipeline. Each scope item represents a coherent capability, deliverable, or workstream.

**Section 2: The 4-Phase Synthesis Algorithm**

Phase 1 -- Cross-Reference Extraction:
- Scan all `raw/confluence-pages.json` entries for Jira key mentions (`jira_keys_mentioned` field)
- Scan all `raw/jira-epics.json` entries for Confluence page URLs or titles in descriptions/comments
- Build a cross-reference map: `{jira_key: [confluence_page_ids], confluence_page_id: [jira_keys]}`
- Scan `raw/jira-stories.json` for parent epic keys to build the epic-to-story hierarchy
- Record all cross-references for audit trail

Phase 2 -- Initial Clustering:
- **Epic-anchored clusters:** Start with each Jira epic as a seed. Attach its child stories, any Confluence pages that mention the epic key, and any Confluence pages whose `deliverables_mentioned` overlap with the epic summary
- **Confluence-anchored clusters:** For Confluence pages not yet assigned to an epic-anchored cluster, check if they describe a distinct deliverable. If multiple pages reference the same deliverable (by keyword overlap in `deliverables_mentioned`), group them together
- **Keyword-overlap clusters:** For remaining unmatched items, use keyword overlap between Jira ticket summaries and Confluence page titles/deliverables. Match if 2+ significant keywords overlap (exclude stop words and common project terms)
- **Orphans:** Items that do not cluster with anything else become single-item scope items. These are flagged for user attention at the checkpoint

Phase 3 -- Confidence Scoring:
- Each scope item receives a synthesis confidence score:
  - **High (0.8-1.0):** Explicit cross-references exist (Jira key in Confluence page AND Confluence URL in Jira ticket), OR epic has clear child stories mapping to a described deliverable
  - **Medium (0.5-0.79):** Keyword overlap match with 3+ matching terms, OR epic-anchored cluster with Confluence pages matched by deliverable keywords but no explicit cross-references
  - **Low (0.2-0.49):** Single keyword match, OR clustering based solely on label/sprint co-occurrence, OR orphan items grouped by weak heuristic
- Confidence score informs the checkpoint presentation: low-confidence items are highlighted for user review

Phase 4 -- Scope Item Naming:
- Use the most descriptive human-readable name from available sources
- Priority order: (1) Confluence PRD/RFC title if it describes a specific deliverable, (2) Jira epic summary if concise, (3) synthesized name from overlapping keywords
- Names should be 3-8 words, action-oriented where possible (e.g., "Data Pipeline V2 Migration" not "PROJ-101")
- Avoid Jira keys in scope item names (they go in the `jira_sources` field)

**Section 3: Complete JSON Schema for `scope-items.json`**

```json
{
  "synthesized_at": "2026-02-19T14:35:00Z",
  "project_keys": ["PROJ"],
  "confluence_spaces": ["TEAMSPACE"],
  "search_terms": ["data pipeline"],
  "total_scope_items": 6,
  "scope_items": [
    {
      "id": "scope-001",
      "name": "Data Pipeline V2 Migration",
      "description": "End-to-end migration of the legacy data pipeline to v2 architecture, including schema validation, monitoring, and rollback procedures.",
      "synthesis_confidence": 0.92,
      "synthesis_method": "epic-anchored",
      "confluence_sources": [
        {
          "page_id": "12345",
          "title": "Data Pipeline V2 - PRD",
          "space": "TEAMSPACE",
          "url": "https://company.atlassian.net/wiki/spaces/TEAMSPACE/pages/12345",
          "match_type": "explicit_reference",
          "dates_mentioned": [
            {
              "date": "2026-04-15",
              "context": "target launch date",
              "confidence": "high"
            }
          ],
          "deliverables_mentioned": ["pipeline v2", "schema validation", "monitoring dashboards"]
        }
      ],
      "jira_sources": [
        {
          "key": "PROJ-101",
          "type": "Epic",
          "summary": "Migrate data pipeline to v2",
          "status": "In Progress",
          "match_type": "explicit_reference",
          "child_count": 8
        },
        {
          "key": "PROJ-310",
          "type": "Story",
          "summary": "Add monitoring dashboard for v2 pipeline",
          "status": "To Do",
          "match_type": "keyword_overlap",
          "child_count": 0
        }
      ],
      "target_dates": [
        {
          "date": "2026-04-15",
          "source": "confluence",
          "page_title": "Data Pipeline V2 - PRD",
          "confidence": "high"
        }
      ],
      "tags": ["migration", "infrastructure"],
      "notes": ""
    },
    {
      "id": "scope-002",
      "name": "Authentication Service Rewrite",
      "description": "Scope item discovered from Jira only -- no corresponding Confluence planning document found.",
      "synthesis_confidence": 0.45,
      "synthesis_method": "epic-anchored",
      "confluence_sources": [],
      "jira_sources": [
        {
          "key": "PROJ-150",
          "type": "Epic",
          "summary": "Rewrite auth service",
          "status": "To Do",
          "match_type": "direct_epic",
          "child_count": 3
        }
      ],
      "target_dates": [],
      "tags": ["authentication"],
      "notes": "No Confluence documentation found. Flagged for user review."
    }
  ],
  "orphans": {
    "unmatched_jira": [
      {
        "key": "PROJ-400",
        "summary": "Investigate caching layer performance",
        "reason": "No matching Confluence content or epic grouping found"
      }
    ],
    "unmatched_confluence": [
      {
        "page_id": "67890",
        "title": "Q2 Platform Scalability Plan",
        "reason": "No matching Jira epics or stories found"
      }
    ]
  },
  "cross_references": {
    "explicit": [
      {
        "jira_key": "PROJ-101",
        "confluence_page_id": "12345",
        "direction": "both",
        "evidence": "PROJ-101 description links to page 12345; page 12345 mentions PROJ-101"
      }
    ],
    "inferred": [
      {
        "jira_key": "PROJ-310",
        "confluence_page_id": "12345",
        "direction": "keyword",
        "evidence": "Keywords overlap: 'monitoring', 'dashboard', 'pipeline'"
      }
    ]
  }
}
```

**Section 4: Worked Examples**

Include at least 3 worked examples:

Example 1 -- Clean epic-anchored match:
- Input: Epic PROJ-101 "Migrate data pipeline to v2" with 8 child stories. Confluence page "Data Pipeline V2 - PRD" in TEAMSPACE mentions PROJ-101 and lists deliverables.
- Process: Phase 1 finds explicit cross-reference. Phase 2 creates epic-anchored cluster. Phase 3 scores 0.92 (explicit bidirectional references). Phase 4 names it "Data Pipeline V2 Migration" from the PRD title.
- Output: Single scope item with 1 confluence source, 1 epic + child stories as jira sources.

Example 2 -- Keyword-overlap match with multiple sources:
- Input: No explicit cross-references. Confluence pages "Monitoring Strategy" and "Observability RFC" both mention "dashboards", "alerts", "SLOs". Jira stories PROJ-310, PROJ-311, PROJ-312 have summaries about dashboards and alerting but belong to different epics.
- Process: Phase 1 finds no explicit links. Phase 2 fails to anchor to a single epic. Phase 2 keyword-overlap step groups the Confluence pages together. Phase 2 also finds the Jira stories by keyword overlap. Phase 3 scores 0.55 (keyword overlap only, no explicit references). Phase 4 names it "Monitoring and Observability" from the RFC title.
- Output: Scope item with medium confidence, flagged for user review.

Example 3 -- Orphan items:
- Input: Jira epic PROJ-400 "Investigate caching layer performance" has 2 child stories. No Confluence pages mention caching, performance, or PROJ-400.
- Process: Phase 1 finds no cross-references. Phase 2 creates an epic-anchored cluster but with no Confluence sources. Phase 3 scores 0.45 (single-source, no corroborating evidence). Phase 4 names it from the epic summary. Marked in orphans for user attention.
- Output: Scope item with low confidence and a note about missing documentation.

**Section 5: Edge Cases**

- Duplicate detection: If two epics describe the same deliverable (e.g., one in PROJ, one in ENG), merge them into a single scope item with both as jira_sources
- Overly broad matches: If a Confluence page mentions 10+ Jira keys, it is likely a release notes or changelog page -- deprioritize it as a scope source and note it as "reference" not "planning"
- Empty results: If no Confluence pages are found, all scope items come from Jira only (epic-anchored) with low confidence
- No epics: If the Jira project uses flat stories without epics, cluster stories by label, sprint, fix version, or component

**Step 2: Commit**

```bash
git add skills/project-insights/references/synthesis-model.md
git commit -m "docs: add synthesis model reference with 4-phase algorithm and worked examples"
```

---

## Task 8: Create the Main SKILL.md

**Model for synthesis (Pass 1b):** opus
**Model for report generation (Pass 4):** sonnet

**Files:**
- Create: `skills/project-insights/SKILL.md`

**Step 1: Write the skill definition**

This is the core orchestration file. It must contain all of the following sections:

**Frontmatter:**

```yaml
---
name: project-insights
description: >
  Generate executive-level project insights from Jira and Confluence.
  Trigger phrases: project insights, project health, risk report, scope assessment
version: 0.1.0
---
```

**Overview:**

Describe the 4-pass file-backed architecture:
- Pass 1: Discovery (parallel haiku agents) + Synthesis (opus, parent context)
- Checkpoint: User reviews scope items
- Pass 2: Refined Discovery (if user requests changes)
- Pass 3: Risk Assessment (single sonnet agent)
- Pass 4: Report Generation (sonnet, parent context)

All agent outputs persist to local JSON files. No agent needs to hold all findings in context. Passes are decoupled -- each reads from the previous pass's files.

**MCP Authentication Check:**

At the very start of execution, before any pass, verify MCP connectivity:

```
1. Check that the Atlassian MCP server is connected (run /mcp or equivalent)
2. Attempt a minimal Jira query (e.g., list projects) to verify auth is valid
3. If auth fails, instruct the user to run /mcp and complete the OAuth flow
4. Do NOT proceed to Pass 1 until auth is confirmed
5. Record auth status in run-metadata.json
```

**Run Directory Setup:**

How to create the `reports/YYYY-MM-DD-PROJ-slug/` directory structure:

```
1. Build directory name: YYYY-MM-DD-<first-jira-key>-<first-search-keyword-kebab-cased>
   - If no search keywords, use "general" as the slug
   - If directory already exists, append -2, -3, etc.
2. Create subdirectories: raw/, synthesized/
3. Write run-metadata.json with:
```

```json
{
  "created_at": "2026-02-19T14:00:00Z",
  "status": "in_progress",
  "current_pass": "setup",
  "parameters": {
    "jira_projects": ["PROJ"],
    "confluence_spaces": ["TEAMSPACE"],
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
    "verified": true,
    "verified_at": "2026-02-19T14:00:05Z"
  }
}
```

**Error Handling and Resume:**

Include explicit error handling instructions:

```
Error Handling:
- Before each pass, update run-metadata.json: set the pass status to "in_progress" and record started_at
- After each pass completes, update run-metadata.json: set the pass status to "completed" and record completed_at
- If any pass fails, set run-metadata.json status to "error" with an error_message field
- Use atomic writes: write to a .tmp file first, then rename to the final filename
- On --resume: read run-metadata.json, skip passes with status "completed", restart from the first non-completed pass

Atomic write pattern:
1. Write content to <filename>.tmp
2. Read back <filename>.tmp to verify it is valid JSON
3. Rename <filename>.tmp to <filename>
4. If rename fails, log error and do not delete the .tmp file
```

**Pass 1 Instructions -- Discovery:**

```
Pass 1: Discovery

Model: haiku (for both agents)

1. Dispatch jira-discovery and confluence-discovery agents IN PARALLEL
   - Pass each agent: the relevant project key(s) or space key(s), search keywords, and the run directory path
   - Each agent writes its findings to raw/*.json files
   - Each agent returns only a brief status message

2. Wait for both agents to complete

3. Verify output files exist:
   - raw/jira-epics.json must exist and contain valid JSON
   - raw/jira-stories.json must exist and contain valid JSON
   - raw/confluence-pages.json must exist and contain valid JSON
   - If any file is missing or invalid, report the error and stop

4. Update run-metadata.json: discovery pass status = "completed"
```

**Pass 1b Instructions -- Synthesis:**

```
Pass 1b: Synthesis

Model: opus (parent context)

This is the intellectual core of the skill. Follow the 4-phase algorithm described
in references/synthesis-model.md EXACTLY.

1. Read all raw/*.json files into context
2. Execute the synthesis algorithm:
   a. Cross-Reference Extraction: Build the cross-reference map between Jira keys and Confluence pages
   b. Initial Clustering: Create epic-anchored, confluence-anchored, keyword-overlap, and orphan clusters
   c. Confidence Scoring: Score each cluster (high 0.8-1.0, medium 0.5-0.79, low 0.2-0.49)
   d. Scope Item Naming: Name each scope item using the priority hierarchy from synthesis-model.md
3. Write synthesized/scope-items.json following the exact schema in synthesis-model.md
4. Include orphans (unmatched Jira and Confluence items) in the output for user review
5. Include the cross_references section for audit trail

Update run-metadata.json: synthesis pass status = "completed"
```

**Checkpoint Instructions:**

```
Checkpoint: User Refinement

Read synthesized/scope-items.json and present to user.

Display format:
- List each scope item with: name, confidence score, source count (N Confluence pages, M Jira tickets)
- Flag low-confidence items with [NEEDS REVIEW]
- Flag orphans separately under "Unmatched Items"
- Show target dates if available

Ask the user:
"I found [N] scope items from [X] Jira tickets and [Y] Confluence pages.
[List scope items with confidence indicators]

Please review and let me know:
1. Are any scope items incorrect or should be removed?
2. Are there missing scope items I should add?
3. Should any items be merged or split?
4. Any additional context (dates, priorities, dependencies)?
5. Should I search deeper on any specific topic?"

Handle user responses:
- Removals: Mark scope items as excluded in scope-items.json
- Additions: Note user-added items for Pass 2 deeper search
- Merges: Combine scope items, union their sources
- Splits: Create new scope items with subsets of sources
- Context: Add user-provided dates/priorities to the relevant scope items

Update run-metadata.json: checkpoint pass status = "completed"
```

**Pass 2 Instructions -- Refined Discovery:**

```
Pass 2: Refined Discovery (if needed)

Only execute if the user added new items, requested deeper search, or provided
information that requires additional Jira/Confluence queries.

1. For user-added scope items without system links:
   - Search Jira for matching epics/stories by keyword
   - Search Confluence for matching pages by keyword
   - Add any matches to the scope item's sources

2. For deeper search requests:
   - Run additional JQL/CQL queries as specified by user
   - Append new findings to raw/*.json files

3. Write synthesized/scope-items-refined.json:
   - Start from scope-items.json
   - Apply all user modifications (removals, additions, merges, splits)
   - Incorporate any new discoveries from step 1-2
   - Recalculate confidence scores where sources changed

If no refinement is needed, copy scope-items.json to scope-items-refined.json unchanged.

Update run-metadata.json: refined_discovery pass status = "completed"
```

**Pass 3 Instructions -- Risk Assessment:**

```
Pass 3: Risk Assessment

Model: sonnet

Dispatch a SINGLE risk-assessment agent (NOT one per scope item).

1. Dispatch the risk-assessment agent with:
   - Run directory path
   - Stale-days threshold
2. The agent reads scope-items-refined.json
3. The agent batch-queries Jira for current ticket statuses
4. The agent calculates weighted risk scores for each scope item
5. The agent writes synthesized/risk-assessments.json and synthesized/people.json
6. The agent returns a brief summary of risk ratings

Wait for the agent to complete.

Verify output files exist and contain valid JSON.

Update run-metadata.json: risk_assessment pass status = "completed"
```

**Pass 4 Instructions -- Report Generation:**

```
Pass 4: Report Generation

Model: sonnet (parent context)

1. Read all synthesized/*.json files:
   - scope-items-refined.json (scope and sources)
   - risk-assessments.json (risk ratings and details)
   - people.json (people mapping)

2. Read the HTML template from skills/project-insights/references/report-template.html

3. Generate the complete HTML report by populating the template:
   - Executive Summary: 3-5 sentences, overall project health, key risks
   - Stated Scope: Each scope item as a card with progress bar and risk badge
   - Risk Dashboard: Summary table sorted by risk rating (Behind > At Risk > On Track)
   - Items Requiring Attention: Blocked items first, then stale, then untracked
   - Key People: Lean table showing only people with risk-involved tickets
   - Recommendations: Prioritized numbered list of specific, actionable steps

4. Write the report as HTML to <run-dir>/project-insights.html

5. Update run-metadata.json:
   - report_generation pass status = "completed"
   - overall status = "completed"
   - completed_at timestamp
```

**Risk Heuristic Decision Table (inline):**

Include this table directly in the SKILL.md so agents and the parent context have quick reference without reading a separate file:

```
Risk Heuristic Decision Table

The risk-assessment agent uses weighted scoring. For quick human reference:

| Signal | Score Impact | Example |
|--------|-------------|---------|
| >=75% tickets done/in-progress | +0 | 9 of 12 tickets in progress or done |
| 50-74% done/in-progress | +10 | 7 of 12 tickets in progress or done |
| 25-49% done/in-progress | +25 | 4 of 12 tickets in progress or done |
| <25% done/in-progress | +40 | 2 of 12 tickets in progress or done |
| Each stale ticket | +5 | Ticket not updated in >14 days |
| Each stale + assigned + not started | +10 | Assigned but untouched for >14 days |
| Each blocked ticket | +15 | Ticket has blocker link |
| Each blocked + unassigned | +25 | Blocked and no one owns it |
| Untracked scope (0 Jira tickets) | +50 | Confluence scope with no Jira tracking |
| Behind schedule (>20% gap) | +20 | 50% done but 70% of time elapsed |
| Critically behind (>40% gap) | +40 total | 30% done but 70% of time elapsed |
| Target date passed | +30 | Deadline was last week |

Final Rating:
  0-15 points:  ON TRACK (green)
  16-39 points: AT RISK (amber)
  40+ points:   BEHIND (red)
```

**Context Management Rules:**

```
Context Management:
- Agents write to files and return ONLY brief status messages (1-2 sentences)
- Parent reads from files, NOT from agent response content
- No single pass holds all project data in context simultaneously
- If context is getting large, summarize and refer to files
- Each pass's instructions reference specific file paths to read from and write to
```

**Reference Doc Pointers:**

```
Reference Documents:
- Synthesis algorithm details: skills/project-insights/references/synthesis-model.md
- MCP tool names and parameters: skills/project-insights/references/mcp-tool-reference.md
- HTML report template: skills/project-insights/references/report-template.html
- Example report output: skills/project-insights/examples/sample-report.html
```

The skill should target approximately 2,500-3,000 words. Detailed methodology is in the reference docs.

**Step 2: Commit**

```bash
git add skills/project-insights/SKILL.md
git commit -m "feat: add project-insights skill with 4-pass orchestration and inline risk heuristics"
```

---

## Task 9: Create the HTML Report Template

**Files:**
- Create: `skills/project-insights/references/report-template.html`

**Step 1: Write the self-contained HTML/CSS template**

A complete, self-contained HTML file with:

- **Google Fonts imports** (in `<head>`):
  - Roboto (400, 500) -- body text
  - Roboto Condensed (700) -- headings and titles
  - Roboto Mono (400) -- code, numbers, Jira keys, references

- **Embedded CSS** with:
  - Typography rules mapping each font to its elements
  - `body` uses `font-family: 'Roboto', sans-serif`
  - `h1, h2, h3, h4, th, .risk-badge` uses `font-family: 'Roboto Condensed', sans-serif`
  - `code, .jira-key, .date, .pct, .url` uses `font-family: 'Roboto Mono', monospace`
  - Risk badge color coding: On Track (#1B8A2D), At Risk (#D97706), Behind (#DC2626)
  - Progress bar styling (visual bar with percentage)
  - Scope item cards with border-left color matching risk rating
  - Clean table styling for Risk Dashboard and People sections
  - Print-friendly `@media print` rules
  - Responsive layout for screen viewing

- **HTML structure** with placeholder comments showing where data is inserted:
  - `<header>` with project name, generation date, source metadata
  - `<section id="executive-summary">` with overall health badge
  - `<section id="stated-scope">` with scope item cards
  - `<section id="risk-dashboard">` with summary table
  - `<section id="attention-required">` with blocked/stale/untracked subsections
  - `<section id="key-people">` with lean involvement table
  - `<section id="recommendations">` with prioritized action list

- **Guidance comments** in the HTML explaining:
  - How to populate each section from the synthesized JSON files
  - Executive Summary tone and length (3-5 sentences)
  - Priority ordering for Items Requiring Attention (blocked > stale > untracked)
  - People section: risk-involvement only, not full activity
  - How to map risk ratings to CSS classes for color coding
  - How to calculate progress bar widths from ticket counts

**Step 2: Commit**

```bash
git add skills/project-insights/references/report-template.html
git commit -m "feat: add styled HTML report template with Roboto typography"
```

---

## Task 10: Create Example Report

**Files:**
- Create: `skills/project-insights/examples/sample-report.html`

**Step 1: Write a realistic example report**

A complete, realistic HTML report for a fictional project using the report-template.html, showing:
- Full styled output with Roboto / Roboto Condensed / Roboto Mono typography
- A fictional project "ACME Data Platform Modernization" with Jira key ACME and Confluence space ACMEENG
- At least 5 scope items with varying risk levels:
  - 1-2 "On Track" items with green badges and >75% progress
  - 1-2 "At Risk" items with amber badges, stale tickets, or minor schedule slippage
  - 1-2 "Behind" items with red badges, blockers, untracked scope, or missed deadlines
- Many-to-many relationships visible (some scope items tracked by 3+ tickets, some referencing 2+ Confluence pages, one untracked)
- Progress bars showing visual completion status
- Blocked and stale items with real-looking details (ticket keys, assignees, days stale)
- People section showing 4-6 people with risk involvement
- At least 5 concrete, actionable recommendations
- Jira keys, dates, and percentages rendered in Roboto Mono
- Realistic target dates relative to the generation date

This serves as a visual reference for what the output looks like and as a test that the template renders correctly in a browser.

**Step 2: Commit**

```bash
git add skills/project-insights/examples/sample-report.html
git commit -m "docs: add sample styled HTML executive report example"
```

---

## Task 11: Create CLAUDE.md Project Context

**Files:**
- Create: `CLAUDE.md`

**Step 1: Write project context file**

The CLAUDE.md must include:

**Project Purpose:**
- Claude Code plugin for project insights reporting
- Synthesizes scope from Jira and Confluence, assesses risk, generates executive HTML reports

**Plugin Structure:**
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

**Model Specifications:**

```
Model assignments for agents and passes:
- jira-discovery agent:        haiku (fast, structured extraction)
- confluence-discovery agent:   haiku (fast, structured extraction)
- risk-assessment agent:        sonnet (reasoning about risk scoring)
- Pass 1b synthesis:            opus (complex cross-referencing and clustering)
- Pass 4 report generation:     sonnet (narrative writing with structure)
```

**MCP Authentication:**

```
This plugin requires the Atlassian MCP server for Jira and Confluence access.

To authenticate:
1. Run /mcp in a Claude Code session
2. The Atlassian server should appear in the list
3. If not authenticated, the OAuth flow will launch in your browser
4. Grant access to the required Jira and Confluence scopes
5. Return to Claude Code -- the session should now have MCP access

The MCP server is configured in .mcp.json with HTTP transport to https://mcp.atlassian.com/v1/mcp

Actual MCP tool names and parameters are documented in:
  skills/project-insights/references/mcp-tool-reference.md
```

**How to Test:**

```
Quick test:
  /project-insights --jira PROJ --spaces TEAMSPACE --search "your feature"

The skill will:
1. Verify MCP authentication
2. Run parallel discovery agents against Jira and Confluence
3. Synthesize findings into scope items
4. Present scope items for your review (checkpoint)
5. Assess risk for each scope item
6. Generate an HTML report in reports/YYYY-MM-DD-PROJ-slug/
```

**Key Design Decisions:**
- File-backed architecture: agents write to JSON files, not context, to prevent context saturation
- Many-to-many relationships: scope items can reference multiple Jira tickets AND multiple Confluence pages
- Weighted risk scoring: numeric scores (0-100+) mapped to three ratings (on_track/at_risk/behind)
- Single risk agent: one agent processes all scope items in one pass with batch Jira queries
- HTML output: self-contained HTML with embedded CSS, not Markdown

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md with MCP auth instructions and model specifications"
```

---

## Task 12: Integration Test -- Verify Plugin Loading

**Step 1: Check plugin recognition**

Start a new Claude Code session in the project directory and verify:
- Run `/mcp` to check Atlassian MCP is available and authenticated
- Check that `/project-insights` appears as an available command
- Verify the project-insights skill loads when triggered

**Step 2: Verify MCP authentication**

- Confirm the Atlassian MCP server shows as "connected" in `/mcp` output
- Run a minimal Jira query through MCP to confirm auth is valid (e.g., list accessible projects)
- Run a minimal Confluence query through MCP to confirm auth is valid (e.g., list accessible spaces)
- If auth fails, complete the OAuth flow and retry

**Step 3: Verify agent dispatching**

- Trigger the skill with a simple invocation
- Confirm that sub-agents are dispatched (check that raw/*.json files are created)
- Confirm that sub-agents can access MCP tools (check that files contain real data, not errors)

**Step 4: Fix any loading issues**

If the plugin is not recognized, check:
- `.claude-plugin/plugin.json` exists and is valid JSON
- Directory structure matches auto-discovery conventions
- File names are kebab-case
- MCP configuration in `.mcp.json` is valid

**Step 5: Commit any fixes**

```bash
git add -A && git commit -m "fix: resolve plugin loading issues"
```

---

## Task 13: End-to-End Test -- Run Against Real Project

**Step 1: Authenticate with Atlassian**

Run `/mcp` in Claude Code session and complete the OAuth flow for the Atlassian server. Verify authentication by running a test query.

**Step 2: Run the skill**

```
/project-insights --jira [REAL_PROJECT_KEY] --spaces [REAL_SPACE_KEY] --search "relevant keywords"
```

**Step 3: Verify each pass executes**

- Pass 1: Discovery agents find Jira tickets and Confluence pages; raw/*.json files created
- Pass 1b: Synthesis produces scope-items.json with cross-references and confidence scores
- Checkpoint: Scope items are presented for user review with confidence indicators
- Pass 2: Any user refinements are incorporated into scope-items-refined.json
- Pass 3: Single risk assessment agent processes all items; risk-assessments.json and people.json created
- Pass 4: HTML report is generated and saved to `<run-dir>/project-insights.html`

**Step 4: Verify run-metadata.json tracking**

Check that run-metadata.json has:
- All pass statuses set to "completed"
- Timestamps recorded for each pass
- Overall status set to "completed"

**Step 5: Review report quality**

Check the generated HTML report for:
- Opens correctly in a browser with proper styling
- Scope items correctly synthesized with many-to-many relationships
- Risk ratings make sense given the data (check against weighted scoring)
- Stale/blocked items correctly identified with correct day counts
- People section is lean and risk-focused (not listing everyone)
- Recommendations are actionable and specific
- Fonts render correctly (Roboto, Roboto Condensed, Roboto Mono)
- Risk badges have correct colors (green/amber/red)
- Progress bars render with correct widths

**Step 6: Commit**

```bash
git add reports/
git commit -m "test: add first generated project insights report"
```

---

## Task 14: Edge Case Testing

**Purpose:** Validate the skill handles non-standard and boundary conditions gracefully.

**Scenario 1: Empty results**

Run the skill against a Jira project with no epics and a Confluence space with no planning documents:
- Verify agents handle empty results without crashing
- Verify synthesis produces an empty scope-items.json (not an error)
- Verify the checkpoint informs the user "No scope items found" and asks for guidance
- Verify the risk assessment handles zero scope items
- Verify the report generates with an "Insufficient Data" message rather than empty sections

Test commands:
```
/project-insights --jira EMPTYPROJ --spaces EMPTYSPACE --search "nonexistent feature"
```

Expected behavior:
- raw/jira-epics.json contains `{"epics": [], "total_epics": 0}`
- raw/confluence-pages.json contains `{"pages": [], "total_pages": 0}`
- Checkpoint says "No scope items discovered. Would you like to add items manually or adjust search parameters?"

**Scenario 2: Large project with pagination**

Run the skill against a Jira project with 100+ epics and a Confluence space with 50+ pages:
- Verify Jira agent paginates correctly (startAt increments by 100)
- Verify Confluence agent paginates correctly (start increments by 25)
- Verify all results are captured (compare total count in JSON against MCP total)
- Verify synthesis handles a large number of raw items without context overflow
- Verify the risk assessment agent handles batch queries with 50+ unique Jira keys

**Scenario 3: Non-standard Jira configuration**

Test against a Jira project that uses non-standard issue types:
- Project uses "Feature" instead of "Epic"
- Project uses "Initiative" as the top-level type
- Project has custom issue types (e.g., "Platform Request")
- Verify the agent's issue type auto-discovery (Task 4) correctly identifies the project's hierarchy
- Verify JQL queries use the discovered type names, not hardcoded "Epic"

**Scenario 4: Missing target dates**

Run the skill without `--target-date` and against items with no dates in Jira or Confluence:
- Verify risk scoring works without time-based assessment (time scoring portion = 0)
- Verify the report clearly states "No target date available" rather than showing incorrect dates

**Scenario 5: MCP authentication expiration**

Test behavior when MCP auth expires mid-run:
- Verify the skill detects the auth failure
- Verify run-metadata.json records the error
- Verify the user is prompted to re-authenticate
- Verify `--resume` can continue from where the run left off after re-authentication

**Scenario 6: Duplicate content across spaces**

Run with multiple Confluence spaces that contain overlapping content:
- Verify synthesis deduplicates pages that appear in multiple search results
- Verify scope items are not duplicated when the same deliverable is described in two spaces

**Step: Document results and commit**

Record the results of each scenario test. Fix any issues discovered. Commit all fixes:

```bash
git add -A && git commit -m "fix: resolve edge case issues from comprehensive testing"
```

---

## Dependency Graph

```
Task 0  (MCP Capability Validation)
Task 0b (Agent File Writing Test)
    │
    ├─── unblocks ──→ Task 4 (Jira Discovery Agent)
    ├─── unblocks ──→ Task 5 (Confluence Discovery Agent)
    ├─── unblocks ──→ Task 6 (Risk Assessment Agent)
    └─── unblocks ──→ Task 8 (Main SKILL.md)

Task 1 (Plugin Scaffold)  ──→ Task 2 (Slash Command) ──→ Task 3 (MCP Config)

Task 4 + Task 5 + Task 6 + Task 7 (Synthesis Model) ──→ Task 8 (SKILL.md)

Task 8 (SKILL.md) ──→ Task 9 (HTML Template) ──→ Task 10 (Example Report)

Task 8 (SKILL.md) ──→ Task 11 (CLAUDE.md)

Task 1-11 ──→ Task 12 (Integration Test) ──→ Task 13 (E2E Test) ──→ Task 14 (Edge Cases)
```

**Parallelism opportunities:**
- Tasks 0 and 0b can run in sequence (0 first, 0b second) before any agent tasks
- Tasks 1, 2, 3 can run in parallel with Tasks 0 and 0b (no MCP dependency)
- Tasks 4, 5, 6 can run in parallel with each other (after Tasks 0 and 0b)
- Task 7 (Synthesis Model) can run in parallel with Tasks 4, 5, 6
- Task 9 (HTML Template) can run in parallel with Task 11 (CLAUDE.md) after Task 8
- Task 10 (Example Report) depends on Task 9

---

## File Manifest

When all tasks are complete, the plugin directory should contain:

```
insights-dashboard/
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json
├── CLAUDE.md
├── commands/
│   └── project-insights.md
├── agents/
│   ├── jira-discovery.md
│   ├── confluence-discovery.md
│   └── risk-assessment.md
├── skills/
│   └── project-insights/
│       ├── SKILL.md
│       ├── references/
│       │   ├── synthesis-model.md
│       │   ├── mcp-tool-reference.md
│       │   └── report-template.html
│       └── examples/
│           └── sample-report.html
├── reports/                          (gitignored, created at runtime)
│   └── YYYY-MM-DD-PROJ-slug/
│       ├── run-metadata.json
│       ├── raw/
│       │   ├── jira-epics.json
│       │   ├── jira-stories.json
│       │   └── confluence-pages.json
│       └── synthesized/
│           ├── scope-items.json
│           ├── scope-items-refined.json
│           ├── risk-assessments.json
│           └── people.json
└── docs/
    └── plans/
        ├── 2026-02-19-project-insights-skill-design.md
        └── 2026-02-19-project-insights-implementation.md
```
