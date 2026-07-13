Synagogue Event RSVP & Reminder Automation (n8n)

Two production-style n8n workflows automating event RSVP intake and reminder
delivery for a synagogue community, built as a portfolio project demonstrating
integration architecture patterns: webhook intake, data validation, capacity
management, scheduled jobs, external API guards, and idempotent writes.

Workflows

1. Event RSVP Intake (workflows/rsvp-intake.json)

Trigger: n8n Form (hosted webform) → runs on each submission.

Form Trigger → Code (normalize & validate) → IF (valid?)
                                              ├─ false → Gmail (graceful failure email)
                                              └─ true  → Sheets (read confirmed RSVPs)
                                                         → Code (capacity math)
                                                         → IF (has_room?)
                                                            ├─ true  → Sheets (append, status=confirmed)
                                                            │          → Gmail (personalized confirmation)
                                                            └─ false → waitlist path

Key patterns:

Normalize at the boundary — trim/lowercase/coerce all form input in a
Code node before it touches anything else; validation failures set a
valid: false flag routed to a friendly email, rather than throwing.

Aggregation vs. per-item execution — capacity math runs once for all
items (summing the attendees column) while pulling the incoming submission
via a cross-node reference ($('Code').first().json).

Graceful failure — every automated rejection gives the human a path
(reply-to instructions), and malformed addresses use On Error: Continue
so one bad row can't kill a run.

2. Daily Reminder Job (workflows/daily-reminders.json)

Trigger: Schedule (daily, 11am America/New_York).

Schedule Trigger → HTTP Request (Hebcal Assur Melacha API) → IF (melacha prohibited? → stop)
  → Sheets (read Events) → Code (find events exactly 3 days out)
  → Sheets (read confirmed RSVPs for event) → Filter (not yet reminded)
  → Gmail (personalized reminder, per item) → Sheets (Update Row: reminded=yes)

Key patterns:

Calendar-aware guard — the workflow checks Hebcal's issur melacha API
before doing anything, so reminders never send on Shabbat or yom tov.
Location-sensitive, correct for two-day diaspora chagim.

Timezone-safe date math — "3 days out" computed with n8n's Luxon $now
in the workflow timezone; naive toISOString() would drift a day after 8pm
Eastern. Event dates stored as plain-text ISO (YYYY-MM-DD) so string
comparison is exact and sortable.

Idempotency — sends are fanned out per item, then each row is marked
reminded=yes (Update Row matched on row_number). Running the workflow
twice sends zero duplicate emails. Ordering is deliberate: send-then-mark
fails toward a duplicate reminder (harmless) rather than a silent miss.

Failure isolation — Gmail uses the error output so a failed send never
reaches the mark-as-reminded step; unmarked rows retry the next day.

Data model (Google Sheets)

RSVPs tab: timestamp | event_id | name | email | phone | attendees | member | dietary | status | reminded

Events tab: event_id | event_name | event_date | capacity | location

event_id joins the two tabs; status (confirmed/waitlist) and
reminded are written exclusively by the workflows.

Lessons learned (real bugs, real fixes)

Silent defaults hide bugs. parseInt(x) || 1 was meant to default blank
party sizes to 1 — but when a form field label changed, it silently turned
every value into 1 and a 1000-person submission passed the capacity check.
Fix: match field names exactly; better, only default when the field is
genuinely absent and flag unparseable input as invalid.

Writer bugs surface at the reader. A quoted literal ('confirmed')
in the append node stored stray apostrophes in the status column; the
reminder workflow's filter then matched nothing, one workflow away from the
actual mistake. Fix at the source, clean the data, and read exact bytes
(the formula bar) rather than trusting rendered cell values.

Date formats are a contract. Sheets auto-reformatting 2026-07-12 to
7-12-2026 broke string matching; forcing the column to plain-text ISO
restored a stable contract between the sheet and the code.

Test both branches. Guards were verified by forcing the rare case —
a fixed Shabbat-afternoon timestamp against the Hebcal API, a temporarily
lowered capacity — rather than waiting for it to occur naturally.

Running it yourself

Import the JSON files from workflows/ into n8n (Workflow → Import from File).

Create Google Sheets and Gmail OAuth2 credentials (n8n walks you through

the Google Cloud OAuth app setup; add yourself as a test user).

Create a spreadsheet with the two tabs above and point the Sheets nodes at it.

Set the workflow timezone (Workflow Settings) — the reminder job's date

math and schedule depend on it.

Activate. The form's production URL goes live with activation.



Notes

Hebcal API is free (CC-BY 4.0) — https://www.hebcal.com attribution appreciated.

Workflow JSON exports contain no credential secrets (n8n stores those
separately), but do contain spreadsheet IDs and email copy — sanitize
before publishing if needed.

