---
description: Generate executive-level project insights from Jira, Confluence, and local documents
argument-hint: --jira PROJECT_KEY --spaces SPACE_KEY [--local-docs SUBFOLDER] [--reprocess-docs] [--search "keywords"] [--target-date YYYY-MM-DD] [--stale-days N]
---

You are generating a project insights report. Use the project-insights skill for full guidance.

Parse the user's arguments:
- `--jira` (required): Comma-separated Jira project key(s)
- `--spaces` (required): Comma-separated Confluence space key(s)
- `--local-docs` (optional): Subfolder name under `external-docs/`. Enables local document discovery. Files (.docx, .pdf, .md, .txt) in this directory are scanned for planning content.
- `--reprocess-docs` (optional): Force re-extraction of all local docs, ignoring the processing cache.
- `--search` (optional): Free-text search keywords
- `--target-date` (optional): User-supplied deadline in YYYY-MM-DD format
- `--stale-days` (optional, default 14): Days of inactivity before flagging as stale

If required arguments are missing, ask the user for them before proceeding.

## MCP Health Check (mandatory — run before anything else)

After parsing arguments, you MUST verify MCP connectivity before proceeding. Do NOT skip this step. Do NOT proceed to the skill if any required check fails.

### 1. Atlassian MCP (always required)

Call `mcp__atlassian__getAccessibleAtlassianResources` (no parameters). This must return a list of accessible sites including cloud ID `0f350409-7469-4766-9759-84b56e79f2f5`.

- **If the call succeeds** and the cloud ID is present: Atlassian MCP is healthy. Continue.
- **If the call fails, times out, or the cloud ID is missing**: STOP. Tell the user:
  > Atlassian MCP health check failed. Please run `/mcp`, complete the OAuth flow for the Atlassian server, and try again.

  Do NOT proceed to the skill. Do NOT attempt retries.

### 2. All checks passed

Only after the Atlassian MCP health check passes, proceed to follow the project-insights skill's 4-pass process exactly.
