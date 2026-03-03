# Application Reviewer Tool — Spec

## Motivation

We have a Google Sheet where rows are applicants and columns are their answers to application questions. Some columns contain links to PDFs. Additional columns exist (or will be added) for reviewer ratings and comments.

Viewing and reviewing applications in the spreadsheet is painful: you have to scroll horizontally across dozens of columns, there's no way to focus on one applicant at a time, and PDFs are just raw links. We want a dedicated review interface that uses the Google Sheet as its backend database — no separate server, no separate database. The sheet remains the source of truth.

There are ~3 reviewers. They are not all technical users. The tool must require zero local setup — just a URL they click.

## Architecture

**Single static HTML file.** No backend server. Host on GitHub Pages or any static file host.

All communication with Google Sheets happens client-side via the Google Sheets API, authenticated with Google OAuth (Google Identity Services). Each reviewer signs in with their Google account. The browser talks directly to the Sheets API using the user's own OAuth token.

### Why no backend

- 3 reviewers, ~300 applicants, ~1MB of data — well within client-side Sheets API limits (60 writes/min, 300 reads/min per user).
- OAuth with user-scoped tokens means no API keys to manage or expose.
- Hosting is free and trivial (GitHub Pages, etc.).
- Each reviewer's writes go through their own auth token, so attribution is automatic.

## Google Cloud Setup (done manually, not by the app)

The developer (me) will:
1. Create a GCP project.
2. Enable the Google Sheets API.
3. Configure the OAuth consent screen (Internal if all reviewers are in the same Workspace org; External/unverified if using personal Gmail — reviewers will see a "Google hasn't verified this app" warning they click through once).
4. Create an OAuth 2.0 Client ID (Web application type) with the hosting URL as an authorized JavaScript origin.
5. The Client ID will be hardcoded in the HTML file (this is fine — it's not a secret, it's scoped to the authorized origins).

## Sheet Configuration via URL Parameter

The app accepts the Google Sheet ID via a URL query parameter:

```
https://your-host.github.io/reviewer.html?sheet=SPREADSHEET_ID
```

The `sheet` parameter is the long alphanumeric string from the Google Sheets URL (`https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/edit`).

If the parameter is missing, the app shows an input field prompting the user to paste a Google Sheets URL. The app extracts the ID from the URL and redirects to the parameterized version (or just uses it directly — either way, the result should be a shareable URL with the sheet ID baked in).

This means one deployment of the app works for any sheet that follows the column conventions below.

## Sheet Schema / Column Conventions

The app auto-detects everything from the **first row (headers)**. No configuration file. No config sheet.

### Read-only columns (application data)

Any column header that does NOT match the editable column pattern (see below) is treated as read-only application data. The column header becomes the label/question, and the cell value becomes the answer.

### Editable columns (reviewer fields)

A column is editable if its header matches this pattern:

```
Column Name [email@example.com]
```

Where:
- `Column Name` is the display label (e.g., "Rating", "Comments").
- The part inside square brackets is the **email address** of the reviewer who owns that column.
- Only the logged-in user whose Google OAuth email matches the bracketed email can edit that column.
- Everyone can **read** all columns.

Special case — shared editable columns:

```
Column Name [*]
```

A `[*]` wildcard means any authenticated user can edit the column.

### PDF / link auto-detection

The app does NOT require columns to be pre-labeled as "PDF" or "link." Instead, for every cell value, the app checks if it looks like a URL (starts with `http://` or `https://`). If it is a URL, the app attempts to render it inline:

- **Google Drive PDF links**: Use the Google Docs viewer embed or Drive API to render inline. The URL pattern `https://drive.google.com/file/d/{FILE_ID}/...` can be converted to an embed: `https://drive.google.com/file/d/{FILE_ID}/preview`.
- **Other PDF URLs**: Use `<iframe>` with the URL directly, or fall back to `https://docs.google.com/viewer?url={encoded_url}&embedded=true`.
- **Other URLs**: Render as a clickable link. Optionally attempt an iframe embed; if it fails (X-Frame-Options, CORS), just show the link.

The goal is best-effort inline rendering with graceful fallback to "open in new tab" links.

## User Interface

### Layout

Full-screen, one applicant at a time. Navigation to go to the next/previous applicant, plus a way to jump to a specific applicant (dropdown, search, or number input).

### Applicant View

For the current applicant (one row in the sheet), display all columns vertically as a scrollable page:

