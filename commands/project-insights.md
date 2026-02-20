---
description: Generate executive-level project insights from Jira, Confluence, and Google Drive
argument-hint: --jira PROJECT_KEY --spaces SPACE_KEY [--drive-email EMAIL] [--drive-folders IDS] [--search "keywords"] [--target-date YYYY-MM-DD] [--stale-days N]
---

You are generating a project insights report. Use the project-insights skill for full guidance.

Parse the user's arguments:
- `--jira` (required): Comma-separated Jira project key(s)
- `--spaces` (required): Comma-separated Confluence space key(s)
- `--drive-email` (optional): Google account email for Drive access. Enables Google Drive discovery.
- `--drive-folders` (optional): Comma-separated Google Drive folder IDs to search within. Requires `--drive-email`.
- `--search` (optional): Free-text search keywords
- `--target-date` (optional): User-supplied deadline in YYYY-MM-DD format
- `--stale-days` (optional, default 14): Days of inactivity before flagging as stale

If required arguments are missing, ask the user for them before proceeding.

If `--drive-folders` is provided without `--drive-email`, ask the user for their Google account email.

## MCP Health Check (mandatory — run before anything else)

After parsing arguments, you MUST verify MCP connectivity before proceeding. Do NOT skip this step. Do NOT proceed to the skill if any required check fails.

### 1. Atlassian MCP (always required)

Call `mcp__atlassian__getAccessibleAtlassianResources` (no parameters). This must return a list of accessible sites including cloud ID `0f350409-7469-4766-9759-84b56e79f2f5`.

- **If the call succeeds** and the cloud ID is present: Atlassian MCP is healthy. Continue.
- **If the call fails, times out, or the cloud ID is missing**: STOP. Tell the user:
  > Atlassian MCP health check failed. Please run `/mcp`, complete the OAuth flow for the Atlassian server, and try again.

  Do NOT proceed to the skill. Do NOT attempt retries.

### 2. Google Workspace MCP (only if `--drive-email` was provided)

Call `mcp__google-workspace__search_drive_files` with `user_google_email` set to the `--drive-email` value, `query: "trashed = false"`, and `page_size: 1`.

- **If the call succeeds** (returns results or an empty list): Google Workspace MCP is healthy. Continue.
- **If the call fails or times out**: STOP. Tell the user:
  > Google Workspace MCP health check failed. Please run `/mcp`, complete the OAuth flow for the google-workspace server, and try again.

  Do NOT proceed to the skill. Do NOT attempt retries.

### 3. All checks passed

Only after all required MCP health checks pass, proceed to follow the project-insights skill's 4-pass process exactly.
