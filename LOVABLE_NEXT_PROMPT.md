# Lovable prompt: fix SMS receipts, email sending, and finish the open items

Paste everything below the line into Lovable.

---

You have database access, runtime logs and secrets. I don't, and neither does the coding assistant that has been working in the GitHub repo, so please use them. Pull the latest `main` first. A lot changed recently, including server-side background jobs (`src/lib/background.server.ts`, `src/lib/cron.server.ts`), branded email (`src/lib/businessMail.server.ts`, `src/lib/emailBrand.ts`), and the receipt/SMS code in `src/data/store.tsx` and `src/lib/twilio.server.ts`.

Work through the sections in order. For every item, **show me what you found in the logs/DB before and after the fix**, and tell me plainly if something can't be done and what I must do myself.

## 1. SMS receipts are not being sent (top priority)

**Symptom:** I paid an invoice with a Stripe *test* card, then linked the payment manually on the Reconciliation page. The invoice became Paid (INV-18321, $170.00, client Grace van der Merwe) but **no SMS receipt arrived**. The client has a mobile number. This happened before my latest publish, so please re-test on the current build first.

**Do this:**
1. Check the runtime logs and the `messages` data in `org_state` for that invoice and client. Did a receipt message get created at all? Was it `Sent`, `Failed` or never created? Tell me exactly where the chain broke.
2. Known silent exit points to check, and make each one **visible** (a toast and/or an admin alert saying "Receipt not sent because …") instead of returning silently:
   - `sendReceipt` in `src/data/store.tsx` (around line 2094) returns silently when `state.automationsEnabled` is false. `settleCardReceipt` in `src/lib/cron.server.ts` does the same ("no receipt: automatic messages are off"). Check my master switch.
   - `client.invoiceReceipts` is off, or the contact's **Receipts** tick isn't set (`planContactMessages`).
   - `receiptSentAt` / `receiptsSent` already set, so it is treated as already sent.
   - The client's phone number format (needs +61 normalisation) and the lookup in `sendSmsFn` ("number must belong to a client").
   - The browser `SmsBridge` (`crm-sms-out` event) only works while a tab is open. The cron `dispatch` sends SMS from `m.to.label.split(" · ").pop()`, which is fragile if the label has no phone.
3. **Make receipts reliable and independent of an open browser.** When an invoice becomes Paid by any route (Stripe live webhook, manual reconciliation link, "mark paid", bank-feed match), a single **server-side** function should create and send the receipt over the channels enabled for that client (email and/or SMS), once, with idempotency. Reuse `settleCardReceipt`. The Reconciliation link-payment action must call it too.
4. Check Twilio: `LOVABLE_API_KEY`, `TWILIO_API_KEY`, `TWILIO_FROM_NUMBER` are set. Is the account a **trial** that can only text verified numbers? Is the from number SMS-capable for Australia (alphanumeric sender IDs and AU number registration rules)? Send one test SMS to my mobile and show me the Twilio response or error.
5. Add a small **"Send test SMS"** and **"Resend receipt"** button on the invoice (owner/admin only) that shows the exact provider error on failure.
6. Stripe test payments are intentionally never matched, marked paid or receipted automatically (`recordInvoiceCardPayment` skips sandbox payments). Add an owner-only setting, "Treat Stripe test payments as real (for testing only)", off by default and with a visible warning, so I can test the whole flow end to end without spending real money.

## 2. Emails are not coming from my business address and land in junk

**Symptom:** Client emails arrive as "Jim's Mowing – Grovedale North via TURVIQ <noreply@mail.turviq.com>" and land in spam. They must come from **matthew.vandermerwe@jimsmowing.net**.

Root cause so far: the server tries SMTP to `mail.jimsmowing.net` and fails with `proxy request failed, cannot connect to the specified address`. The runtime can't open raw outbound SMTP sockets. So it falls back to Resend from noreply@mail.turviq.com with reply-to set to my inbox.

**Do this:**
1. Confirm in the logs why SMTP fails (blocked port 465/587 from the worker runtime). If raw SMTP can't work here, don't keep retrying it. Pick the best working option and tell me what you chose:
   - **Preferred:** Gmail or Microsoft 365 / Outlook OAuth through the existing `integrations.server.ts emailSend`, if my mailbox is on one of those. Tell me if it is not.
   - Or send via an **HTTP-based sender** (Resend or similar) using a domain I own and have verified. If I have to verify a domain, tell me exactly which DNS records to add (SPF, DKIM, DMARC) and where. The From address must be on that verified domain, with Reply-To set to my business mailbox.
   - Or an SMTP relay that works over HTTPS or an allowed port.
2. Verify `mail.turviq.com` in Resend (SPF, DKIM, DMARC alignment) so TURVIQ-sent mail stops going to junk. Check the DNS and tell me what's missing.
3. Remove the hard-coded Jim's Mowing constants in `src/lib/mailbox.server.ts` (`SMTP_HOST = "mail.jimsmowing.net"`, `MAILBOX_FROM_NAME`, `SIGNATURE_PHONE`). They must come from each business's own settings, because this is a multi-tenant product (TURVIQ) and other businesses must not send as Jim's Mowing.
4. Branded email: check the HTML renders well in Gmail, Outlook and mobile, with the logo, brand colour and signature, and with plain-text alternatives. Send me a test to my own address and show the headers (SPF/DKIM/DMARC pass) so we know it won't be junked.
5. The business inbox (IMAP) pull isn't automated yet. If it's feasible in this runtime, do it. If not, tell me what's blocking.
6. An invoice must **never** show "Sent" unless a message was actually delivered. The app now reverts to "Awaiting Approval" when every message for it fails (`markMessageFailed`). Check that this holds for the server dispatch path as well, and that the real provider error is shown to me, not a generic one.

