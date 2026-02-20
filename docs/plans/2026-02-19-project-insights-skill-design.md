# Project Insights Skill Design

**Date:** 2026-02-19
**Status:** Approved (revised per architect review)

## Purpose

A reusable `/project-insights` skill for Claude Code that queries Jira and Confluence via MCP, synthesizes project scope, assesses execution coverage and risk, and produces an executive-level progress report.

## Problem Statement

Project scope lives across multiple systems (Confluence planning docs, Jira tickets) with many-to-many relationships. Executives need a consolidated view of: what was scoped, whether it's being tracked, what's at risk, and what needs attention. Manual assembly of this view is time-consuming and error-prone.

## Architecture: 4-Pass with User Refinement (File-Backed)

All agent outputs are persisted to local JSON files, not held in context. This prevents
context saturation, preserves progress if agents crash, and allows incremental re-runs.

```
PASS 1: DISCOVERY (automated, 2 parallel agents + synthesis)
    |  Jira Discovery Agent (haiku): query epics, stories, issues
    |  Confluence Discovery Agent (haiku): query planning docs
    |  Write raw findings -> raw/*.json
    |  Synthesis (opus): cross-reference extraction, clustering, scoring
    |  Write -> synthesized/scope-items.json
    v
CHECKPOINT: Present scope to user (reads from scope-items.json)
    |  User confirms, adds, removes, or clarifies scope items
    |  User can provide additional context (dates, priorities)
    |  User can request deeper search on specific items
    v
PASS 2: REFINED DISCOVERY (automated, if needed)
    |  Fetch additional details (haiku) for user-added items
    |  Re-synthesize (sonnet) with new data
    |  Write -> synthesized/scope-items-refined.json
    v
PASS 3: RISK ASSESSMENT (automated, single agent)
    |  Single sonnet agent reads scope-items-refined.json
    |  Batch-queries linked Jira tickets for current status
    |  Applies weighted risk scoring heuristic
    |  Write -> synthesized/risk-assessments.json, synthesized/people.json
    v
PASS 4: REPORT GENERATION (automated)
    |  Sonnet agent reads all synthesized/*.json files
    |  Produces styled HTML executive report
    '-- -> project-insights.html
```

### Model Specifications

Each agent and pass uses a model matched to the cognitive demands of the task:

| Agent/Pass | Model | Rationale |
|-----------|-------|-----------|
| Jira Discovery Agent | **haiku** | Mechanical data extraction — query MCP, extract fields, write JSON. No judgment needed. |
| Confluence Discovery Agent | **haiku** | Mechanical data extraction — query MCP, extract page metadata and content. |
| Pass 1b: Synthesis | **opus** | Hardest task — semantic clustering, fuzzy matching, judgment about scope item boundaries. Requires highest intelligence. |
| Pass 2: Refined Discovery | **haiku** for queries, **sonnet** for re-synthesis | Fetching is mechanical, but re-clustering needs judgment. |
| Pass 3: Risk Assessment | **sonnet** | Calculation + some judgment (interpreting blockers, assessing time alignment). Good balance of speed and quality. |
| Pass 4: Report Generation | **sonnet** | Requires generating executive prose (summary, recommendations) and populating HTML template. |

### Local Knowledge Base Structure

Each run creates an isolated dated directory. Multiple runs coexist — same project can be
analyzed on different dates, or different projects analyzed independently. Every run retains
its full context for auditing, comparison, or re-analysis.

```
reports/
'-- YYYY-MM-DD-PROJ-search-slug/
    |-- run-metadata.json             <- Parameters, status, pass checkpoints
    |-- raw/                          <- Agent outputs (structured data)
    |   |-- jira-epics.json           <- All epics with metadata
    |   |-- jira-stories.json         <- Stories linked to epics
    |   '-- confluence-pages.json     <- Page metadata + content excerpts
    |
    |-- synthesized/                  <- Post-processing outputs
    |   |-- cross-references.json     <- Links between Jira and Confluence (from synthesis)
    |   |-- scope-items.json          <- Clustered scope items (Pass 1 output)
    |   |-- scope-items-refined.json  <- After user checkpoint (Pass 2 output)
    |   |-- risk-assessments.json     <- Per scope-item risk data (Pass 3 output)
    |   '-- people.json               <- People -> scope item -> risk mapping
    |
    '-- project-insights.html          <- Final styled executive report (Pass 4 output)
```

The `run-metadata.json` file records the exact parameters and run status:
- Comparison between runs of the same project over time
- Re-running with the same parameters to track progress
- Understanding what sources were queried for any given report
- Resuming from the last successful pass on failure

### Error Handling and Resume

