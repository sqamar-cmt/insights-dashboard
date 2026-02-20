# Synthesis Model Reference

## 1. Purpose and Context

Synthesis is the intellectual core of the project-insights pipeline. It takes raw Jira, Confluence, and local document data (from `raw/*.json`) and produces **logical scope items** -- the unit of analysis for the rest of the pipeline.

Each scope item represents a coherent capability, deliverable, or workstream. Scope items have **many-to-many relationships** with their sources: a single scope item may be tracked by multiple Jira tickets AND described by multiple Confluence pages AND referenced in multiple local documents. A single source document may contribute to multiple scope items.

**Note:** Local document data is optional. If `raw/local-docs.json` does not exist, all local-doc-related steps are skipped and the algorithm operates on Jira + Confluence only (the original 2-source model).

The synthesis step runs in the parent context using **opus** (the most capable model) because it requires complex cross-referencing, clustering, and judgment.

## 2. The 4-Phase Synthesis Algorithm

### Phase 1: Cross-Reference Extraction

Build a bidirectional map between Jira keys, Confluence pages, and local docs.

**Step 1a -- Confluence → Jira references:**
- Read `raw/confluence-pages.json`
- For each page, extract the `jira_keys_mentioned` array
- Build map: `{confluence_page_id: [jira_keys]}`

**Step 1b -- Jira → Confluence references:**
- Read `raw/jira-epics.json`
- For each epic, scan `description_snippet` for Confluence page URLs or titles
- URL patterns: `*/wiki/spaces/*/pages/*`, `*/wiki/display/*/*`
- Title patterns: Match against known page titles from `raw/confluence-pages.json`
- Build map: `{jira_key: [confluence_page_ids]}`

**Step 1c -- Epic-to-story hierarchy:**
- Read `raw/jira-stories.json`
- Build map: `{epic_key: [story_keys]}` using the `parent_key` field
- This hierarchy is used to attach child stories to epic-anchored clusters

**Step 1d -- LocalDoc → Jira references (if local docs data exists):**
- Read `raw/local-docs.json`
- For each doc, extract the `jira_keys_mentioned` array
- Build map: `{file_path: [jira_keys]}`

**Step 1e -- LocalDoc → Confluence references (if local docs data exists):**
- For each local doc, scan extracted text content for Confluence URLs
- Match Confluence URLs to page IDs from `raw/confluence-pages.json`
- Build map: `{file_path: [confluence_page_ids]}`

**Step 1f -- Record all cross-references:**
- Classify each cross-reference as:
  - `explicit_reference`: Jira key appears in Confluence page AND/OR Confluence URL appears in Jira ticket AND/OR Jira key appears in local doc AND/OR Confluence URL appears in local doc
  - `keyword_overlap`: Matched by keyword similarity (no direct link)
- Store in the `cross_references` section of the output for audit trail
- Cross-references now span three source types: Jira↔Confluence, Jira↔LocalDoc, and Confluence↔LocalDoc

### Phase 2: Initial Clustering

Create clusters using a priority-ordered approach. Each item can belong to at most one cluster.

**2a -- Epic-anchored clusters (highest priority):**
- Start with each Jira epic as a seed
- Attach its child stories (from Phase 1c hierarchy)
- Attach any Confluence pages that explicitly mention the epic key (from Phase 1a)
- Attach any local docs that explicitly mention the epic key (from Phase 1d)
- Attach any Confluence pages whose `deliverables_mentioned` overlap significantly with the epic summary (2+ matching keywords, excluding stop words and common project terms)
- Attach any local docs whose `deliverables_mentioned` overlap significantly with the epic summary (same criteria)
- Mark these items as "assigned"

**2b -- Confluence-anchored clusters:**
- For Confluence pages NOT yet assigned to an epic-anchored cluster:
  - Check if they describe a distinct deliverable
  - If multiple unassigned pages reference the same deliverable (by keyword overlap in `deliverables_mentioned`), group them together
  - Look for any unassigned Jira stories whose summaries match the deliverable keywords
  - Look for any unassigned local docs whose deliverables or titles match the deliverable keywords
- Mark these items as "assigned"

**2c -- LocalDoc-anchored clusters (if local docs data exists):**
- For local docs NOT yet assigned to an epic-anchored or confluence-anchored cluster:
  - Check if they describe a distinct deliverable
  - If multiple unassigned local docs reference the same deliverable (by keyword overlap in `deliverables_mentioned`), group them together
  - Look for any unassigned Jira stories whose summaries match the deliverable keywords
  - Look for any unassigned Confluence pages whose deliverables match
  - Also check LocalDoc→Confluence cross-references (Phase 1e) to pull in related Confluence pages
