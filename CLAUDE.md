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
("Deploy > New deployment" in the `Code.gs` header applies to the very first deploy only.)

1. Deploy > Manage deployments > edit (pencil) the existing web app deployment.
2. Version: **New version**, then Deploy.

This keeps each `/exec` URL stable. The two apps point at each other
(`CONFIG.SYNC_APP_URL` in the main app, `MAIN_APP_URL` in Sync), so a "New deployment"
would mint a new URL and silently break that link. If a new deployment is ever truly
needed, update both URLs and redeploy both apps.

## Terms date format

`Start` and `End` on the Terms tab are **plain text in `yyyy-MM-dd`** (e.g. `2026-09-14`),
not Sheets Date values. `setup()` sets every column to the `@` (text) number format, and
rows are read with `getDisplayValues()`, so dates stay strings end to end and compare
correctly as strings. Parse with `parseDate_()` (a local date, no time zone shift) and
write with `fmtDate_()`. A blank Start/End means the term's dates are not set yet
(e.g. Spring), and code must skip undated terms. The Terms tab is the source of truth;
`CONFIG.TERM_START`/`TERM_END` are only a fallback for when no term is dated.

Other tabs follow the same convention: Visits `Date` is `yyyy-MM-dd`, `Start`/`End` are
`HH:mm`, and visit keys are `<eventId>|yyyy-MM-ddTHH:mm`.

## Chairs tab security model

Columns: `Email | Name | Department | IsDean | Title | Keyword`.

The main app runs as the deployer (Dan), so the script itself can read and write the
whole Sheet. All access control is in code, driven by the Chairs tab:

- **Allowlist.** `currentUser_()` matches `Session.getActiveUser().getEmail()` against
  Chairs. No row means role `none`, and `requireUser_()` throws. Every `api*` function
  must start with `requireUser_()`.
- **Roles.** `IsDean = TRUE` is role `dean`; anything else is role `chair`.
- **Department scope.** `deptFilter_(u)`: a chair sees only rows whose `Department`
  equals their own. A dean sees every department.
- **Notes are private by default.** `noteVisible_()`: the author always sees their own
  notes. A dean also sees a note when its `VisibleTo` list contains that dean's `Title`.
  Other chairs never see each other's notes, even within a department.
- **Deans are read-only.** Every write API checks `u.role === 'chair'`. Sync (`doPost`)
  also rejects dean rows.
- **Keyword** is the chair's calendar title prefix (default `Class Visit`). The only
  in-app write to Chairs is `apiSaveSettings`, which changes the caller's own Keyword.
  Everything else in Chairs is maintained by hand in the Sheet by the owner. Don't widen
  that write.
- **Identity is server-side only.** Never trust an email, role, or department sent from
  the page. In `doPost`, identity comes from the chair's OAuth token, checked through
  Google's tokeninfo endpoint, and must be on the `berkshireschool.org` domain.
- Department Drive folders (`ensureDeptFolder_`) are shared as editor with that
  department's non-dean chairs.