The skill implements checkpoint-based error handling to ensure resilience:

**Run Status Tracking:**

`run-metadata.json` includes a `status` field with one of the following values:
- `in_progress` — a pass is currently executing
- `checkpoint` — waiting for user input at the checkpoint step
- `completed` — all passes finished successfully
- `failed` — a pass encountered an unrecoverable error

Each pass updates the status before starting and after completing:

```json
{
  "status": "in_progress",
  "current_pass": "pass_1_discovery",
  "passes_completed": [],
  "last_updated": "2026-02-19T14:30:00Z",
  "error": null
}
```

After a pass completes successfully:

```json
{
  "status": "in_progress",
  "current_pass": "pass_3_risk",
  "passes_completed": ["pass_1_discovery", "pass_1b_synthesis", "pass_2_refinement"],
  "last_updated": "2026-02-19T14:35:00Z",
  "error": null
}
```

On failure:

```json
{
  "status": "failed",
  "current_pass": "pass_3_risk",
  "passes_completed": ["pass_1_discovery", "pass_1b_synthesis", "pass_2_refinement"],
  "last_updated": "2026-02-19T14:36:00Z",
  "error": "MCP connection timeout during Jira status query for PROJ-205"
}
```

**Resume Flag:**

The `--resume` flag continues from the last successful pass:
- Reads `run-metadata.json` to determine the last completed pass
- Skips all completed passes
- Re-executes the failed or next pending pass
- Requires specifying the existing run directory (not creating a new one)

**Atomic File Writes:**

All file writes use an atomic two-step process:
1. Write content to a temporary file (`<filename>.tmp`)
2. Rename the temporary file to the final filename on success

This prevents partial writes from corrupting the knowledge base if an agent crashes mid-write.

**JSON Validation:**

After every file write, the agent must:
1. Read the file back
2. Parse it as JSON to confirm validity
3. If parsing fails, report the error and retry the write

### Why File-Backed (Not Context-Only)

- **No context saturation**: Agents write to files and return only a brief status message. No agent needs to hold all findings in memory.
- **Crash resilience**: If a sub-agent fails mid-query, files already written are preserved. The pass can resume from where it left off.
- **Incremental re-runs**: On repeat invocations, agents can check for existing raw files and skip already-fetched data.
- **Auditable**: The raw/ directory preserves exactly what was queried, for review or debugging.
- **Decoupled passes**: Each pass reads from the previous pass's files, not from shared context. Passes can be re-run independently.

## MCP Validation

Before the first run, the skill validates that MCP capabilities are available and functional. This prevents confusing errors deep into a multi-pass run.

**Validation Steps:**

1. **List available tools**: Query the Atlassian MCP server for its tool manifest. Confirm that Jira and Confluence query tools are present. Log the actual tool names and signatures to `run-metadata.json` under `mcp_capabilities`.

2. **Test a simple query**: Execute a minimal Jira query (e.g., `project = <KEY> AND issuetype = Epic ORDER BY rank ASC LIMIT 1`) to confirm authentication and connectivity. If this fails, surface the error immediately with remediation guidance ("Run /mcp to authenticate with Atlassian").

3. **Document actual tool signatures**: Record the exact MCP tool names, parameter schemas, and return formats in `run-metadata.json`. This ensures agents use the correct tool invocations and provides a debugging reference if tool signatures change between MCP server versions.

**Validation Output (stored in run-metadata.json):**

```json
{
  "mcp_validation": {
    "validated_at": "2026-02-19T14:29:00Z",
    "server": "atlassian",
    "tools_available": ["jira_search", "jira_get_issue", "confluence_search", "confluence_get_page"],
    "test_query_success": true,
    "tool_signatures": {
      "jira_search": {"params": ["jql", "fields", "maxResults", "startAt"], "returns": "issues[]"},
      "confluence_search": {"params": ["cql", "limit", "start"], "returns": "results[]"}
    }
  }
}
```

If validation fails, the skill aborts with a clear error message and does not proceed to Pass 1.

## Skill Interface

### Invocation

```
/project-insights --jira PROJ --spaces TEAMSPACE --search "data pipeline" --target-date 2026-04-15
```

### Parameters

| Parameter | Required | Default | Example | Purpose |
|-----------|----------|---------|---------|---------|
| `--jira` | Yes | — | `PROJ`, `PROJ,ENG` | Jira project key(s), comma-separated |
| `--spaces` | Yes | — | `TEAMSPACE`, `TEAM,ARCH` | Confluence space key(s), comma-separated |
| `--search` | No | — | `"data pipeline migration"` | Free-text keywords to broaden discovery |
| `--target-date` | No | — | `2026-04-15` | User-supplied deadline (when not in Jira/Confluence) |
| `--stale-days` | No | 14 | `14` | Days of inactivity before flagging as stale |
| `--resume` | No | — | `reports/2026-02-19-PROJ-data-pipeline` | Resume a failed run from the last successful pass |

