# TURVIQ full-site audit: morning report

Scope: every route (36 pages on desktop and mobile), the reports and numbers, roster/routing, money flow (quote to invoice to payment to reconciliation), communications, payroll, HR/SOPs, client portal, security, accessibility and scale. I checked against the business you actually run: recurring lawn and garden maintenance, about 1,500 clients, a small crew, Jim's Mowing franchise, some contract and housing-provider work.

**State of the code:** everything below marked *Fixed* is on `main` in `lazerlawns/lawnsforlife` (latest `231804b`), ready for Lovable to sync. `bun run check` passes (auth, security, comms, platform, files, supabase, performance, routing, reports) and `tsc` is clean apart from the old vitest-only file. Please press **Publish** in Lovable.

**What I could not verify:** anything that needs the live database, logs or secrets (SMTP from the server, SMS receipts, cron actually running, live Stripe webhook, Twilio limits). Those are in `LOVABLE_NEXT_PROMPT.md`.

---

## 1. Reports: do the numbers line up?

**Short answer: they did not, for a real account. They do now.**

The Business reports page, the LTV section and the analytics engine read the *sample* clients and employees compiled into the app, not your own. On a real organisation that meant:

| Symptom | Cause | Status |
|---|---|---|
| Client tables empty or showing raw ids | client lookup used sample list | **Fixed** |
| Team table listed sample staff (Rodney / Markus / Nathan) | sample employees | **Fixed** |
| Every wage was a flat $38 | wage lookup used sample list | **Fixed**: uses each employee's actual rate |
| Every client "One-off", 120-day cycle, no lost-client reasons | recurring profiles were an empty list | **Fixed**: built from active Recurring services |
| Payment-method card guessed the method | inferred from amount | **Fixed**: shows how it was really paid (Bank transfer / Card / Recorded by hand) |
| Quote values counted lines you would not charge | included optional lines | **Fixed**: only what would be charged |
| Revenue compared ex-GST labour against inc-GST collected | mismatched bases | **Fixed** and labelled |

**How I proved it:** `scripts/check-reports.ts` builds a small business worked out by hand (Ann $30/h, Bob $40/h, labour revenue $500, labour cost $240, 6 hours, GST, a recurring client, card and bank payments) and asserts every headline number. It runs in `bun run check` so a future change cannot silently break the reports.

**Still to be aware of (not bugs, but affect how you read the page):**
- "Labour hours" are on-site hours per person. The invoice suggestion now includes drive time, so revenue per hour on the report will read a little lower than the invoice rate. Both are correct; they measure different things.
- Travel hours depend on the driving-time cache being filled. The server now keeps it (not just the browser), so it will fill as routes are planned.
- Report dates use Melbourne time. Fine for you; it is hard-coded in about 85 places, so a business in another state would see edge-of-day errors (see suggestions).

## 2. Bugs found and fixed during the audit

| # | Severity | Finding | Fix |
|---|---|---|---|
| 1 | High | Reports used sample data (above) | Fixed, tested |
| 2 | High | **Monthly jobs starting on the 29th-31st** skipped a month and doubled another (Jan 31 gave Feb 2, Mar 3, Mar 31, May 1) | Fixed: stays in its own month; a weekend date moves to the Friday rather than into the next month. Tested. |
| 3 | High | **Roster route map** looked clients up in the sample list, so on a real account every pin sat at the default point and the map was meaningless | Fixed: uses your clients |
| 4 | High | **Bulk messages** sent `{{first_name}}` literally, with no confirmation, to up to ~1,465 clients in one click | Fixed: placeholders filled per person, confirm step shows recipient count and SMS parts |
| 5 | Medium | Curly quotes/dashes in texts silently doubled SMS cost (unicode encoding) | Fixed: messages made plain before sending, capped at 1,600 characters |
| 6 | Medium | Duplicate invoice numbers when closing a project (count-based) | Fixed: next free number |
| 7 | Medium | A client already reminded of a time could be moved by the planner | Fixed: reminded time is pinned |
| 8 | Medium | Clashing fixed-time jobs could be handed to another person | Fixed: stays put with a warning or goes to overflow with the reason (your rule) |
| 9 | Medium | Admins appeared on the roster by default | Fixed: Admin is off the roster unless switched on |
| 10 | Medium | Client portal and links exposed on-site time, check-out and crew notes | Fixed: clients only see "checked in" |
| 11 | Medium | Long, ugly, cut-off links in texts and emails | Fixed: short `/s/<code>` links, signature verified |
| 12 | Low | Low-contrast text (mint on light, muted grey) failed accessibility contrast | Fixed |
| 13 | Low | Unlabelled controls for screen readers | Fixed |
| 14 | Low | A tab left open overnight showed yesterday | Fixed: reloads at day rollover |
| 15 | Low | Three sample photos were 7 MB | Compressed to 0.7 MB |