- Mark these items as "assigned"

**2d -- Keyword-overlap clusters:**
- For remaining unmatched items:
  - Compare Jira ticket summaries against Confluence page titles, local doc titles, and `deliverables_mentioned` from both
  - Match if 2+ significant keywords overlap
  - Significant keywords: Exclude stop words (the, a, an, is, are, was, were, be, been, being, have, has, had, do, does, did, will, would, shall, should, may, might, must, can, could, of, in, to, for, with, on, at, by, from, as, or, and, not, no, but, if, then, else, when, up, out, about) and common project terms (project, team, sprint, task, issue, ticket, jira, confluence, page, update, review, create, local, file, document)
- Mark these items as "assigned"

**2e -- Orphans:**
- Items that do not cluster with anything else become single-item scope items
- Flag orphans for user attention at the checkpoint
- Separate into `unmatched_jira`, `unmatched_confluence`, and `unmatched_local_docs` lists

### Phase 3: Confidence Scoring

Each scope item receives a synthesis confidence score (0.0 to 1.0):

**High (0.8-1.0):**
- Explicit bidirectional cross-references exist (Jira key in Confluence page AND Confluence URL in Jira ticket)
- OR: Epic has clear child stories AND Confluence page explicitly references the epic key
- OR: Multiple explicit unidirectional references (e.g., 3+ Confluence pages all mention the same epic)
- OR: **Three-source corroboration** -- all three sources (Jira + Confluence + LocalDoc) reference the same scope item, even if individual links are keyword-based. Three independent sources confirming the same deliverable is strong evidence. Apply a **+0.15 confidence boost** (capped at 1.0) when a scope item has explicit or keyword references from all three source types.

**Medium (0.5-0.79):**
- Keyword overlap match with 3+ matching terms
- OR: Epic-anchored cluster with Confluence pages matched by deliverable keywords (no explicit cross-references)
- OR: Single explicit unidirectional reference with keyword corroboration
- OR: Two-source corroboration from any pair (Jira+LocalDoc, Confluence+LocalDoc) with keyword overlap

**Low (0.2-0.49):**
- Single keyword match only
- OR: Clustering based solely on label/sprint/component co-occurrence
- OR: Orphan items grouped by weak heuristic
- OR: Epic with no Confluence or local doc documentation found
- OR: LocalDoc-only scope item (no Jira or Confluence corroboration)

Confidence score informs the checkpoint presentation: low-confidence items are highlighted with `[NEEDS REVIEW]` for user attention.

### Phase 4: Scope Item Naming

Use the most descriptive human-readable name from available sources.

**Priority order:**
1. Confluence PRD/RFC title if it describes a specific deliverable (e.g., "Data Pipeline V2 - PRD" → "Data Pipeline V2 Migration")
2. Local doc title (filename without extension, cleaned up) if it describes a specific deliverable (e.g., "Pipeline V2 Technical Spec.pdf" → "Pipeline V2 Technical Spec")
3. Jira epic summary if concise (e.g., "Migrate data pipeline to v2" → "Data Pipeline V2 Migration")
4. Synthesized name from overlapping keywords (e.g., keywords "monitoring", "dashboard", "alerts" → "Monitoring and Observability")

**Naming rules:**
- 3-8 words, action-oriented where possible
- Avoid Jira keys in the name (they go in `jira_sources`)
- Avoid generic words like "Project" or "Work" unless truly descriptive
- Use title case

## 3. Output Schema: `scope-items.json`

