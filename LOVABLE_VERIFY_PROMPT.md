# Lovable prompt: check everything the coding assistant can't check

How to use it:

1. Press **Publish** in Lovable first, so the live site has the latest `main`.
2. Paste everything below the line into Lovable.
3. When Lovable finishes, copy its whole answer from `=== TURVIQ CHECK REPORT START ===` to
   `=== TURVIQ CHECK REPORT END ===` and paste it back to Claude. Claude will tell you what's working, what isn't, and
   what to do next.

---

You have database access, runtime logs, the secrets and the live site. The coding assistant working in the GitHub repo
has none of those. Please **check** each item below and show proof. Pull the latest `main` first.

## Rules for this pass

- **Checks only.** Don't change code, settings or data, except where an item says **DO IF NOT DONE** (those were asked
  for in an earlier prompt). If you find a problem, describe the fix you'd make under `proposed_fix` and don't apply it.
- **Never delete or overwrite any organisation's data.** Real business data was lost once. Read-only queries only on
  `org_state`, `org_archive`, `org_state_backup` and `org_secrets`. Don't run the example-data clear.
- **Never print a secret's value.** For settings, say only `set` or `not set` (and the length where asked).
- **Keep client details private:** don't print client names, phone numbers, emails or addresses. Use counts, ids, or
  initials plus the last 3 digits of a phone. My own email and mobile (the owner's) are fine.
- Test messages (one email, one text, one push) go **only to me**, the owner. Never send a bulk message to clients.
- "My business" means the organisation whose owner login is mine (Jim's Mowing – Grovedale North).
- If you can't check something, say `CANNOT CHECK` and why. Don't guess, and don't mark anything PASS without evidence.

## The checks

**C01 Published build.** Which git commit is the published app built from (hash and date)? It must be `d3489f0`
("Seasonal frequency and on-request services") or later. Does `bun run check` pass on that commit? Give the number of
`ok` lines and any `FAIL` lines. Does `bun run build` pass?

**C02 Database tables.** Confirm these tables exist and give each one's row count: `orgs`, `org_state`,
`org_accounts`, `org_sessions`, `org_secrets`, `org_archive`, `auth_tokens`, `auth_throttle`, `ai_usage`,
`org_state_backup`, `cron_config`. Confirm the `pg_cron` and `pg_net` extensions are enabled.

**C03 Background job schedule.** Show `select jobname, schedule, active from cron.job;`. Expected: `turviq-hourly`,
`*/5 * * * *`, active. Confirm the job's URL is `https://app.turviq.com/api/cron` (the published domain). Confirm
`CRON_SECRET` is set (length only, must be 24 or more) and matches what the job sends (or what's in `cron_config`).
**DO IF NOT DONE:** run `supabase/automatic-jobs.sql` as its comments explain.

**C04 Background job runs.** Show the last 10 rows of `cron.job_run_details` (status, start time, message) and the
status codes of the last 10 responses in `net._http_response` (expect 200; 401 means the secret doesn't match). For
my business, show `updated_at` of `org_secrets` row `automation:lastRun`; it should be under 10 minutes old. Show the
JSON body of one recent `/api/cron` response (its `background` counts and `errors` list), or the matching log lines.

**C05 Older history (90-day archive).** For my business: `org_archive` row counts grouped by `list`, and the value of
`org_secrets` row `archive:lastRun` (when, how many moved, by list). Any `history:` or archive errors in the logs?
Confirm the settings `ARCHIVE_DISABLED` and `ARCHIVE_AFTER_DAYS` are **not set**. In the app, what does Settings →
Data & import → Older history show?

**C06 Data guard.** For my business, show `org_secrets` row `health:counts` (`updated_at` and the stored high-water
counts) next to the current counts in `org_state` for `clients`, `services`, `visits`, `invoices`, `payments`,
`messages`, `quotes`. Has a "records dropped" alert or email gone out? If yes, when, and for which list?

**C07 Backups.** Has `supabase/backups-more-often.sql` been run (is its trigger on `org_state` present)? For my
business: the newest `org_state_backup.saved_at`, how many backups are kept, and the record counts in the newest one.
What are the Supabase project's own backup settings (daily backups, point-in-time recovery)? **DO IF NOT DONE:** run
`supabase/backups-more-often.sql`.

**C08 Email.** Which sender sends my client emails right now: Gmail or Outlook connection, Resend on a verified
domain, or something else? Send **one** test email to me through the same path client emails use (Settings →
Follow-ups → **Send a test**, or the server function behind it). Show the `From` and `Reply-To` it arrived with and the SPF, DKIM and DMARC
results from the received headers. Is `mail.turviq.com` verified in Resend (SPF, DKIM, DMARC)? List any missing DNS
records exactly.

**C09 Texts (SMS).** Are `LOVABLE_API_KEY`, `TWILIO_API_KEY` and `TWILIO_FROM_NUMBER` set? Is the Twilio account a
trial (texts only to verified numbers) or full? Is the from-number allowed to text Australian mobiles, and is the
Australian sender registration done? Send **one** test text to my mobile and show the Twilio message SID, status and
any error code.

**C10 Receipts.** For the last 5 paid invoices in my business: was a receipt created, by which channel, and its
status (`sent` / `failed` + error)? Does every route to Paid (card webhook, Reconciliation link, mark paid, bank match)
now go through one server-side receipt function, as asked in the earlier prompt? Answer yes or no, and name the
function.

**C11 STOP replies.** Is there an incoming-text webhook (for example `/api/twilio-sms`), and is it set on the Twilio
number? When a text fails with Twilio error 21610 (the person replied STOP), is the client marked unsubscribed? Show
one log line or record if it has happened.

**C12 Bulk messages.** In Bulk Comms, use **"Send a test to yourself first"** for one email and one text (to me only).
Did both arrive? From `org_state.campaigns` for my business: how many campaigns, and their statuses. Any `campaigns:`
errors in recent `/api/cron` results? **Don't send a campaign to clients.**

**C13 Unsubscribe page.** Open `https://app.turviq.com/u/test`. Does it return 200 and show the example unsubscribe
page?

**C14 Stripe.** For each webhook, is it registered in Stripe (live mode) with its signing secret stored, and when was
its last successful (2xx) delivery? `/api/billing-webhook`, `/api/connect-webhook`,
`/api/public/payments/webhook?env=live`. Any failed deliveries in the last 7 days? Is Stripe Connect connected for my
business?

**C15 Google Maps and Places.** Is `GOOGLE_MAPS_API_KEY` set? Are **Places API (New)** and the routes/distance API
enabled on that key's Google project? Run one address lookup the way the app's Address check does (for "1 Moorabool
St Geelong VIC") and show whether it returned a match. How many addresses has Settings → Data → Address check looked
at so far, and how many are flagged?

**C16 Push notifications.** Are the `FIREBASE_MESSAGING_*` settings and `FIREBASE_MESSAGING_API_KEY` set? How many
devices are registered for my business? Press **Send a test** under Settings → Integrations → Push notifications and
report the result message.

**C17 Working offline (crew phones).** On the live domain, do `https://app.turviq.com/firebase-messaging-sw.js` and
`https://app.turviq.com/offline-assets.json` both return 200? Give each one's content type and `Cache-Control`
header, and the number of entries in `offline-assets.json`.

**C18 Dependencies.** **DO IF NOT DONE:** switch `xlsx` to `https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz`,
refresh `bun.lock`, and check an Excel import still works. Then show `bun audit`: counts by severity, and whether any
remaining warning is in a package the app uses at runtime (not just build or dev tools).

**C19 Errors in the logs.** In the last 24 hours, list the 10 most frequent server errors (message, count, last
seen), and any server function that returned 500.

**C20 Settings present.** For each, say `set` or `not set` only: `CRON_SECRET`, `SECRETS_ENCRYPTION_KEY`,
`RESEND_API_KEY`, `EMAIL_FROM`, `APP_URL` (also give its value: it's not secret), `BILLING_STRIPE_SECRET_KEY`,
`BILLING_PRICE_ID`, `BILLING_WEBHOOK_SECRET`, `BILLING_ENFORCED`, `STRIPE_CONNECT_CLIENT_ID`, `CONNECT_WEBHOOK_SECRET`,
`LOVABLE_API_KEY`, `TWILIO_API_KEY`, `TWILIO_FROM_NUMBER`, `GOOGLE_MAPS_API_KEY`, `FIREBASE_MESSAGING_API_KEY`,
`ALLOW_SIGNUPS`, `PLATFORM_ADMIN_EMAILS`, `ARCHIVE_DISABLED`, `ARCHIVE_AFTER_DAYS`.

**C21 Time zone and the new seasons.** For my business, what is `business.timezone` in `org_state`? Does the date the
server job treats as "today" match today's date in that zone? On the live site, does Settings → Scheduling show the
new **Seasons** section? Are there any plan-ahead or `seasons` errors in recent `/api/cron` results? Don't change my
services or seasons.

## Reply in exactly this format

Reply with one block, in this exact shape, and nothing after the END line. Keep each `evidence` to the key lines
(trim long output, at most about 8 lines). `status` is one of `PASS`, `FAIL`, `PARTIAL` or `CANNOT CHECK`.

```
=== TURVIQ CHECK REPORT START ===
build: <commit hash> <commit date> | checked: <date and time, with time zone>

[C01] Published build
status: PASS | FAIL | PARTIAL | CANNOT CHECK
evidence: <exact query, log line or response, trimmed>
changed: <nothing | what you changed (only for DO IF NOT DONE items)>
proposed_fix: <none | the fix you'd make>
owner_action: <none | exactly what I must do>

[C02] Database tables
status: ...
evidence: ...
changed: ...
proposed_fix: ...
owner_action: ...

(one block for every item, C01 to C21, in order, none skipped)

SUMMARY
pass: <ids>
fail: <ids>
partial: <ids>
cannot_check: <ids>
owner_to_do:
- <each thing I must do myself, in order of importance>
=== TURVIQ CHECK REPORT END ===
```
