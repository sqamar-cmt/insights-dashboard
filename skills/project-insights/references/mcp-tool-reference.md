# Atlassian MCP Tool Reference

> **Source:** [sooperset/mcp-atlassian](https://github.com/sooperset/mcp-atlassian) (open-source community MCP server)
> **Also see:** [Atlassian Remote MCP Server](https://www.atlassian.com/blog/announcements/remote-mcp-server) (first-party, beta, Cloudflare-hosted)
> **Last updated:** 2026-02-19
> **Extracted from:** Source code (`src/mcp_atlassian/servers/jira.py`, `src/mcp_atlassian/servers/confluence.py`)

---

## Authentication

### Method 1: API Token (Recommended for Cloud)
- **Auth method:** Basic Auth (username + API token)
- **Env vars:** `JIRA_USERNAME`, `JIRA_API_TOKEN`, `CONFLUENCE_USERNAME`, `CONFLUENCE_API_TOKEN`
- **Token source:** https://id.atlassian.com/manage-profile/security/api-tokens

### Method 2: Personal Access Token (Server / Data Center)
- **Auth method:** PAT header auth
- **Env vars:** `JIRA_PERSONAL_TOKEN`, `CONFLUENCE_PERSONAL_TOKEN`

### Method 3: OAuth 2.0 (3LO) (Cloud Only)
- **Auth method:** OAuth 2.0 with refresh tokens
- **Env vars:** `ATLASSIAN_OAUTH_CLIENT_ID`, `ATLASSIAN_OAUTH_CLIENT_SECRET`, `ATLASSIAN_OAUTH_REDIRECT_URI`, `ATLASSIAN_OAUTH_SCOPE`, `ATLASSIAN_OAUTH_CLOUD_ID`
- **Required scopes:** `read:jira-work write:jira-work read:jira-user read:confluence-space.summary read:confluence-content.summary read:confluence-content.all write:confluence-content search:confluence read:page:confluence offline_access`
- **Setup:** Run `mcp-atlassian --oauth-setup -v`

### Method 4: Bring Your Own Token (BYOT) OAuth (Cloud Only)
- **Auth method:** Pre-existing externally managed OAuth access token
- **Env vars:** `ATLASSIAN_OAUTH_CLOUD_ID`, `ATLASSIAN_OAUTH_ACCESS_TOKEN`
- **Note:** No token refresh; external system manages tokens

### Access Control
- **Read-only mode:** `READ_ONLY_MODE=true` disables all write tools
- **Tool filtering:** `ENABLED_TOOLS=tool1,tool2` limits which tools are available
- **Project filtering:** `JIRA_PROJECTS_FILTER=PROJ,DEVOPS` limits Jira project scope
- **Space filtering:** `CONFLUENCE_SPACES_FILTER=DEV,TEAM` limits Confluence space scope

---

## Jira Tools

### jira_get_user_profile
- **Description:** Retrieve profile information for a specific Jira user.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `user_identifier` (string, required): Identifier for the user -- email address (`user@example.com`), username (`johndoe`), account ID (`accountid:...`), or key (Server/DC).
- **Returns:** JSON string `{ "success": true, "user": { ... } }` with user profile object, or `{ "success": false, "error": "...", "user_identifier": "..." }` on failure.
- **Pagination:** N/A
- **Notes:** Supports Cloud and Server/DC identifier formats.

---

### jira_get_issue
- **Description:** Get details of a specific Jira issue including Epic links and relationship information.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key, e.g. `PROJ-123`. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `fields` (string, optional, default: DEFAULT_READ_JIRA_FIELDS joined): Comma-separated list of fields. Use `*all` for all fields, or specific fields like `summary,status,customfield_10010`.
  - `expand` (string | null, optional, default: null): Fields to expand, e.g. `renderedFields`, `transitions`, `changelog`.
  - `comment_limit` (int, optional, default: 10, range: 0-100): Maximum number of comments to include. 0 for no comments.
  - `properties` (string | null, optional, default: null): Comma-separated list of issue properties to return.
  - `update_history` (bool, optional, default: true): Whether to update issue view history for the requesting user.
- **Returns:** JSON string of the Jira issue object (simplified dict).
- **Pagination:** N/A (single issue)
- **Notes:** Fields parameter accepts comma-separated field IDs or `*all`.

---

### jira_search
- **Description:** Search Jira issues using JQL (Jira Query Language).
- **Tags:** `jira`, `read`
- **Parameters:**
  - `jql` (string, required): JQL query string. Examples: `issuetype = Epic AND project = PROJ`, `assignee = currentUser()`, `updated >= -7d AND project = PROJ`.
  - `fields` (string, optional, default: DEFAULT_READ_JIRA_FIELDS): Comma-separated fields. Use `*all` for all.
  - `limit` (int, optional, default: 10, min: 1): Maximum number of results.
  - `start_at` (int, optional, default: 0, min: 0): Starting index for pagination (0-based).
  - `projects_filter` (string | null, optional, default: null): Comma-separated project keys. Overrides `JIRA_PROJECTS_FILTER` env var.
  - `expand` (string | null, optional, default: null): Fields to expand, e.g. `renderedFields`, `transitions`, `changelog`.
- **Returns:** JSON string with search results including pagination info: `{ "total": N, "start_at": 0, "max_results": 10, "issues": [...] }`.
- **Pagination:** Use `start_at` offset and `limit`. Increment `start_at` by `limit` for next page. Check `total` for completeness.
- **Notes:** JQL is case-sensitive for field names but not values. `projects_filter` auto-appends `AND project IN (...)`.

---

### jira_search_fields
- **Description:** Search Jira fields by keyword with fuzzy match.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `keyword` (string, optional, default: ""): Keyword for fuzzy search. Empty lists first `limit` fields in default order.
  - `limit` (int, optional, default: 10, min: 1): Maximum number of results.
  - `refresh` (bool, optional, default: false): Whether to force refresh the field list cache.
- **Returns:** JSON string with list of matching field definitions (id, name, type, etc.).
- **Pagination:** N/A (uses limit only)
- **Notes:** Useful for discovering custom field IDs (`customfield_XXXXX`).

---

### jira_get_project_issues
- **Description:** Get all issues for a specific Jira project.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `project_key` (string, required): Jira project key, e.g. `PROJ`. Pattern: `^[A-Z][A-Z0-9]+$`
  - `limit` (int, optional, default: 10, range: 1-50): Maximum number of results.
  - `start_at` (int, optional, default: 0, min: 0): Starting index for pagination.
- **Returns:** JSON string with search results including pagination info.
- **Pagination:** Same as `jira_search`.

---

### jira_get_transitions
- **Description:** Get available status transitions for a Jira issue.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
- **Returns:** JSON string with list of available transitions `[{ "id": "11", "name": "To Do", "to": { ... } }, ...]`.
- **Pagination:** N/A
- **Notes:** Must call this before `jira_transition_issue` to get valid transition IDs.

---

### jira_get_worklog
- **Description:** Get worklog entries for a Jira issue.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
- **Returns:** JSON string `{ "worklogs": [...] }`.
- **Pagination:** N/A (returns all worklogs)

---

### jira_download_attachments
- **Description:** Download all attachments from a Jira issue as base64-encoded embedded resources.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
- **Returns:** List of `TextContent` (summary) and `EmbeddedResource` (base64 blob per attachment). Summary: `{ "success": true, "issue_key": "...", "total": N, "downloaded": N, "failed": [...] }`.
- **Pagination:** N/A
- **Notes:** Returns MCP `EmbeddedResource` type with `attachment:///ISSUE-KEY/filename` URI.

---

### jira_get_agile_boards
- **Description:** Get Jira agile boards by name, project key, or type.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `board_name` (string | null, optional): Board name for fuzzy search.
  - `project_key` (string | null, optional): Jira project key. Pattern: `^[A-Z][A-Z0-9]+$`
  - `board_type` (string | null, optional): Board type, e.g. `scrum`, `kanban`.
  - `start_at` (int, optional, default: 0, min: 0): Starting index for pagination.
  - `limit` (int, optional, default: 10, range: 1-50): Maximum results.
- **Returns:** JSON array of board objects.
- **Pagination:** Offset-based via `start_at` and `limit`.

---

### jira_get_board_issues
- **Description:** Get all issues linked to a specific board filtered by JQL.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `board_id` (string, required): Board ID, e.g. `1001`.
  - `jql` (string, required): JQL query to filter issues.
  - `fields` (string, optional, default: DEFAULT_READ_JIRA_FIELDS): Comma-separated fields.
  - `start_at` (int, optional, default: 0, min: 0): Starting index.
  - `limit` (int, optional, default: 10, range: 1-50): Maximum results.
  - `expand` (string, optional, default: "version"): Fields to expand, e.g. `changelog`.
- **Returns:** JSON string with search results including pagination info.
- **Pagination:** Offset-based via `start_at` and `limit`.

---

### jira_get_sprints_from_board
- **Description:** Get Jira sprints from board, optionally filtered by state.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `board_id` (string, required): Board ID, e.g. `1000`.
  - `state` (string | null, optional): Sprint state filter: `active`, `future`, `closed`. Null returns all.
  - `start_at` (int, optional, default: 0, min: 0): Starting index.
  - `limit` (int, optional, default: 10, range: 1-50): Maximum results.
- **Returns:** JSON array of sprint objects.
- **Pagination:** Offset-based via `start_at` and `limit`.

---

### jira_get_sprint_issues
- **Description:** Get Jira issues from a sprint.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `sprint_id` (string, required): Sprint ID, e.g. `10001`.
  - `fields` (string, optional, default: DEFAULT_READ_JIRA_FIELDS): Comma-separated fields.
  - `start_at` (int, optional, default: 0, min: 0): Starting index.
  - `limit` (int, optional, default: 10, range: 1-50): Maximum results.
- **Returns:** JSON string with search results including pagination info.
- **Pagination:** Offset-based via `start_at` and `limit`.

---

### jira_get_issue_link_types
- **Description:** Get all available issue link types.
- **Tags:** `jira`, `read`
- **Parameters:** None (only `ctx`)
- **Returns:** JSON array of link type objects, e.g. `[{ "id": "...", "name": "Blocks", "inward": "is blocked by", "outward": "blocks" }, ...]`.
- **Pagination:** N/A

---

### jira_get_all_projects
- **Description:** Get all Jira projects accessible to the current user.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `include_archived` (bool, optional, default: false): Whether to include archived projects.
- **Returns:** JSON array of project objects. Keys always uppercase. Filtered by `JIRA_PROJECTS_FILTER` if configured.
- **Pagination:** N/A (returns all projects)

---

### jira_get_project_versions
- **Description:** Get all fix versions for a specific Jira project.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `project_key` (string, required): Jira project key. Pattern: `^[A-Z][A-Z0-9]+$`
- **Returns:** JSON array of version objects.
- **Pagination:** N/A

---

### jira_get_issue_dates
- **Description:** Get date information and status transition history for a Jira issue. Returns created, updated, due date, resolution date, and optionally status change history with time tracking.
- **Tags:** `jira`, `read`, `metrics`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `include_status_changes` (bool, optional, default: true): Include status change history with timestamps and durations.
  - `include_status_summary` (bool, optional, default: true): Include aggregated time spent in each status.
- **Returns:** JSON string with issue dates and optional status tracking data.
- **Pagination:** N/A

---

### jira_get_issue_sla
- **Description:** Calculate SLA metrics for a Jira issue (cycle time, lead time, time in status, due date compliance, etc.).
- **Tags:** `jira`, `read`, `metrics`, `sla`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `metrics` (string | null, optional, default: configured or `cycle_time,time_in_status`): Comma-separated list. Available: `cycle_time`, `lead_time`, `time_in_status`, `due_date_compliance`, `resolution_time`, `first_response_time`.
  - `working_hours_only` (bool | null, optional, default: from env `JIRA_SLA_WORKING_HOURS_ONLY`): Calculate using working hours only.
  - `include_raw_dates` (bool, optional, default: false): Include raw date values in response.
- **Returns:** JSON string with calculated SLA metrics.
- **Pagination:** N/A
- **Notes:** Working hours configurable via `JIRA_SLA_WORKING_HOURS_START`, `JIRA_SLA_WORKING_HOURS_END`, `JIRA_SLA_WORKING_DAYS`, `JIRA_SLA_TIMEZONE` env vars.

---

### jira_get_issue_development_info
- **Description:** Get development information (PRs, commits, branches) linked to a Jira issue from connected source control systems (Bitbucket, GitHub, GitLab).
- **Tags:** `jira`, `read`, `development`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key, e.g. `PROJ-123`.
  - `application_type` (string | null, optional): Filter by source control type: `stash` (Bitbucket Server), `bitbucket`, `github`, `gitlab`.
  - `data_type` (string | null, optional): Filter by data type: `pullrequest`, `branch`, `repository`.
- **Returns:** JSON string with `{ "pullRequests": [...], "branches": [...], "commits": [...], "repositories": [...] }`.
- **Pagination:** N/A

---

### jira_get_issues_development_info
- **Description:** Batch get development information for multiple Jira issues.
- **Tags:** `jira`, `read`, `development`
- **Parameters:**
  - `issue_keys` (list[string], required): List of Jira issue keys, e.g. `["PROJ-123", "PROJ-456"]`.
  - `application_type` (string | null, optional): Filter by source control type.
  - `data_type` (string | null, optional): Filter by data type.
- **Returns:** JSON array with development info per issue.
- **Pagination:** N/A

---

### jira_batch_get_changelogs
- **Description:** Get changelogs for multiple Jira issues. **Cloud only.**
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_ids_or_keys` (list[string], required): List of issue IDs or keys, e.g. `["PROJ-123", "PROJ-124"]`.
  - `fields` (list[string] | null, optional, default: null): Filter changelogs by fields, e.g. `["status", "assignee"]`. Null for all fields.
  - `limit` (int, optional, default: -1): Max changelogs per issue in results. -1 for all (but still fetches all data).
- **Returns:** JSON array `[{ "issue_id": "...", "changelogs": [...] }]`.
- **Pagination:** N/A (fetches all; `limit` only truncates output)
- **Notes:** Raises `NotImplementedError` on Server/Data Center.

---

### jira_get_issue_proforma_forms
- **Description:** Get all ProForma forms associated with a Jira issue.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key, e.g. `PROJ-123`.
- **Returns:** JSON string `{ "success": true, "forms": [...], "count": N }` or error object.
- **Pagination:** N/A
- **Notes:** Uses Jira Forms REST API. Form IDs are UUIDs.

---

### jira_get_proforma_form_details
- **Description:** Get detailed information about a specific ProForma form including ADF design structure.
- **Tags:** `jira`, `read`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key.
  - `form_id` (string, required): ProForma form UUID, e.g. `1946b8b7-8f03-4dc0-ac2d-5fac0d960c6a`.
- **Returns:** JSON string `{ "success": true, "form": { ... } }` or error object.
- **Pagination:** N/A

---

### jira_create_issue
- **Description:** Create a new Jira issue.
- **Tags:** `jira`, `write` (requires write access)
- **Parameters:**
  - `project_key` (string, required): Project key, e.g. `PROJ`. Pattern: `^[A-Z][A-Z0-9]+$`
  - `summary` (string, required): Summary/title of the issue.
  - `issue_type` (string, required): Issue type, e.g. `Task`, `Bug`, `Story`, `Epic`, `Subtask`.
  - `assignee` (string | null, optional): Assignee identifier -- email, display name, or account ID.
  - `description` (string | null, optional): Description in Markdown format.
  - `components` (string | null, optional): Comma-separated component names, e.g. `Frontend,API`.
  - `additional_fields` (dict | string | null, optional): Dict or JSON string of additional fields. Examples: `{"priority": {"name": "High"}}`, `{"labels": ["frontend"]}`, `{"parent": "PROJ-123"}`, `{"fixVersions": [{"id": "10020"}]}`, `{"customfield_10010": "value"}`.
- **Returns:** JSON string `{ "message": "Issue created successfully", "issue": { ... } }`.
- **Notes:** For subtasks, use `Subtask` type and include `parent` in `additional_fields`.

---

### jira_batch_create_issues
- **Description:** Create multiple Jira issues in a batch.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issues` (string, required): JSON array string of issue objects. Each: `{ "project_key": "PROJ", "summary": "...", "issue_type": "Task", "description": "...", "assignee": "...", "components": [...] }`.
  - `validate_only` (bool, optional, default: false): If true, validates without creating.
- **Returns:** JSON `{ "message": "Issues created successfully", "issues": [...] }`.

---

### jira_update_issue
- **Description:** Update an existing Jira issue including status changes, Epic links, field updates, and attachments.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `fields` (dict, required): Dictionary of fields to update. For `assignee`, provide string identifier. For `description`, use Markdown. Example: `{"assignee": "user@example.com", "summary": "New Summary"}`.
  - `additional_fields` (dict | null, optional): Additional fields for custom or complex updates.
  - `attachments` (string | null, optional): JSON array string or comma-separated file paths, e.g. `/path/to/file1.txt,/path/to/file2.txt`.
- **Returns:** JSON string `{ "message": "Issue updated successfully", "issue": { ... } }`.

---

### jira_delete_issue
- **Description:** Delete an existing Jira issue.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
- **Returns:** JSON string `{ "message": "Issue PROJ-123 has been deleted successfully." }`.

---

### jira_add_comment
- **Description:** Add a comment to a Jira issue.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `comment` (string, required): Comment text in Markdown format.
  - `visibility` (dict | null, optional): Comment visibility, e.g. `{"type": "group", "value": "jira-users"}`.
- **Returns:** JSON string representing the added comment.

---

### jira_edit_comment
- **Description:** Edit an existing comment on a Jira issue.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `comment_id` (string, required): The ID of the comment to edit.
  - `comment` (string, required): Updated comment text in Markdown format.
  - `visibility` (dict | null, optional): Comment visibility settings.
- **Returns:** JSON string representing the updated comment.

---

### jira_add_worklog
- **Description:** Add a worklog entry to a Jira issue.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `time_spent` (string, required): Time in Jira format, e.g. `1h 30m`, `1d`, `30m`, `4h`.
  - `comment` (string | null, optional): Worklog comment in Markdown.
  - `started` (string | null, optional): Start time in ISO format, e.g. `2023-08-01T12:00:00.000+0000`. Defaults to current time.
  - `original_estimate` (string | null, optional): New original estimate value.
  - `remaining_estimate` (string | null, optional): New remaining estimate value.
- **Returns:** JSON string `{ "message": "Worklog added successfully", "worklog": { ... } }`.

---

### jira_transition_issue
- **Description:** Transition a Jira issue to a new status.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `transition_id` (string, required): ID of the transition (get from `jira_get_transitions`), e.g. `11`, `21`, `31`.
  - `fields` (dict | null, optional): Fields to update during transition, e.g. `{"resolution": {"name": "Fixed"}}`.
  - `comment` (string | null, optional): Comment for the transition in Markdown.
- **Returns:** JSON string `{ "message": "Issue PROJ-123 transitioned successfully", "issue": { ... } }`.
- **Notes:** Always call `jira_get_transitions` first to get valid transition IDs.

---

### jira_link_to_epic
- **Description:** Link an existing issue to an epic.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Issue key to link. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `epic_key` (string, required): Epic key to link to. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
- **Returns:** JSON string `{ "message": "Issue ... linked to epic ...", "issue": { ... } }`.

---

### jira_create_issue_link
- **Description:** Create a link between two Jira issues.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `link_type` (string, required): Link type, e.g. `Duplicate`, `Blocks`, `Relates to`.
  - `inward_issue_key` (string, required): Source issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `outward_issue_key` (string, required): Target issue key. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `comment` (string | null, optional): Comment to add to the link.
  - `comment_visibility` (dict | null, optional): Visibility for the comment, e.g. `{"type": "group", "value": "jira-users"}`.
- **Returns:** JSON string indicating success or failure.

---

### jira_create_remote_issue_link
- **Description:** Create a remote issue link (web link or Confluence link) for a Jira issue. Appears in the issue's "Links" section.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Issue key to add the link to. Pattern: `^[A-Z][A-Z0-9]+-\d+$`
  - `url` (string, required): URL to link to (any web page or Confluence page).
  - `title` (string, required): Display title for the link.
  - `summary` (string | null, optional): Description of the link.
  - `relationship` (string | null, optional): Relationship description, e.g. `causes`, `relates to`, `documentation`.
  - `icon_url` (string | null, optional): URL to a 16x16 icon for the link.
- **Returns:** JSON string indicating success or failure.

---

### jira_remove_issue_link
- **Description:** Remove a link between two Jira issues.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `link_id` (string, required): The ID of the link to remove.
- **Returns:** JSON string indicating success.

---

### jira_create_sprint
- **Description:** Create a Jira sprint for a board.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `board_id` (string, required): Board ID, e.g. `1000`.
  - `sprint_name` (string, required): Sprint name, e.g. `Sprint 1`.
  - `start_date` (string, required): Start time in ISO 8601 format.
  - `end_date` (string, required): End time in ISO 8601 format.
  - `goal` (string | null, optional): Goal of the sprint.
- **Returns:** JSON string representing the created sprint object.

---

### jira_update_sprint
- **Description:** Update an existing Jira sprint.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `sprint_id` (string, required): Sprint ID, e.g. `10001`.
  - `sprint_name` (string | null, optional): New name.
  - `state` (string | null, optional): New state: `future`, `active`, `closed`.
  - `start_date` (string | null, optional): New start date.
  - `end_date` (string | null, optional): New end date.
  - `goal` (string | null, optional): New goal.
- **Returns:** JSON string of updated sprint, or error payload.

---

### jira_create_version
- **Description:** Create a new fix version in a Jira project.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `project_key` (string, required): Project key. Pattern: `^[A-Z][A-Z0-9]+$`
  - `name` (string, required): Version name.
  - `start_date` (string | null, optional): Start date `YYYY-MM-DD`.
  - `release_date` (string | null, optional): Release date `YYYY-MM-DD`.
  - `description` (string | null, optional): Description.
- **Returns:** JSON string of the created version object, or `{ "success": false, "error": "..." }`.

---

### jira_batch_create_versions
- **Description:** Batch create multiple versions in a Jira project.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `project_key` (string, required): Project key. Pattern: `^[A-Z][A-Z0-9]+$`
  - `versions` (string, required): JSON array string of version objects. Each: `{ "name": "v1.0", "startDate": "YYYY-MM-DD", "releaseDate": "YYYY-MM-DD", "description": "..." }`.
- **Returns:** JSON array of results, each `{ "success": true, "version": { ... } }` or `{ "success": false, "error": "..." }`.

---

### jira_update_proforma_form_answers
- **Description:** Update ProForma form field answers using the Jira Forms REST API.
- **Tags:** `jira`, `write`
- **Parameters:**
  - `issue_key` (string, required): Jira issue key.
  - `form_id` (string, required): ProForma form UUID.
  - `answers` (list[dict], required): List of answer objects. Each: `{ "questionId": "q1", "type": "TEXT", "value": "..." }`. Types: `TEXT`, `NUMBER`, `DATE`, `DATETIME`, `SELECT`, `MULTI_SELECT`, `CHECKBOX`.
- **Returns:** JSON string `{ "success": true, "message": "...", "updated_fields": N, "result": { ... } }`.
- **Notes:** **DATETIME fields lose time components** -- only date is preserved. Workaround: use `jira_update_issue` to set underlying custom fields directly. ISO 8601 strings are auto-converted to Unix timestamps for DATE/DATETIME fields.

---

## Confluence Tools

### confluence_search
- **Description:** Search Confluence content using simple text terms or CQL (Confluence Query Language).
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `query` (string, required): Search query. Can be simple text (auto-converted to `siteSearch ~ "..."` with fallback to `text ~ "..."`) or CQL. Examples: `type=page AND space=DEV`, `title~"Meeting Notes"`, `label=documentation`, `lastModified > startOfMonth("-1M")`.
  - `limit` (int, optional, default: 10, range: 1-50): Maximum number of results.
  - `spaces_filter` (string | null, optional, default: null): Comma-separated space keys. Overrides `CONFLUENCE_SPACES_FILTER` env var. Empty string disables filtering.
- **Returns:** JSON array of simplified Confluence page objects.
- **Pagination:** Limited by `limit` parameter. No offset parameter -- use CQL sorting and date/ID filters for deep pagination.
- **Notes:** Simple text queries use `siteSearch` by default with automatic fallback to `text` search if unsupported. Personal space keys starting with `~` must be quoted in CQL.

---

### confluence_get_page
- **Description:** Get content of a specific Confluence page by ID, or by title and space key.
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `page_id` (string | null, optional): Numeric page ID from URL, e.g. `123456789`. If provided, `title` and `space_key` are ignored.
  - `title` (string | null, optional): Exact page title. Must be used with `space_key`.
  - `space_key` (string | null, optional): Space key, e.g. `DEV`, `TEAM`. Required if using `title`.
  - `include_metadata` (bool, optional, default: true): Include page metadata (creation date, last update, version, labels).
  - `convert_to_markdown` (bool, optional, default: true): Convert to Markdown (true) or keep raw HTML (false). Raw HTML reveals macros but increases token usage.
- **Returns:** JSON string: `{ "metadata": { ... } }` (with metadata) or `{ "content": { "value": "..." } }` (without).
- **Pagination:** N/A
- **Notes:** Must provide either `page_id` OR both `title` and `space_key`. Raises `ValueError` if neither.

---

### confluence_get_page_children
- **Description:** Get child pages and folders of a specific Confluence page.
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `parent_id` (string, required): ID of the parent page.
  - `expand` (string, optional, default: "version"): Fields to expand, e.g. `version`, `body.storage`.
  - `limit` (int, optional, default: 25, range: 1-50): Maximum child items to return.
  - `include_content` (bool, optional, default: false): Include page content in response.
  - `convert_to_markdown` (bool, optional, default: true): Convert content to Markdown. Only relevant if `include_content` is true.
  - `start` (int, optional, default: 0, min: 0): Starting index for pagination (0-based).
  - `include_folders` (bool, optional, default: true): Include child folders in addition to pages.
- **Returns:** JSON string `{ "parent_id": "...", "count": N, "limit_requested": N, "start_requested": N, "results": [...] }`.
- **Pagination:** Offset-based via `start` and `limit`.

---

### confluence_get_comments
- **Description:** Get comments for a specific Confluence page.
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `page_id` (string, required): Confluence page ID.
- **Returns:** JSON array of comment objects.
- **Pagination:** N/A

---

### confluence_get_labels
- **Description:** Get labels for Confluence content (pages, blog posts, or attachments).
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `page_id` (string, required): Content ID. For pages: numeric ID. For attachments: ID with `att` prefix, e.g. `att123456789`.
- **Returns:** JSON array of label objects.
- **Pagination:** N/A

---

### confluence_search_user
- **Description:** Search Confluence users using CQL.
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `query` (string, required): CQL query for user search, e.g. `user.fullname ~ "First Last"`. Simple text auto-converts to `user.fullname ~ "..."`.
  - `limit` (int, optional, default: 10, range: 1-50): Maximum results.
- **Returns:** JSON array of simplified Confluence user search result objects.
- **Pagination:** N/A (uses limit only)

---

### confluence_get_page_history
- **Description:** Get a historical version of a specific Confluence page.
- **Tags:** `confluence`, `read`
- **Parameters:**
  - `page_id` (string, required): Confluence page ID.
  - `version` (int, required, min: 1): Version number to retrieve.
  - `convert_to_markdown` (bool, optional, default: true): Convert to Markdown or keep raw HTML.
- **Returns:** JSON string representing the page content at the specified version.
- **Pagination:** N/A

---

### confluence_get_page_views
- **Description:** Get view statistics for a Confluence page. **Cloud only.**
- **Tags:** `confluence`, `read`, `analytics`
- **Parameters:**
  - `page_id` (string, required): Confluence page ID.
  - `include_title` (bool, optional, default: true): Whether to fetch and include the page title.
- **Returns:** JSON string with page view statistics (total views, last viewed date).
- **Pagination:** N/A
- **Notes:** Only available on Confluence Cloud. Server/Data Center instances do not support the Analytics API.

---

### confluence_get_attachments
- **Description:** List all attachments for a Confluence content item (page or blog post).
- **Tags:** `confluence`, `read`, `attachments`
- **Parameters:**
  - `content_id` (string, required): Content ID, e.g. `123456789`.
  - `start` (int, optional, default: 0): Starting index for pagination.
  - `limit` (int, optional, default: 50, range: 1-100): Max attachments per request.
  - `filename` (string | null, optional): Exact filename filter, e.g. `report.pdf`.
  - `media_type` (string | null, optional): MIME type filter. **Caveat:** Confluence returns `application/octet-stream` for most binary files (PNG, JPG, PDF). Use `filename` for more reliable filtering.
- **Returns:** JSON string with list of attachments (ID, title, file type, size, download URL, dates, version).
- **Pagination:** Offset-based via `start` and `limit`.
- **Notes:** Check `mediaTypeDescription` field for human-readable type (e.g., "PNG Image").

---

### confluence_download_attachment
- **Description:** Download a single attachment from Confluence as a base64-encoded embedded resource.
- **Tags:** `confluence`, `read`, `attachments`
- **Parameters:**
  - `attachment_id` (string, required): Attachment ID, e.g. `att123456789`. Find IDs using `confluence_get_attachments`.
- **Returns:** `EmbeddedResource` with base64-encoded content and `attachment:///attachment_id/filename` URI, or `TextContent` with error. Files larger than 50 MB are rejected.
- **Pagination:** N/A
- **Notes:** Max file size: 50 MB.

---

### confluence_download_content_attachments
- **Description:** Download all attachments for a Confluence content item as base64-encoded embedded resources.
- **Tags:** `confluence`, `read`, `attachments`
- **Parameters:**
  - `content_id` (string, required): Content ID, e.g. `123456789`.
- **Returns:** List: first item is `TextContent` summary `{ "success": true, "content_id": "...", "total": N, "downloaded": N, "failed": [...] }`, followed by `EmbeddedResource` per successful download. Files > 50 MB are skipped.
- **Pagination:** N/A

---

### confluence_create_page
- **Description:** Create a new Confluence page.
- **Tags:** `confluence`, `write`
- **Parameters:**
  - `space_key` (string, required): Space key, e.g. `DEV`, `TEAM`, `DOC`.
  - `title` (string, required): Page title.
  - `content` (string, required): Page content. Format depends on `content_format`.
  - `parent_id` (string | null, optional): Parent page ID for creating a child page.
  - `content_format` (string, optional, default: "markdown"): Content format: `markdown` (default), `wiki` (Confluence wiki markup), `storage` (Confluence storage format/XHTML).
  - `enable_heading_anchors` (bool, optional, default: false): Enable automatic heading anchor generation. Only for `markdown` format.
  - `emoji` (string | null, optional): Page title emoji/icon for navigation, e.g. any emoji character. Null to remove.
- **Returns:** JSON string `{ "message": "Page created successfully", "page": { ... } }`.

---

### confluence_update_page
- **Description:** Update an existing Confluence page.
- **Tags:** `confluence`, `write`
- **Parameters:**
  - `page_id` (string, required): Page ID to update.
  - `title` (string, required): New page title.
  - `content` (string, required): New page content.
  - `is_minor_edit` (bool, optional, default: false): Whether this is a minor edit.
  - `version_comment` (string | null, optional): Comment for this version.
  - `parent_id` (string | null, optional): New parent page ID (for moving pages).
  - `content_format` (string, optional, default: "markdown"): `markdown`, `wiki`, or `storage`.
  - `enable_heading_anchors` (bool, optional, default: false): Enable heading anchors (markdown only).
  - `emoji` (string | null, optional): Page title emoji.
- **Returns:** JSON string `{ "message": "Page updated successfully", "page": { ... } }`.

---

### confluence_delete_page
- **Description:** Delete an existing Confluence page.
- **Tags:** `confluence`, `write`
- **Parameters:**
  - `page_id` (string, required): Page ID to delete.
- **Returns:** JSON string `{ "success": true, "message": "Page ... deleted successfully" }` or `{ "success": false, ... }`.

---

### confluence_add_label
- **Description:** Add a label to Confluence content (pages, blog posts, or attachments).
- **Tags:** `confluence`, `write`
- **Parameters:**
  - `page_id` (string, required): Content ID. For pages: numeric ID. For attachments: `att` prefix.
  - `name` (string, required): Label name (lowercase, no spaces), e.g. `draft`, `reviewed`, `confidential`.
- **Returns:** JSON array of updated label objects.

---

### confluence_add_comment
- **Description:** Add a comment to a Confluence page.
- **Tags:** `confluence`, `write`
- **Parameters:**
  - `page_id` (string, required): Page ID.
  - `content` (string, required): Comment content in Markdown format.
- **Returns:** JSON string `{ "success": true, "message": "Comment added successfully", "comment": { ... } }` or error.

---

### confluence_upload_attachment
- **Description:** Upload a single attachment to Confluence content. Creates a new version if the file already exists.
- **Tags:** `confluence`, `write`, `attachments`
- **Parameters:**
  - `content_id` (string, required): Content ID to attach to, e.g. `123456789`.
  - `file_path` (string, required): Full path to file. Absolute or relative.
  - `comment` (string | null, optional): Attachment comment visible in history.
  - `minor_edit` (bool, optional, default: false): If true, watchers are not notified.
- **Returns:** JSON string `{ "message": "Attachment uploaded successfully", "attachment": { ... } }`.

---

### confluence_upload_attachments
- **Description:** Upload multiple attachments to Confluence content in a single operation.
- **Tags:** `confluence`, `write`, `attachments`
- **Parameters:**
  - `content_id` (string, required): Content ID.
  - `file_paths` (list[string], required): List of file paths, e.g. `["./file1.pdf", "./file2.png"]`.
  - `comment` (string | null, optional): Comment for all attachments.
  - `minor_edit` (bool, optional, default: false): If true, no notifications.
- **Returns:** JSON string `{ "message": "Uploaded N attachment(s) successfully", "attachments": [...] }`.

---

### confluence_delete_attachment
- **Description:** Permanently delete an attachment from Confluence. Cannot be undone.
- **Tags:** `confluence`, `write`, `attachments`
- **Parameters:**
  - `attachment_id` (string, required): Attachment ID, e.g. `att123456789`.
- **Returns:** JSON string `{ "message": "Attachment deleted successfully", "attachment_id": "..." }`.
- **Notes:** Deletes ALL versions permanently.

---

## Error Handling

### Standard Error Response Shape
All tools return errors as JSON strings with consistent structure:

```json
{
  "success": false,
  "error": "Human-readable error message",
  "issue_key": "PROJ-123"
}
```

### Authentication Errors
- **Type:** `MCPAtlassianAuthenticationError`
- **Shape:** `{ "error": "Authentication failed. Please check your credentials.", "details": "..." }`
- **Cause:** Invalid API token, expired OAuth token, insufficient permissions

### Not Found Errors
- **Type:** `ValueError` with "not found" in message
- **Shape:** `{ "success": false, "error": "...", "issue_key": "..." }`
- **Logged at:** WARNING level

### Network / API Errors
- **Type:** `HTTPError`, `OSError`
- **Shape:** `{ "success": false, "error": "Network or API Error: ..." }`
- **Logged at:** ERROR level

### Validation Errors
- **Type:** `ValueError`
- **Cause:** Missing required parameters, invalid JSON, invalid pattern match (e.g., issue key format)
- **Shape:** Raises `ValueError` directly (not caught as JSON response)

### Write Access Denied
- **Cause:** `READ_ONLY_MODE=true` or tool not in `ENABLED_TOOLS`
- **Shape:** `ValueError` raised by `@check_write_access` decorator

### Rate Limiting
- **Observed behavior:** No explicit rate limiting at the MCP server level. Rate limits are inherited from the underlying Atlassian REST API:
  - Atlassian Cloud: Standard rate limits apply (varies by endpoint; typically ~100-200 req/min for REST API v2)
  - Server/DC: Configurable by admin
- **No retry logic built in** -- callers must handle HTTP 429 responses

### Cloud-Only Features
- `jira_batch_get_changelogs`: Raises `NotImplementedError` on Server/DC
- `confluence_get_page_views`: Raises `ValueError` on Server/DC (Analytics API unavailable)

---

## Pagination Summary

| Pattern | Parameters | Tools |
|---------|-----------|-------|
| Offset-based | `start_at` (0-based) + `limit` | `jira_search`, `jira_get_project_issues`, `jira_get_board_issues`, `jira_get_sprint_issues`, `jira_get_agile_boards`, `jira_get_sprints_from_board` |
| Offset-based | `start` (0-based) + `limit` | `confluence_get_page_children`, `confluence_get_attachments` |
| Limit only | `limit` | `confluence_search`, `confluence_search_user`, `jira_search_fields` |
| No pagination | N/A | All single-entity and list-all tools |

**General pattern:** Increment `start_at`/`start` by `limit` for each subsequent page. Check `total` in response to know when complete.

---

## Sub-Agent Access
- **Can sub-agents call MCP tools:** To be validated in Task 12
- **Auth inheritance:** To be validated in Task 12
- **Any differences from parent-context calls:** To be validated in Task 12

---

## Complete Tool Index

### Jira Read Tools (19)
| # | Tool Name | Description |
|---|-----------|-------------|
| 1 | `jira_get_user_profile` | Get user profile by identifier |
| 2 | `jira_get_issue` | Get issue details |
| 3 | `jira_search` | Search issues with JQL |
| 4 | `jira_search_fields` | Search available fields by keyword |
| 5 | `jira_get_project_issues` | Get all issues in a project |
| 6 | `jira_get_transitions` | Get available status transitions |
| 7 | `jira_get_worklog` | Get worklog entries |
| 8 | `jira_download_attachments` | Download issue attachments |
| 9 | `jira_get_agile_boards` | Get agile boards |
| 10 | `jira_get_board_issues` | Get board issues filtered by JQL |
| 11 | `jira_get_sprints_from_board` | Get sprints from a board |
| 12 | `jira_get_sprint_issues` | Get issues in a sprint |
| 13 | `jira_get_issue_link_types` | Get available link types |
| 14 | `jira_get_all_projects` | Get all accessible projects |
| 15 | `jira_get_project_versions` | Get fix versions for a project |
| 16 | `jira_get_issue_dates` | Get dates and status history |
| 17 | `jira_get_issue_sla` | Calculate SLA metrics |
| 18 | `jira_get_issue_development_info` | Get linked PRs/branches/commits |
| 19 | `jira_get_issues_development_info` | Batch get dev info for multiple issues |
| 20 | `jira_batch_get_changelogs` | Batch get changelogs (Cloud only) |
| 21 | `jira_get_issue_proforma_forms` | Get ProForma forms on an issue |
| 22 | `jira_get_proforma_form_details` | Get form details by UUID |

### Jira Write Tools (15)
| # | Tool Name | Description |
|---|-----------|-------------|
| 1 | `jira_create_issue` | Create a new issue |
| 2 | `jira_batch_create_issues` | Batch create issues |
| 3 | `jira_update_issue` | Update issue fields |
| 4 | `jira_delete_issue` | Delete an issue |
| 5 | `jira_add_comment` | Add a comment |
| 6 | `jira_edit_comment` | Edit an existing comment |
| 7 | `jira_add_worklog` | Add a worklog entry |
| 8 | `jira_transition_issue` | Transition issue status |
| 9 | `jira_link_to_epic` | Link issue to an epic |
| 10 | `jira_create_issue_link` | Create link between issues |
| 11 | `jira_create_remote_issue_link` | Create web/Confluence link |
| 12 | `jira_remove_issue_link` | Remove an issue link |
| 13 | `jira_create_sprint` | Create a sprint |
| 14 | `jira_update_sprint` | Update a sprint |
| 15 | `jira_create_version` | Create a fix version |
| 16 | `jira_batch_create_versions` | Batch create fix versions |
| 17 | `jira_update_proforma_form_answers` | Update form answers |

### Confluence Read Tools (10)
| # | Tool Name | Description |
|---|-----------|-------------|
| 1 | `confluence_search` | Search content with CQL |
| 2 | `confluence_get_page` | Get page content |
| 3 | `confluence_get_page_children` | Get child pages/folders |
| 4 | `confluence_get_comments` | Get page comments |
| 5 | `confluence_get_labels` | Get content labels |
| 6 | `confluence_search_user` | Search users with CQL |
| 7 | `confluence_get_page_history` | Get historical page version |
| 8 | `confluence_get_page_views` | Get page view stats (Cloud only) |
| 9 | `confluence_get_attachments` | List attachments on content |
| 10 | `confluence_download_attachment` | Download single attachment |
| 11 | `confluence_download_content_attachments` | Download all attachments for content |

### Confluence Write Tools (7)
| # | Tool Name | Description |
|---|-----------|-------------|
| 1 | `confluence_create_page` | Create a new page |
| 2 | `confluence_update_page` | Update an existing page |
| 3 | `confluence_delete_page` | Delete a page |
| 4 | `confluence_add_label` | Add a label to content |
| 5 | `confluence_add_comment` | Add a comment to a page |
| 6 | `confluence_upload_attachment` | Upload single attachment |
| 7 | `confluence_upload_attachments` | Upload multiple attachments |
| 8 | `confluence_delete_attachment` | Delete an attachment |

**Total: 22 Jira read + 17 Jira write + 11 Confluence read + 8 Confluence write = 58 tools**
