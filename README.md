# Application Reviewer

A single-file web app for reviewing applications stored in a Google Sheet. No backend — talks directly to the Sheets API via OAuth.

## Setup

1. Create a GCP project, enable the Sheets API, and create an OAuth Client ID
2. Replace `YOUR_CLIENT_ID.apps.googleusercontent.com` in `index.html` with your Client ID
3. Host the file anywhere (GitHub Pages, etc.)

## Usage

```
https://your-host/index.html?sheet=SPREADSHEET_ID
```

Optional: `&tab=SheetTabName` to target a specific tab.

## Sheet Column Conventions

| Header format | Behavior |
|---|---|
| `Anything` | Read-only for all users |
| `Label [user@email.com]` | Editable only by that user |
| `Label [*]` | Editable by any signed-in user |

URLs in cells are auto-detected. Google Drive links and PDFs are embedded inline.