```json
{
  "synthesized_at": "2026-02-19T14:35:00Z",
  "project_keys": ["PROJ"],
  "confluence_spaces": ["TEAMSPACE"],
  "local_docs_subfolder": "data-pipeline",
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
      "local_doc_sources": [
        {
          "file_path": "prd.pdf",
          "file_name": "prd.pdf",
          "file_type": "pdf",
          "match_type": "explicit_reference",
          "dates_mentioned": [
            {
              "date": "2026-04-01",
              "context": "code freeze for v2 migration",
              "confidence": "high"
            }
          ],
          "deliverables_mentioned": ["pipeline v2", "rollback procedures", "data validation"]
        }
      ],
      "target_dates": [
        {
          "date": "2026-04-15",
          "source": "confluence",
          "page_title": "Data Pipeline V2 - PRD",
          "confidence": "high"
        },
        {
          "date": "2026-04-01",
          "source": "local_doc",
          "doc_title": "prd.pdf",
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
      "local_doc_sources": [],
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
      "notes": "No Confluence or local doc documentation found. Flagged for user review."
    }
  ],
  "orphans": {
    "unmatched_jira": [
      {
        "key": "PROJ-400",
        "summary": "Investigate caching layer performance",
        "reason": "No matching Confluence/local doc content or epic grouping found"
      }
    ],
    "unmatched_confluence": [
      {
        "page_id": "67890",
        "title": "Q2 Platform Scalability Plan",
        "reason": "No matching Jira epics or stories found"
      }
    ],
    "unmatched_local_docs": [
      {
        "file_path": "api-gateway-proposal.docx",
        "file_name": "api-gateway-proposal.docx",
        "reason": "No matching Jira epics or Confluence pages found"
      }
    ]
  },
  "cross_references": {
    "explicit": [
      {
        "jira_key": "PROJ-101",
        "confluence_page_id": "12345",
        "local_doc_path": null,
        "direction": "both",
        "evidence": "PROJ-101 description links to page 12345; page 12345 mentions PROJ-101"
      },
      {
        "jira_key": "PROJ-101",
        "confluence_page_id": null,
        "local_doc_path": "prd.pdf",
        "direction": "local_doc_to_jira",
        "evidence": "Local doc 'prd.pdf' mentions PROJ-101"
      }
    ],
    "inferred": [
      {
        "jira_key": "PROJ-310",
        "confluence_page_id": "12345",
        "local_doc_path": null,
        "direction": "keyword",
        "evidence": "Keywords overlap: 'monitoring', 'dashboard', 'pipeline'"
      }
    ]
  }
}
```

## 4. Worked Examples

### Example 1: Clean Epic-Anchored Match

**Input:**
- Epic `PROJ-101` "Migrate data pipeline to v2" with 8 child stories
- Confluence page "Data Pipeline V2 - PRD" in TEAMSPACE mentions `PROJ-101` in its content and lists deliverables: "pipeline v2", "schema validation", "monitoring dashboards"
- The epic's description contains a link to the Confluence page

**Process:**
- Phase 1: Finds explicit bidirectional cross-reference (PROJ-101 ↔ page 12345)
- Phase 2a: Creates epic-anchored cluster with the epic, its 8 child stories, and the Confluence page
- Phase 3: Scores **0.92** (explicit bidirectional references + child stories corroborate scope)
- Phase 4: Names it "Data Pipeline V2 Migration" from the PRD title (cleaned up from "Data Pipeline V2 - PRD")

**Output:** Single scope item with 1 Confluence source, 1 epic + 8 child stories as Jira sources, confidence 0.92.

### Example 2: Keyword-Overlap Match with Multiple Sources

**Input:**
- No explicit cross-references between Jira and Confluence
- Confluence pages "Monitoring Strategy" and "Observability RFC" both mention deliverables: "dashboards", "alerts", "SLOs"
- Jira stories PROJ-310 ("Build alerting dashboard"), PROJ-311 ("Define SLO thresholds"), PROJ-312 ("Integrate Grafana metrics") have summaries about dashboards and alerting but belong to different epics or no epic

**Process:**
- Phase 1: Finds no explicit links
- Phase 2a: PROJ-310, 311, 312 don't cluster under a single epic (they're from different epics or are unparented)
- Phase 2b: Groups "Monitoring Strategy" and "Observability RFC" together (deliverable keyword overlap: "dashboards", "alerts", "SLOs")
- Phase 2c: Finds PROJ-310, 311, 312 by keyword overlap with the Confluence-anchored cluster ("dashboard", "alert", "SLO")
- Phase 3: Scores **0.55** (keyword overlap only, no explicit references, but 3+ keyword matches)
- Phase 4: Names it "Monitoring and Observability" from the RFC title

**Output:** Scope item with medium confidence, 2 Confluence sources, 3 Jira stories as sources. Flagged for user review due to medium confidence.

### Example 3: Orphan Items

**Input:**
- Jira epic PROJ-400 "Investigate caching layer performance" has 2 child stories
- No Confluence pages mention caching, performance, or PROJ-400
- No local docs mention caching, performance, or PROJ-400

**Process:**
- Phase 1: Finds no cross-references
- Phase 2a: Creates epic-anchored cluster (PROJ-400 + 2 children) but with no Confluence or local doc sources
- Phase 3: Scores **0.45** (single-source epic with no corroborating evidence)
- Phase 4: Names it "Caching Layer Performance Investigation" from the epic summary

