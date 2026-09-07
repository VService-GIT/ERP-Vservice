# Project Status — Mobile Service Centre ERP

**As at 7 September 2026**

- Live: https://vservice-mobile-service-erp.lovable.app
- Editor: https://lovable.dev/projects/f5ee7928-9a2f-4d9a-aafd-8dc467142a1f
- Head commit: `87e3dd6`

**Overall: ready for a supervised first week of real use.** Every stated
requirement is met and the security holes found have been closed. Two items remain
before it should carry a full day's takings unsupervised — both listed below.

## Requirements

| # | Requirement | Status |
| --- | --- | --- |
| — | Service modules only, no retail sales | Done — POS hidden, routes redirect to Job Sheets |
| — | Jobsheet → diagnosed → repaired → ready → delivered | Done — full lifecycle with QC gate and delivery OTP |
| — | Delivery bill = spare cost + service charge | Done — one job, one bill, enforced server-side |
| 1 | Service spares stock maintenance | Done — five-tab stock screen, reorder levels, rack locations |
| 2 | Party ledger maintenance | Done — running balance, bill-wise settlement, ageing |
| 3 | Settings switch: with GST / without GST | Done — shop-wide, enforced in the posting RPC, currently **off** |
| 4 | Delivery invoice shared on WhatsApp | Done — plus intake, ready and reminder templates |
| 5 | Spares purchase adds stock; job issue reduces stock | Done — negative stock impossible at the RPC |
| — | One firm, no branches | Done — single firm, branch and warehouse UI removed |
| — | Two users: owner sees all, technician sees all but profit | Done — see below |

## Added beyond the requirement

Each because the workflow is wrong or exploitable without it: estimate over-run
re-approval; refusal to deliver an unbilled job; defective-spare bucket; purchase
returns; unclaimed-device tracking; day book for tallying the drawer; spares
separated from IMEI-tracked devices; technician cost-blindness enforced at the
database rather than by hiding a report.

## Users

| Login | Role | Sees |
| --- | --- | --- |
| Master Admin | owner | Everything, including cost and profit |
| *(to be created)* | technician | Every module; no cost, margin or profit |

The shop's second login has not been created yet. The owner creates it from
**Users & Roles → Invite staff**. No email service is configured, so the app shows
a **one-time password on screen** to pass to the technician rather than emailing an
invite. The path was tested end to end and works.

Technician holds 32 permissions: view and export on every module; create and edit
on job sheets, customers, masters/spares, stock and billing. Proven by live test
that he cannot post a payment, book an expense, adjust stock, cancel an invoice,
delete a customer, change settings, open the profit report, read item cost, or
reach Users & Roles.

## Data

The database is at a clean pre-launch state. All benchmark data was purged: 12,029
job sheets, 15,059 invoices, 92,209 ledger rows, 81,176 stock rows and 262,796 log
rows removed; parties cut from 2,008 to 3; items from 3,006 to 4. Document
numbering resets to 1, so the first real job sheet is **#1**.

Retained: Bangalore Distributors, Sri Vinayaga Mobiles, Mohan dass; Redmi 13C 4/64,
Redmi 13C Display, Refurb Battery R13C, USB-C Cable; 15 brands; tax rates, payment
accounts, shop profile, print settings and all four WhatsApp templates.

## Outstanding

**1. Enable leaked-password protection** *(2 minutes, owner)*
Supabase dashboard → project → Authentication → Sign In / Providers → Password →
enable **Prevent use of leaked passwords**, set minimum length to 10.

**2. Run one real job end to end on the published site** *(owner)*
The full loop was proven before the data purge, but not after it on the clean
database, because an authenticated owner session cannot be created from the build
environment. Take one job from intake through spare issue, delivery, payment and
WhatsApp share before trusting it with a day's work.

**Also worth doing:** confirm navigation speed on the published URL. The 5-second
delay was traced to the unpublished dev preview compiling each screen on click;
publishing should resolve it, but it has not been confirmed from the shop's own
connection.

## Track record of this work

Recorded because it shows which claims needed independent checking. Five separate
items were reported complete by the pass that built them and were not:

1. Technician cost-gating (Phase A) — technician could read every purchase rate.
2. Delivery and billing (Phase C) — broken by a type error; the core function of
   the app did not work.
3. Job number collisions between test and live data.
4. GST wording leaking into ledger labels with GST off.
5. Navigation performance — reported fixed against a measurement taken on the
   wrong machine.

Plus two security holes that no build pass surfaced: open self-registration with
attacker-chosen roles, and two RPCs answering anonymous callers.

The lesson worth keeping: ask for proof by live API call against the real
environment, not a summary of what was implemented.
