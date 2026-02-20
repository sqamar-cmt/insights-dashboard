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

Then follow the project-insights skill's 4-pass process exactly.