**Output:** Scope item with low confidence and note: "No Confluence or local doc documentation found. Flagged for user review."

### Example 4: Three-Source Corroboration (Jira + Confluence + Local Doc)

**Input:**
- Epic `PROJ-101` "Migrate data pipeline to v2" with 8 child stories
- Confluence page "Data Pipeline V2 - PRD" mentions `PROJ-101` and lists deliverables
- Local doc "pipeline-v2-technical-spec.pdf" mentions `PROJ-101` and contains a reference to the Confluence PRD. Lists deliverables: "pipeline v2", "rollback procedures", "data validation". Contains date "2026-04-01" as code freeze.

**Process:**
- Phase 1a: Confluence page → Jira: finds PROJ-101
- Phase 1b: Jira epic → Confluence: finds link to page 12345
- Phase 1d: Local doc → Jira: finds PROJ-101
- Phase 1e: Local doc → Confluence: finds URL reference to page 12345
- Phase 2a: Creates epic-anchored cluster with epic, 8 stories, Confluence page, AND local doc (all explicitly reference the same epic)
- Phase 3: Base score **0.92** (explicit bidirectional Jira↔Confluence) + **0.15 three-source boost** = **1.0** (capped). All three sources independently confirm the same deliverable.
- Phase 4: Names it "Data Pipeline V2 Migration" from the Confluence PRD title (highest priority)

**Output:** Scope item with confidence 1.0, 1 Confluence source, 1 local doc source, 1 epic + 8 child stories as Jira sources. The local doc provides an additional target date (2026-04-01 code freeze) not found in the Confluence PRD.

## 5. Edge Cases

### Duplicate Detection
If two epics in different projects describe the same deliverable (e.g., PROJ-101 "Migrate pipeline" and ENG-50 "Pipeline v2 migration"), merge them into a single scope item with both as `jira_sources`. Detect duplicates by:
- Explicit cross-references to the same Confluence page or local doc
- 3+ keyword overlap in epic summaries

### Overly Broad Documents
If a Confluence page or local doc mentions 10+ Jira keys, it is likely a release notes page, changelog, or sprint summary -- not a planning document. Handle by:
- Deprioritize it as a scope source
- Mark its `match_type` as "reference" instead of "planning"
- Do not use it as a primary anchor for clustering
- Still record its Jira key mentions in cross-references for audit

### Empty Confluence Results
If no Confluence pages are found:
- Scope items come from Jira and/or local doc sources
- Items with only Jira sources receive low confidence (0.2-0.49)
- Items with Jira + local doc sources receive medium confidence (benefit from 2-source corroboration)
- Note in the output: "No Confluence planning documents found."

### Empty Local Docs Results
If `raw/local-docs.json` does not exist or contains zero docs:
- Skip all local-doc-related steps in Phases 1-4
- Algorithm operates on Jira + Confluence only (original 2-source model)
- No confidence penalty -- local docs are an optional enhancement

### LocalDoc-Only Scope Items
If a local doc describes a deliverable not found in Jira or Confluence:
- It becomes an orphan in `unmatched_local_docs`
- If the user promotes it to a scope item at checkpoint, it receives low confidence (0.2-0.3) until corroborating Jira/Confluence sources are found in Pass 2

### No Epics in Jira
If the Jira project uses flat stories without epics:
- Skip Phase 2a (epic-anchored clustering)
- In Phase 2c, cluster stories by:
  1. Label co-occurrence (stories sharing 2+ labels)
  2. Sprint co-occurrence (stories in the same sprint with similar summaries)
  3. Fix version grouping (stories targeting the same release)
  4. Component grouping (stories under the same Jira component)
- These clusters receive medium-to-low confidence

### Single Scope Item
If all evidence points to a single deliverable (small project), produce a single scope item. This is valid -- not every project has multiple workstreams.

### Conflicting Dates
If a scope item has dates from Jira (due dates), Confluence (mentioned dates), and/or local docs (mentioned dates):
- Include all dates in the `target_dates` array with their source type (`jira`, `confluence`, or `local_doc`)
- Prefer Jira due dates (highest confidence, directly actionable) for risk assessment
- Local doc and Confluence mentioned dates have equal weight
- Note conflicts: "Jira due date (2026-03-15) differs from Confluence target (2026-04-15) and local doc target (2026-04-01)"