Also delivered earlier this session: simplified PDFs, SOP-based induction with signed PDFs, signed contracts emailed to both parties, portal History tab, split Email / SMS tabs, chat scrolls to newest, reconciliation auto-link with a searchable invoice picker, roster follows each person's hours and leave, server-side driving-time cache.

**New safety net:** Settings → Data & import now has a **Data health** panel. It flags duplicate invoice numbers, double-booked visits, weekend jobs (the roster is Mon-Fri), month-end services, completed work with no invoice after 7-120 days, invoices over 60 days overdue, recurring clients with no visit in 4 weeks, clients with no contact method, unverified addresses, and employees with no wage. There is also a one-click **Download a copy of your data** backup.

**Crawl results:** 36 routes on desktop and mobile: no page errors, no blank pages, no horizontal overflow. The only console noise is external fonts and maps blocked by the audit sandbox.

## 3. Open findings (not fixed): decide and prioritise

### Payroll (check with your accountant before relying on it)
- **Super is calculated on gross including overtime.** Superannuation Guarantee applies to ordinary time earnings, so it is overstated whenever there is overtime.
- No **casual loading**, no **weekend or public-holiday penalty**, **overtime is flat** (38 h at 1.5x), no leave loading or accruals.
- **Payday Super** (super paid with wages, from 1 Jul 2026) is not tracked.
- Recommendation: treat the payroll page as an estimate and keep Xero Payroll as the system of record until these are addressed.

### Scheduling
- **Roster and router are Monday-Friday only**, although employee hours allow Saturday/Sunday. A service dated on a weekend is created but never scheduled; the Data health panel now flags these. If you ever work Saturdays, this needs a setting.
- No **rain-day reschedule** (move a whole day's jobs to the next dry day and notify clients). For lawn care this is probably the single most useful scheduling feature missing.
- No **seasonal frequency** (fortnightly in spring, every 4-6 weeks in winter) on a single service.

### Communications
- **Bulk messages have no unsubscribe or consent record.** The Spam Act requires a working unsubscribe on marketing messages and a record of consent. Bulk messaging uses the "Visit reminders" permission. Keep it to service messages until this is added.
- Bulk sending runs as a loop in the browser tab; closing the tab part-way stops it. It should be a server job.

### Multi-tenant
- Melbourne timezone and Jim's Mowing assumptions are hard-coded in places (about 85 timezone uses; `mailbox.server.ts` constants listed in the Lovable prompt). Fine for you; blocking before selling TURVIQ to businesses elsewhere.

### Needs Lovable (live access)
See `LOVABLE_NEXT_PROMPT.md`: server SMTP/mailbox sending, SMS receipt cause, cron verification, live Stripe webhook, Twilio limits, Google Places API, Firebase push, Xero auto-reconcile, bank-feed enrichment of stored data.

## 4. Suggestions for the business you want to run

Ordered by value for a lawn and garden business with about 1,500 clients.

1. **Rain-day reschedule** (see above). Biggest time saver, and a good client message to automate.
2. **Price-increase tool:** pick clients or a service type, apply a % or $ uplift from a date, send a notice and keep the history. With ~1,500 recurring clients this is one of the highest-value things a CRM can do each year.
3. **Seasonal frequency and "skip this visit"** on a service, with the client able to skip from the portal.
4. **Direct debit or card on file** for recurring clients. Chasing invoices is where small lawn businesses lose time; the Data health panel's "overdue over 60 days" will show you how big the problem is.
5. **Quote-to-recurring conversion:** a manual convert step exists on the Quotes page; accepting a quote for ongoing work from the client's link could do this automatically.
6. **Before/after photos on the invoice or job report** by default. It is your best proof of work in disputes and a strong retention tool.
7. **Reviews:** ask for a Google review automatically a day after a good-rated job (I did not confirm a job rating is captured today; the job report is).
8. **Weather-aware planning:** flag days with forecast rain or extreme heat on the roster.
9. **NDIS / housing-provider fields** (plan number, support item, parent account roll-up). The Necessity Housing umbrella import is still open in the Lovable prompt.
10. **Offline-friendly app shell** for crew in poor reception: check-in, notes and photos queued and sent later.
11. **Archive old history** once data grows. The whole organisation is one JSON document that is downloaded in full; with about 1,500 clients and years of visits it will slow down. Moving visits, messages and invoices older than 12-18 months into separate tables is worth planning before it hurts.
12. **Reconciliation line on every report:** a footer "Invoiced X, collected Y, outstanding Z" tying the revenue figures together so you can trust them at a glance.
13. **Sage / reports question box** could be given a short list of "ask me" prompts (e.g. clients not visited in 6 weeks, top 20 debtors).

## 5. What to do first

1. Press **Publish** in Lovable so the fixes above go live.
2. Paste `LOVABLE_NEXT_PROMPT.md` into Lovable (SMS and email sending are the main blockers to trusting automation).
3. Open **Settings → Data & import**: read Data health and run the Address check.
4. Open **Reports** with your real data and sanity-check one client and one month by hand. They should now tie out.
5. Keep payroll in Xero until the super/overtime points are decided.
