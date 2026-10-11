# Class Observations

This repo holds static assets (the Berkshire School seal) for the Class Observations
Apps Script app. The code itself lives in Google Apps Script, not here:

- **Main app**: container-bound to the "Class Visits" Google Sheet (tabs: Departments,
  Terms, Sections, Chairs, Faculty, Visits, Notes, Goals, PD, Comms). Config lives in
  `CONFIG` in its `Code.gs`, including `CONFIG.SYNC_APP_URL`.
- **Sync companion**: standalone script "Class Observations Sync". It runs as the user
  accessing it, domain-only (berkshireschool.org), reads that chair's own calendar for
  events whose title starts with their keyword, and POSTs only those events with the
  chair's OAuth token to `MAIN_APP_URL`. It never touches the Sheet.

Script time zone is `America/New_York`.

## Deploy habit

Always update the existing deployment. Never create a new one for an update.

1. Deploy > Manage deployments > edit (pencil) the existing web app deployment.
2. Version: **New version**, then Deploy.

This keeps each `/exec` URL stable. The two apps point at each other
(`CONFIG.SYNC_APP_URL` in the main app, `MAIN_APP_URL` in Sync), so a "New deployment"
would mint a new URL and silently break that link. If a new deployment is ever truly
needed, update both URLs and redeploy both apps.

## Terms date format

`Start` and `End` on the Terms tab are **plain text in `yyyy-MM-dd`** (e.g. `2026-09-14`),
not Sheets Date values. This avoids time zone shifts when dates cross between Sheets and
script. Parse them as local dates in `America/New_York`, and write any new dates the same
way (format with `Utilities.formatDate(d, tz, 'yyyy-MM-dd')`). A blank Start/End means
the term's dates are not set yet (e.g. Spring).

Other tabs follow the same convention: Visits `Date` is `yyyy-MM-dd`, `Start`/`End` are
`HH:mm`, and visit keys are `<eventId>|yyyy-MM-ddTHH:mm`.

## Chairs tab security model

Columns: `Email | Name | Department | IsDean | Title | Keyword`.

- The main app runs as the owner. Access is an **allowlist**: the signed-in user's email
  must match a row in Chairs, or the app denies access.
- A chair sees only their own `Department`'s teachers, visits, and notes.
- `IsDean = TRUE` sees all departments (deans have no Department).
- `Keyword` is that chair's calendar title prefix used by Sync (default `Class Visit`).
- Only the owner edits the Chairs tab. Never add a code path that lets a user change
  their own row, and never trust a department or email sent from the client; always
  derive identity server-side from the signed-in session (or the verified OAuth token
  that Sync sends).