### Output

Each invocation creates an isolated run directory under `reports/`. Multiple runs coexist
without overwriting each other, and each retains its full context (raw data, synthesized
scope, risk assessments, and final report) for future reference or re-analysis.

```
reports/
|-- 2026-02-19-PROJ-data-pipeline/       <- Run 1
|   |-- raw/
|   |-- synthesized/
|   '-- project-insights.html
|-- 2026-02-25-PROJ-data-pipeline/       <- Re-run of same project (later date)
|   |-- raw/
|   |-- synthesized/
|   '-- project-insights.html
|-- 2026-02-20-ENG-auth-service/         <- Different project entirely
|   |-- raw/
|   |-- synthesized/
|   '-- project-insights.html
```

**Directory naming:** `YYYY-MM-DD-<jira-key>-<search-slug>/`
- Date ensures chronological ordering and uniqueness
- Jira key identifies the project
- Search slug (first keyword, kebab-cased) distinguishes runs for the same project on the same day
- If a directory already exists (same project, same day, same search), append a numeric suffix: `-2`, `-3`

## Agent Design

### Pass 1: Discovery (2 Parallel Agents + Synthesis)

**Jira Discovery Agent (haiku):**

Before querying for specific issue types, the agent first discovers the project's issue type scheme:
1. Query the project's metadata to retrieve available issue types and their hierarchy levels
2. Identify which issue types correspond to epics, initiatives, or other high-level containers
3. Fall back to querying all issue types sorted by hierarchy level if "Epic" or "Initiative" types do not exist in the project's scheme

Once the issue type scheme is known, proceed with targeted queries:
- Query epics, initiatives, and high-level stories in the specified project(s)
- Extract: epic summaries, linked stories, fix versions, sprint assignments
- Identify high-level deliverables and milestones
- **Writes:** `raw/jira-epics.json`, `raw/jira-stories.json`
- **Returns to parent:** Brief status ("Found 12 epics, 47 stories in PROJ")

**Pagination:** Jira returns a default of 100 results per page. The agent must handle pagination using the `startAt` parameter:
1. Issue the initial query with `startAt=0`
2. Check the response `total` field against the number of results returned
3. If `total > startAt + maxResults`, issue subsequent queries incrementing `startAt` by `maxResults`
4. Continue until all results are fetched
5. Concatenate all pages into the final output file

**Confluence Discovery Agent (haiku):**
- Search specified spaces for planning documents: PRDs, RFCs, tech specs, SOWs, decision logs, roadmaps
- Extract: page title, space, excerpt/content summary, labels, last modified date, author
- Use CQL with space key and search terms
- **Writes:** `raw/confluence-pages.json`
- **Returns to parent:** Brief status ("Found 8 planning docs in TEAMSPACE")

**Pagination:** Confluence returns a default of 25 results per page. The agent must handle pagination using the `start` parameter:
1. Issue the initial query with `start=0`
2. Check the response for a `_links.next` field or compare result count against `limit`
3. If more results exist, issue subsequent queries incrementing `start` by the page size
4. Continue until all results are fetched
5. Concatenate all pages into the final output file

**Synthesis Step (opus) — Pass 1b:**

This is the most critical step in the entire skill. The synthesis step reads all raw discovery files and produces a structured, clustered representation of project scope. Cross-reference detection, which was previously a separate agent, is now integrated into this synthesis step as its first phase.

See the **Synthesis Model** section below for the full specification.

### Checkpoint: User Refinement (reads from scope-items.json)

Read `synthesized/scope-items.json` and present discovered scope items to the user via AskUserQuestion. User can:
- Confirm scope items are correct
- Add missing scope items (with or without system links)
- Remove irrelevant items
- Provide additional context: dates, priorities, dependencies
- Request deeper search on specific topics

### Pass 2: Refined Discovery (If Needed)

This pass has two phases with different model requirements:

**Phase 2a: Fetch (haiku):**
- Pull additional details for user-added items via MCP queries
- Re-search for anything flagged as missing
- Append new findings to `raw/*.json` files
- Handle pagination as described above for both Jira and Confluence queries

**Phase 2b: Re-synthesis (sonnet):**
- Re-run the synthesis clustering algorithm with the expanded raw data
- Incorporate user corrections and additions
- Merge new findings with existing scope items where appropriate
- **Writes:** `synthesized/scope-items-refined.json`

