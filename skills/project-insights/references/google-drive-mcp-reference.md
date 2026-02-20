# Google Workspace MCP Tool Reference

> **Source:** [taylorwilsdon/google_workspace_mcp](https://github.com/taylorwilsdon/google_workspace_mcp) (community MCP server)
> **Package:** `workspace-mcp` on PyPI
> **Install:** `uvx workspace-mcp`
> **Last updated:** 2026-02-20
> **Auth:** OAuth 2.0 (requires `GOOGLE_OAUTH_CLIENT_ID` + `GOOGLE_OAUTH_CLIENT_SECRET`)

---

## Authentication

### OAuth 2.0 Setup

1. Create a Google Cloud project at https://console.cloud.google.com/
2. Enable the Google Drive API and Google Docs API
3. Create OAuth 2.0 credentials (Desktop application type)
4. Set environment variables:
   ```bash
   export GOOGLE_OAUTH_CLIENT_ID=your_client_id
   export GOOGLE_OAUTH_CLIENT_SECRET=your_client_secret
   ```
5. On first run, the server will open a browser for OAuth consent. Grant access to Drive and Docs scopes.
6. Tokens are cached locally for subsequent runs.

### Required Scopes

- `https://www.googleapis.com/auth/drive.readonly` -- Search and read Drive files
- `https://www.googleapis.com/auth/documents.readonly` -- Read Google Docs content

---

## Drive Tools (Used by Discovery Agent)

### search_drive_files

- **Description:** Search Google Drive files by name, content, or structured Drive API query.
- **Parameters:**
  - `user_google_email` (string, required): Google account email for Drive access.
  - `query` (string, required): Search query. Supports Google Drive query syntax:
    - Name search: `name contains 'PRD'`
    - Full-text: `fullText contains 'data pipeline'`
    - MIME type filter: `mimeType = 'application/vnd.google-apps.document'`
    - Folder scope: `'<folder_id>' in parents`
    - Combined: `name contains 'PRD' and mimeType = 'application/vnd.google-apps.document'`
    - Modified date: `modifiedTime > '2025-01-01T00:00:00'`
  - `page_size` (int, optional, default: 10): Maximum number of results (1-100).
- **Returns:** JSON array of file objects with `id`, `name`, `mimeType`, `modifiedTime`, `webViewLink`, `owners`, `parents`.
- **Pagination:** Limited by `page_size` parameter. No cursor-based pagination exposed. Use multiple targeted queries for broader coverage.
- **Notes:** Google Drive search is eventually consistent -- recently created or modified files may not appear immediately.

#### Query Syntax Reference

| Pattern | Example | Description |
|---------|---------|-------------|
| Name contains | `name contains 'PRD'` | Files with "PRD" in the name |
| Full-text search | `fullText contains 'migration'` | Files containing "migration" in body |
| MIME type | `mimeType = 'application/vnd.google-apps.document'` | Google Docs only |
| Folder | `'1abc2def' in parents` | Files in a specific folder |
| Modified after | `modifiedTime > '2025-06-01T00:00:00'` | Recently modified files |
| Trashed | `trashed = false` | Exclude trashed files (default) |
| Combine | `name contains 'spec' and mimeType = 'application/vnd.google-apps.document'` | AND conditions |

Common MIME types:
- Google Docs: `application/vnd.google-apps.document`
- Google Sheets: `application/vnd.google-apps.spreadsheet`
- Google Slides: `application/vnd.google-apps.presentation`
- PDF: `application/pdf`
- Word: `application/vnd.openxmlformats-officedocument.wordprocessingml.document`

---

### get_drive_file_content

- **Description:** Read the content of any Drive file as text. Handles Google Docs, Sheets, and uploaded files (.docx, .txt, etc.) by exporting or converting to text.
- **Parameters:**
  - `user_google_email` (string, required): Google account email.
  - `file_id` (string, required): Drive file ID (from search results or URL).
- **Returns:** Text content of the file. For Google Docs, returns the document text. For Sheets, returns CSV-like text. For binary files, returns extracted text where possible.
- **Pagination:** N/A (returns full content)
- **Notes:** For Google Docs specifically, prefer `get_doc_content` which preserves more structure. This tool is a general-purpose fallback for non-Docs files.

---

### get_doc_content

- **Description:** Read the content of a Google Doc with structural information preserved.
- **Parameters:**
  - `user_google_email` (string, required): Google account email.
  - `document_id` (string, required): Google Doc document ID (same as Drive file ID for Docs).
- **Returns:** Document content as structured text with headings, lists, and tables preserved.
- **Pagination:** N/A (returns full content)
- **Notes:** Only works for files with MIME type `application/vnd.google-apps.document`. For other file types, use `get_drive_file_content`.

---

### list_drive_items

- **Description:** List contents of a Google Drive folder.
- **Parameters:**
  - `user_google_email` (string, required): Google account email.
  - `folder_id` (string, required): Drive folder ID. Use `root` for the root of My Drive.
  - `page_size` (int, optional, default: 10): Maximum number of items to return (1-100).
- **Returns:** JSON array of file/folder objects with `id`, `name`, `mimeType`, `modifiedTime`, `webViewLink`.
- **Pagination:** Limited by `page_size`. No cursor-based pagination exposed.
- **Notes:** Returns both files and subfolders. Use MIME type `application/vnd.google-apps.folder` to identify subfolders.

---

## Error Handling

### Authentication Errors
- **Cause:** Missing or expired OAuth tokens, insufficient scopes
- **Shape:** Error message indicating auth failure
- **Resolution:** Re-run the OAuth flow by clearing cached tokens and restarting the MCP server

### Not Found Errors
- **Cause:** Invalid file ID, file deleted, or no access
- **Shape:** Error message with file ID context

### Rate Limiting
- **Source:** Google Drive API quotas (default: 20,000 queries/100 seconds per project)
- **Behavior:** HTTP 429 responses
- **Resolution:** Space out queries; use targeted searches rather than broad scans

### Permission Errors
- **Cause:** File not shared with the authenticated user
- **Shape:** Error indicating insufficient permissions
- **Resolution:** Ensure the user has at least Viewer access to the files being queried

---

## Tool Index

| # | Tool Name | Description | Used by Agent |
|---|-----------|-------------|---------------|
| 1 | `search_drive_files` | Search Drive files by query | Yes |
| 2 | `get_drive_file_content` | Read any file as text | Yes (fallback) |
| 3 | `get_doc_content` | Read Google Doc content | Yes (preferred for Docs) |
| 4 | `list_drive_items` | List folder contents | Yes (when folder IDs provided) |

**Total: 4 tools (all read-only)**