## 3. Verify the background job is really running

Many features now depend on the server job `/api/cron` (every 5 minutes): the overdue flip, bank feed pull, driver alerts, geocoding, plan-ahead visits, Xero/QuickBooks invoice push, inbox pull, licence reminders, signed-quote copies, and reminders/follow-ups.
1. Check that `supabase/automatic-jobs.sql` has actually been run, `pg_cron` and `pg_net` are enabled, `CRON_SECRET` is set and matches, and the job's URL points at the **published** app domain (`https://app.turviq.com/api/cron`).
2. Show me the last 5 runs and their results (`CronResult.background`) and any errors.
3. Add a "Background jobs" status card in Settings that shows the last run time and any errors, with a red warning if it hasn't run in the last 15 minutes.

## 4. Live Stripe payments

1. Confirm the live webhook `/api/public/payments/webhook?env=live` is registered in Stripe, the signing secret is stored, and recent events are being received. Check the logs.
2. Walk me through making one small **real** payment, and confirm it marks the invoice paid and sends both the email and the SMS receipt.

## 5. Other things still open

- **Travel times in routing:** the route optimiser's driving-time cache lives in browser localStorage, so server-side planning uses straight-line estimates. Move the cache to the database or server so background planning uses real driving times.
- **Vehicle reports "All time"** are capped at 12 months. Remove the cap or paginate.
- **Necessity Housing:** the importer doesn't link the 18 sites to their parent account. Finish the umbrella-account import so sites link to the parent and invoices roll up like the old CRM.
- **Landing page error** on the quotes/invoices pages (turviq.com / landing repo `lazerlawns/turviqlp`): I still haven't captured the exact message. If you can see it in the logs, fix it. Otherwise tell me where to look.
- **Xero payments:** pulling Xero payments only gives suggestions. Make it auto-reconcile when the match is unambiguous (amount plus invoice number), and keep the manual review for anything else.
- **Tests:** `src/lib/dataGuard.test.ts` imports `vitest`, which isn't installed and isn't run by `bun run check`. Either add it to the check suite or convert it to the same style as the other `scripts/check-*.ts`.
- **Lockfile:** I upgraded `jspdf` to ^4.2.1. Please refresh `bun.lock` from the registry on your side.
- **Google Places API (New)** must be enabled for address verification. Check it, then run Settings → Data → Address check.
- **Push notifications (Firebase):** check device registration works for employees, so driver alerts reach them.
- **Twilio voice webhook:** check the URL from Settings → Integrations is set on the Twilio number.

## 5b. New since the last prompt (October 9)

- **The 5-minute background job now does more**, so please confirm it really runs every 5 minutes (section 3): bulk
  messages keep sending from it when nobody has the page open, and the nightly move of history older than 90 days
  (1am to 6am) runs from it.
- **Older history:** you've already created the `org_archive` table, thank you. After tonight, check
  Settings → Data & import → Older history shows records moved, and that the logs show no `history:` errors.
- **Spreadsheet library security fix:** `xlsx` 0.18.5 (used to import spreadsheets) has two known security problems.
  The fixed version isn't on npm; switch the dependency to SheetJS's own build:
  `"xlsx": "https://cdn.sheetjs.com/xlsx-0.20.3/xlsx-0.20.3.tgz"`, refresh `bun.lock`, and check an Excel import still works.
- **Other dependency warnings** (`bun audit`): the rest are in build and development tools (vite, eslint, esbuild,
  devtools), not in the app people use. Run `bun update` for those when convenient and check the build still passes.
- **Texts and STOP:** replies of STOP are recorded when Twilio refuses the next text (error 21610). The app has no
  incoming-text webhook yet: if you can, add one (a route like `/api/twilio-sms` that records client replies in their
  chat and marks "STOP" as unsubscribed via `markUnsubscribed` in `src/lib/campaign.server.ts`) and set it on the Twilio
  number. Also confirm the Australian number is registered for sending (sender ID rules).

## 6. Data safety (please double check)

Real business data was lost once. Do not delete or overwrite any organisation data. Do not run the example-data clear on my behalf. Confirm the data guard and the hourly backups (`supabase/backups-more-often.sql`) are running, and show me the newest backup's timestamp and row counts for my account.

## What I will do myself (tell me if anything is missing from this list)

- Press **Publish** after your changes.
- Re-enter my business details, logo and terms in Settings (the live client portal still shows sample "Matt / Greenline" details), then use **Settings → Data & import → Clear example data** after checking the counts.
- Invite my accountant and my small-engine technician from Team logins.
- Link each driver to their vehicle, and link team members to Xero Payroll.
- Add any DNS records you tell me to.