### Pass 3: Risk Assessment (Single sonnet Agent)

Risk assessment is a calculation task, not a discovery task. A single sonnet agent handles all scope items in one pass:

1. Read `synthesized/scope-items-refined.json` to get the full list of scope items and their linked Jira tickets
2. For each scope item, batch-query all linked Jira tickets for their current status, assignee, last update date, and blockers via MCP
3. Apply the weighted risk scoring heuristic (see **Risk Heuristic** section below) to each scope item
4. Aggregate people data across all scope items
5. **Writes:** `synthesized/risk-assessments.json`, `synthesized/people.json`
6. **Returns to parent:** Summary status ("8 scope items assessed: 3 On Track, 3 At Risk, 2 Behind")

### Pass 4: Report Generation (sonnet)

Read all `synthesized/*.json` files and generate a self-contained styled HTML report.
- **Writes:** `project-insights.html`
- Uses the HTML/CSS template from `skills/project-insights/references/report-template.html`
- No agent context needs to hold all data — each section is generated by reading the relevant JSON file
- Report content guidance is embedded as HTML comments in the template itself

## Synthesis Model

This is the core intellectual task of the entire skill. The synthesis step takes raw, unstructured discovery data from Jira and Confluence and produces a structured, clustered set of scope items with confidence scores and many-to-many relationships. It runs as Pass 1b immediately after the parallel discovery agents complete, using the **opus** model for its high demands on semantic reasoning and judgment.

### Input

The synthesis step reads three files from `raw/`:
- `raw/jira-epics.json` — Epics and high-level issues with their child issue counts and metadata
- `raw/jira-stories.json` — Stories, subtasks, and other issues linked to epics
- `raw/confluence-pages.json` — Planning documents with extracted content, deliverables, and dates

### Phase 1: Cross-Reference Extraction

This phase replaces the former cross-source-search agent. The Jira and Confluence discovery agents already perform keyword searches and retrieve content. Cross-referencing is a parsing task performed on the data already collected, not an additional search task.

**Step 1: Parse Confluence pages for Jira key patterns.**

Scan the `content_summary` and `deliverables_mentioned` fields of every Confluence page for Jira issue key patterns using the regex `[A-Z]+-\d+`. For example, if a Confluence page titled "Pipeline Migration RFC" contains the text "See PROJ-101 for implementation tracking," the key `PROJ-101` is extracted and linked to that page.

**Step 2: Parse Jira ticket descriptions for Confluence URLs.**

Scan the `description_snippet` field of every Jira ticket for Confluence URLs. Match patterns such as:
- `https://<domain>.atlassian.net/wiki/spaces/<space>/pages/<page_id>/<title>`
- `https://<domain>.atlassian.net/wiki/x/<short_id>`
- Any URL containing `/wiki/` that points to the same Atlassian instance

Extract the page ID from matched URLs and link it to the Jira ticket.

**Step 3: Build the cross-reference map.**

Produce a bidirectional mapping:

```json
{
  "jira_to_confluence": {
    "PROJ-101": ["page-123", "page-456"],
    "PROJ-205": ["page-123"]
  },
  "confluence_to_jira": {
    "page-123": ["PROJ-101", "PROJ-205"],
    "page-456": ["PROJ-101"]
  }
}
```

**Step 4: Write the cross-reference map to `synthesized/cross-references.json`.**

This file is used by subsequent phases and is available for auditing. It is placed in `synthesized/` (not `raw/`) because it is derived from raw data through parsing, not directly from MCP queries.

### Phase 2: Initial Clustering

Build scope items by applying the following heuristics in strict priority order. Each heuristic is applied in sequence; items already claimed by a higher-priority heuristic are not reconsidered by lower-priority ones.

**Heuristic 1: Epic-anchored clusters (highest priority).**

Each Jira epic becomes a seed scope item. All stories and subtasks that are children of the epic (found via the `child_issues` or `linked_issues` fields in `jira-stories.json`) are grouped under it. If a Confluence page mentions the epic's key (found via the cross-reference map from Phase 1), that page is linked to the scope item as a Confluence source.

Example: Epic `PROJ-101` "Data Pipeline Migration" has 5 child stories. Confluence page "Pipeline Migration RFC" (page-123) mentions `PROJ-101`. All six items (1 epic + 5 stories + 1 Confluence page) form one scope item.

**Heuristic 2: Confluence-anchored clusters.**

Confluence pages with planning labels (`prd`, `rfc`, `spec`, `sow`, `roadmap`, `design`, `architecture`, `requirements`) that were NOT already claimed by an epic-anchored cluster become seed scope items. For each such page, search the Jira data for tickets whose summaries contain keywords from the page title. Matching tickets are grouped under this scope item.