```
Question / Column Header 1
Answer / Cell Value 1

Question / Column Header 2  
Answer / Cell Value 2

[If cell value is a URL: rendered inline or as a link]

...

--- Reviewer Section ---

Rating [you@email.com]        ← editable text box (this is your column)
Rating [other@email.com]      ← read-only display (someone else's column)

Comments [you@email.com]      ← editable text box
Comments [other@email.com]    ← read-only display

Status [*]                    ← editable by anyone
```

The reviewer section should visually distinguish:
- Your editable fields (text inputs/textareas).
- Other reviewers' fields (displayed as read-only text, maybe slightly grayed out).
- Shared editable fields (editable by anyone, visually distinct from personal fields).

### Saving

When the user edits a field and moves away (blur) or explicitly saves, the app writes the value to the corresponding cell in the Google Sheet via the Sheets API. A write queue ensures requests are serialized (one at a time) to avoid rate limit issues and ordering problems. Show a small save indicator (e.g., "Saving..." → "Saved" or a checkmark).

### Data Loading

On initial load after auth, read the entire sheet into memory (~300 rows, expect 3-5MB of JSON from the API). Cache in memory. Navigate between applicants using the in-memory data. Provide a "Refresh" button to re-fetch the sheet (to pick up other reviewers' changes). No need for live/polling updates — a manual refresh is fine.

If performance is an issue with the full sheet load, consider reading just the header row first, then loading applicant data in batches. But try the simple approach first.

## Authentication Flow

1. Page loads. Google Identity Services library is loaded.
2. User clicks "Sign in with Google."
3. OAuth popup/redirect. Requested scopes: `https://www.googleapis.com/auth/spreadsheets` (read/write to sheets the user has access to).
4. On success, the app stores the access token in memory (NOT localStorage — tokens are short-lived anyway) and fetches the user's email from the token/userinfo endpoint.
5. The user's email is matched against column headers to determine which columns are editable.
6. If the token expires during a session, re-prompt for auth.

## Concurrency Model

With ~3 reviewers working simultaneously:
- Each reviewer writes to their own columns (different cells), so Sheets handles this fine at the storage level.
- The per-user write queue in the browser serializes that user's own writes to avoid race conditions with themselves (e.g., rapid edits).
- No cross-user coordination needed. If two users both edit a `[*]` column on the same row, last write wins. This is acceptable for 3 users.

## Technical Constraints

- **Single HTML file.** All JS and CSS inline. External dependencies loaded from CDNs only (Google's API libraries, possibly a PDF viewer library).
- **No build step.** No React, no bundler. Vanilla JS or a CDN-loaded lightweight framework if needed.
- **No localStorage/sessionStorage for critical state.** The sheet is the source of truth. Browser state is ephemeral.
- **OAuth client ID is hardcoded.** This is standard for client-side OAuth apps.
- **Works in modern browsers (Chrome, Firefox, Safari, Edge).** No IE support needed.

## Summary of Column Header Format

| Header | Behavior |
|---|---|
| `Anything without brackets` | Read-only for all users. Displayed as question/answer. |
| `Label [user@email.com]` | Editable text box, only for the user whose OAuth email matches. Read-only for everyone else. Label is displayed. |
| `Label [*]` | Editable text box for any authenticated user. |

## Filtering

Each column header in the applicant view should have a small filter icon next to it. Clicking the icon opens a filter UI for that column. The filter determines which applicants are shown when navigating with the previous/next arrows — filtered-out applicants are skipped.

### Filter behavior based on distinct value count

The app counts the distinct non-empty values in that column across all applicants:

- **20 or fewer distinct values**: Show a dropdown/checklist of all values. The user can select one or more values to filter by. Include an option for "(empty)" to match rows where that column is blank.
- **More than 20 distinct values**: Show a text search box. The user types a substring and only applicants whose value in that column contains that substring are shown. Case-insensitive.

### Navigation with active filters

When one or more filters are active:

- The previous/next arrows skip applicants that don't match ALL active filters (AND logic).
- Show a visible indicator of how many filters are active and how many applicants match (e.g., "Showing 47 of 300 applicants").
- Provide a "Clear all filters" button.

### Multiple simultaneous filters

The user can set filters on multiple columns at once. An applicant must match all active filters to be included (AND logic across columns).

## Out of Scope (for now)

- Real-time collaboration / live updates (manual refresh is fine).
- User management or admin panel.
- Custom rating widgets (everything is a text box).
- Sorting applicants.
- Multiple sheets/tabs within the same spreadsheet (just use the first sheet, or allow specifying a tab name as another URL parameter if needed).
- Offline support.

