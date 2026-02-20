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