Example: Confluence page "Authentication Service Redesign" (labeled `rfc`) is not referenced by any epic. Jira stories `ENG-401` "Redesign auth token flow" and `ENG-402` "Migrate to OAuth2" match on keywords "auth" and "redesign." These form one scope item anchored by the Confluence page.

**Heuristic 3: Keyword-overlap clusters.**

For items not yet claimed by heuristics 1 or 2, compute keyword overlap between:
- Jira ticket summaries (tokenized, stopwords removed)
- Confluence page titles and `deliverables_mentioned` fields (tokenized, stopwords removed)

Items with greater than 40% keyword overlap (measured as Jaccard similarity of keyword sets) are candidates for clustering together. When multiple items overlap, form the cluster around the item with the most connections.

Example: Jira story `PROJ-310` "Add monitoring dashboards for pipeline" and Confluence page "Observability Requirements" both contain keywords "monitoring" and "dashboards." Keyword overlap is 2/7 unique keywords = 28.6%, which is below 40%, so they are NOT clustered. But if the Confluence page also mentioned "pipeline," the overlap would be 3/7 = 42.8%, qualifying them for clustering.

**Heuristic 4: Cross-reference clusters.**

Items linked through the cross-reference map from Phase 1 that were not already clustered by heuristics 1-3 are always clustered together. This catches cases where a Jira ticket mentions a Confluence page (or vice versa) but neither is an epic nor a labeled planning doc.

**Heuristic 5: Orphans (lowest priority).**

Items that match nothing through any of the above heuristics remain as standalone scope items. They are flagged with `classification: "unmatched"` and a note: "unmatched — needs user review." These are surfaced prominently at the checkpoint so the user can decide whether they belong to an existing scope item, represent a new scope item, or should be excluded.

### Phase 3: Confidence Scoring

Each scope item receives a numerical confidence score (0.0 to 1.0) and a categorical confidence level. The confidence score reflects how certain the synthesis is that the grouped items truly belong together and represent a coherent scope item.

**High confidence (0.8 to 1.0):**
- Direct cross-references exist: a Jira key appears in a Confluence page body, or a Confluence URL appears in a Jira ticket description
- Epic-story hierarchy explicitly links the items (parent-child relationship in Jira)
- Multiple independent signals confirm the grouping (e.g., cross-reference AND keyword overlap AND same labels)

Scoring within the high range:
- 1.0: Epic with child stories and a Confluence page that directly references the epic key
- 0.9: Cross-reference exists in one direction (e.g., Confluence mentions Jira key, but Jira does not link back)
- 0.8: Epic-story hierarchy only, no Confluence cross-reference but page exists with matching labels

**Medium confidence (0.5 to 0.79):**
- Keyword overlap exceeds 40% but no direct cross-reference exists
- Items share the same Jira labels or components, suggesting topical alignment
- A Confluence page title partially matches a Jira epic summary but without explicit linking

Scoring within the medium range:
- 0.79: High keyword overlap (>60%) plus shared labels/components
- 0.65: Moderate keyword overlap (40-60%) with one additional signal (same sprint, same assignee)
- 0.5: Keyword overlap just above 40% with no additional signals

**Low confidence (0.0 to 0.49):**
- Weak keyword match (below 40% overlap) where items were grouped heuristically
- Orphan items that could not be matched to any other item
- Items grouped solely because they were the only remaining unmatched items in a topic area

Scoring within the low range:
- 0.49: Keyword overlap of 30-39%, just below the clustering threshold, but manually grouped due to topical similarity
- 0.25: Very weak signals, single keyword match
- 0.0: Complete orphan with no matching signals whatsoever

### Phase 4: Scope Item Naming

Each scope item needs a clear, concise name (3-8 words) that describes the capability or deliverable, not the document or ticket that defines it.

**Naming rules in priority order:**

1. If epic-anchored: use the epic summary as the scope item name. Epic summaries are typically already written as capability descriptions (e.g., "Data Pipeline Migration" rather than "PROJ-101").

2. If Confluence-anchored: use the Confluence page title as the scope item name, unless the title is generic (e.g., "Q2 Planning" or "Technical Spec"). In that case, extract a more specific name from the page's `deliverables_mentioned` field.

3. If merged cluster (multiple sources, no single anchor): select the most specific descriptive name from any source in the cluster. Prefer names that describe what is being built or delivered over names that describe the process or document type.

4. Names must be 3-8 words long. If the source name is longer, truncate to the most descriptive portion. If shorter than 3 words, expand with context from the deliverables or description fields.

5. Names should never include Jira keys, document types (RFC, PRD, spec), or dates. These are metadata, not names.

**Examples:**
- Good: "Data Pipeline Migration", "Authentication Service Redesign", "Real-Time Analytics Dashboard"
- Bad: "PROJ-101", "Pipeline RFC", "Q2 Infrastructure Work", "Spec"

### Phase 5: Output — scope-items.json

The synthesis step writes the final clustered, scored, and named scope items to `synthesized/scope-items.json` using the following schema:

```json
{
  "schema_version": "1.0",
  "generated_at": "2026-02-19T14:32:00Z",
  "scope_items": [
    {
      "id": "SI-001",
      "name": "Data Pipeline Migration",
      "confidence": 0.92,
      "confidence_level": "high",
      "cluster_reason": "Epic PROJ-101 cross-referenced in Confluence page 'Pipeline Migration RFC'",
      "confluence_sources": [
        {
          "page_id": "123",
          "title": "Pipeline Migration RFC",
          "url": "https://team.atlassian.net/wiki/spaces/TEAM/pages/123/Pipeline+Migration+RFC",
          "relevance": "direct_reference",
          "dates_mentioned": [
            {
              "date": "2026-04-15",
              "context": "target launch",
              "confidence": "high"
            }
          ],
          "deliverables_mentioned": ["pipeline v2", "schema validation"]
        }
      ],
      "jira_tracking": [
        {
          "key": "PROJ-101",
          "type": "Epic",
          "summary": "Data Pipeline Migration",
          "status": "In Progress",
          "relevance": "anchor"
        },
        {
          "key": "PROJ-205",
          "type": "Story",
          "summary": "Schema validation for pipeline v2",
          "status": "To Do",
          "relevance": "child_of_epic"
        },
        {
          "key": "PROJ-206",
          "type": "Story",
          "summary": "Pipeline v2 integration tests",
          "status": "In Progress",
          "relevance": "child_of_epic"
        },
        {
          "key": "PROJ-207",
          "type": "Story",
          "summary": "Data migration script for legacy pipeline",
          "status": "Done",
          "relevance": "child_of_epic"
        },
        {
          "key": "PROJ-310",
          "type": "Subtask",
          "summary": "Add monitoring dashboards for pipeline v2",
          "status": "To Do",
          "relevance": "keyword_match"
        }
      ],
      "target_date": {
        "date": "2026-04-15",
        "source": "confluence",
        "confidence": "high"
      },
      "classification": "tracked"
    },
    {
      "id": "SI-002",
      "name": "Authentication Service Redesign",
      "confidence": 0.65,
      "confidence_level": "medium",
      "cluster_reason": "Confluence page 'Authentication Service Redesign' keyword-matched with Jira stories ENG-401, ENG-402",
      "confluence_sources": [
        {
          "page_id": "456",
          "title": "Authentication Service Redesign",
          "url": "https://team.atlassian.net/wiki/spaces/ARCH/pages/456/Auth+Service+Redesign",
          "relevance": "keyword_match",
          "dates_mentioned": [
            {
              "date": "2026-03-31",
              "context": "milestone: token migration complete",
              "confidence": "medium"
            }
          ],
          "deliverables_mentioned": ["OAuth2 migration", "token refresh flow", "session management"]
        }
      ],
      "jira_tracking": [
        {
          "key": "ENG-401",
          "type": "Story",
          "summary": "Redesign auth token flow",
          "status": "In Progress",
          "relevance": "keyword_match"
        },
        {
          "key": "ENG-402",
          "type": "Story",
          "summary": "Migrate to OAuth2",
          "status": "To Do",
          "relevance": "keyword_match"
        }
      ],
      "target_date": {
        "date": "2026-03-31",
        "source": "confluence",
        "confidence": "medium"
      },
      "classification": "tracked"
    },
    {
      "id": "SI-003",
      "name": "Cost Optimization Analysis",
      "confidence": 0.0,
      "confidence_level": "low",
      "cluster_reason": "Orphan Confluence page, no matching Jira tickets found — needs user review",
      "confluence_sources": [
        {
          "page_id": "789",
          "title": "Q2 Cost Optimization Analysis",
          "url": "https://team.atlassian.net/wiki/spaces/TEAM/pages/789/Q2+Cost+Optimization",
          "relevance": "label_match",
          "dates_mentioned": [],
          "deliverables_mentioned": ["cost reduction targets", "infrastructure audit"]
        }
      ],
      "jira_tracking": [],
      "target_date": null,
      "classification": "scoped_but_untracked"
    }
  ],
  "unmatched_jira": ["PROJ-999", "ENG-550"],
  "unmatched_confluence": ["page-321"]
}
```

### Classification Values

Each scope item receives a classification based on the presence of Confluence and Jira sources:

- **`tracked`**: Both Confluence scope documentation and Jira tracking exist. This is the ideal state — the scope item was planned and is being executed.
- **`scoped_but_untracked`**: A Confluence source exists (PRD, RFC, spec) but no Jira tickets were found. The scope item was planned but may not be in active execution.
- **`execution_only`**: Jira tickets exist but no Confluence scope document was found. Work is happening but may not have been formally scoped or planned.
- **`unmatched`**: An orphan item that could not be confidently grouped. Needs user review at the checkpoint to determine if it belongs to an existing scope item or should be excluded.

### Synthesis Model Relationship Diagram

```
Scope Item (synthesized capability/deliverable)
    |
    |-- Confluence Sources (0..many)
    |   |-- PRD page mentioning this capability
    |   |-- Tech spec detailing implementation
    |   '-- Decision log entry
    |
    '-- Jira Tracking (0..many)
        |-- PROJ-101 (epic, anchor)
        |-- PROJ-205 (story, child_of_epic)
        |-- PROJ-206 (story, child_of_epic)
        '-- PROJ-310 (subtask, keyword_match)
```

A scope item with Confluence sources but zero Jira tickets = `scoped_but_untracked`.
A scope item with Jira tickets but no Confluence source = `execution_only`.
A scope item with both = `tracked`.
A scope item with neither (orphan) = `unmatched`.

The many-to-many nature is critical: one Confluence page may describe multiple scope items, and one Jira epic may be relevant to multiple scope items (though this is less common). The synthesis step must handle these cases by allowing items to appear in multiple clusters when warranted, with appropriate confidence adjustments.

## Risk Heuristic

Risk assessment uses a weighted scoring model that replaces simple binary thresholds with nuanced, context-aware calculations.

### Risk Score Calculation

```
base_score = (done + 0.5 * in_progress) / total_tickets  # Range: 0.0 to 1.0

# Penalties (each reduces the score)
stale_penalty = (stale_count / total_tickets) * 0.3
blocker_penalty = (blocked_count / total_tickets) * 0.5
time_penalty = max(0, pct_time_elapsed - base_score) * 0.4  # Only applies if behind schedule

# Final risk score
risk_score = base_score - stale_penalty - blocker_penalty - time_penalty
```

Where:
- `done` = number of tickets in a "Done" or "Closed" status
- `in_progress` = number of tickets in an "In Progress" or equivalent active status
- `total_tickets` = total number of Jira tickets linked to the scope item
- `stale_count` = tickets with no update in more than `stale_days` (default 14)
- `blocked_count` = tickets with blocker links or a "Blocked" status
- `pct_time_elapsed` = proportion of time elapsed between project start and target date (0.0 to 1.0)

### Risk Rating Thresholds

The risk rating is derived from the risk score with special-case overrides:

| Condition | Rating | Rationale |
|-----------|--------|-----------|
| Scope is `scoped_but_untracked` (no Jira tickets) | **BEHIND** | Cannot assess progress without tracking |
| Target date has passed and `base_score < 1.0` | **BEHIND** | Deadline missed with incomplete work |
| `risk_score >= 0.6` | **ON TRACK** | Strong progress with manageable issues |
| `risk_score >= 0.3` | **AT RISK** | Significant concerns but not yet critical |
| `risk_score < 0.3` | **BEHIND** | Insufficient progress or severe blockers |

### Context-Aware Adjustments

The raw risk score is adjusted based on project context before applying the rating thresholds:

**Near-completion adjustment:** If `base_score > 0.8` (more than 80% of tickets complete), reduce `stale_penalty` by 50%. Rationale: when a scope item is nearly done, remaining stale tickets are likely minor cleanup items, not indicators of systemic risk.

**No target date adjustment:** If no target date is available from any source (Jira, Confluence, or user-provided), skip `time_penalty` entirely. Rationale: without a deadline, time-based risk assessment is meaningless and would produce misleading scores.

**Early project adjustment:** If the project is less than 2 weeks old (based on the earliest Jira ticket creation date in the scope item), reduce all penalties by 30%. Rationale: it is too early in the project lifecycle to draw meaningful conclusions from inactivity or slow starts.

### Worked Example

Consider a scope item "Data Pipeline Migration" with 12 linked tickets:
- 6 done, 3 in progress, 2 not started, 1 blocked
- 2 tickets stale (no update in 20 days)
- Target date: 2026-04-15, current date: 2026-02-19, project start: 2026-01-01
- Time elapsed: 50 days / 105 days total = 47.6%

```
base_score = (6 + 0.5 * 3) / 12 = 7.5 / 12 = 0.625

stale_penalty = (2 / 12) * 0.3 = 0.05
blocker_penalty = (1 / 12) * 0.5 = 0.042
time_penalty = max(0, 0.476 - 0.625) * 0.4 = max(0, -0.149) * 0.4 = 0.0  # Ahead of schedule

risk_score = 0.625 - 0.05 - 0.042 - 0.0 = 0.533

Rating: AT RISK (0.533 is between 0.3 and 0.6)
```

The scope item is rated AT RISK despite being ahead on the time-progress ratio because the stale tickets and blocker drag the score below the ON TRACK threshold.

## Executive Report Format

The report is a **self-contained HTML file** with embedded CSS. It loads fonts from Google
Fonts and renders correctly when opened in any browser. No external dependencies beyond
the font CDN.

### Typography

| Element | Font | Weight | Usage |
|---------|------|--------|-------|
| Body text, paragraphs, descriptions | Roboto | 400 (Regular) | All prose content |
| Headings, titles, section headers | Roboto Condensed | 700 (Bold) | H1-H4, table headers, risk labels |
| Code, numbers, Jira keys, references | Roboto Mono | 400 (Regular) | `PROJ-101`, dates, percentages, URLs |

### Color Coding for Risk

| Rating | Color | Usage |
|--------|-------|-------|
| On Track | `#1B8A2D` (green) | Risk badge, progress indicators |
| At Risk | `#D97706` (amber) | Risk badge, stale item highlights |
| Behind | `#DC2626` (red) | Risk badge, blocked item highlights |

### Report Sections (HTML Structure)

```html
<header>
  <h1>Project Insights: [Project Name]</h1>
  <div class="meta">
    Generated: <code>YYYY-MM-DD</code> |
    Sources: Jira <code>[PROJ]</code>, Confluence <code>[TEAMSPACE]</code>
  </div>
</header>

<section id="executive-summary">
  <!-- 3-5 sentence overview with overall health badge -->
</section>

<section id="stated-scope">
  <!-- Each scope item as a card with:
       - Name (h3, Roboto Condensed)
       - Defined in: Confluence links (Roboto Mono for URLs)
       - Tracked by: Jira keys (Roboto Mono)
       - Target date (Roboto Mono for date)
       - Progress bar (visual)
       - Risk badge (colored) -->
</section>

<section id="risk-dashboard">
  <!-- Summary table with sortable columns
       Risk badges use colored pills
       Progress shown as X/Y with Roboto Mono
       Dates in Roboto Mono -->
</section>

<section id="attention-required">
  <!-- Blocked items, stale items, untracked scope
       Each with severity indicator and details -->
</section>

<section id="key-people">
  <!-- Lean table: name, scope items, risk items only -->
</section>

<section id="recommendations">
  <!-- Prioritized numbered list of actions -->
</section>
```

The full HTML/CSS template is maintained in `skills/project-insights/references/report-template.html`. Report content guidance (executive summary tone, priority ordering for attention items, people section scope, recommendation style) is embedded as HTML comments within the template itself.

## Reference Documents

The skill maintains two reference documents. Content guidance for the report and risk heuristic details are kept inline (in the HTML template as comments and in SKILL.md respectively) rather than in separate files.

| File | Purpose |
|------|---------|
| `skills/project-insights/references/synthesis-model.md` | Detailed synthesis methodology — clustering algorithms, confidence scoring rubric, naming conventions. Complex enough to warrant its own file. |
| `skills/project-insights/references/report-template.html` | Self-contained HTML/CSS template with embedded content guidance comments. |

Risk heuristics are documented inline in `SKILL.md` as a decision table, not in a separate reference file. The scoring formula, thresholds, and context adjustments are concise enough to keep in the main skill file.

## MCP Dependencies

| Service | MCP Server | Transport | Status |
|---------|-----------|-----------|--------|
| Jira + Confluence | Atlassian (`https://mcp.atlassian.com/v1/mcp`) | HTTP | Installed, needs OAuth auth |
| Google Drive | `workspace-mcp` via `uvx` | stdio | Not yet installed (future) |

## Future Extensions

- **Google Drive agent**: Add as a 3rd discovery agent in Pass 1 once MCP is configured
- **Confluence publishing**: Optionally publish the report as a Confluence page
- **Scheduled runs**: Periodic report generation for ongoing tracking
- **Scope creep detection**: Flag work in Jira that was not in original stated scope
- **Historical trending**: Compare risk scores across multiple runs to show trajectory
- **Custom issue type mapping**: Allow users to specify which issue types to treat as epics/initiatives
