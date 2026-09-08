# Project Status — Mobile Service Centre ERP

**As at 7 September 2026**

- Live: https://vservice-mobile-service-erp.lovable.app
- Editor: https://lovable.dev/projects/f5ee7928-9a2f-4d9a-aafd-8dc467142a1f
- Head commit: `c1df366`

**Overall: loaded with the shop's real opening position and ready for a supervised
first week.** Every stated requirement is met, the security holes are closed, and
the books now hold real stock, real suppliers and a real opening balance sheet.
What remains is listed under Outstanding.

## Opening position as at 1 September 2026

| | |
| --- | --- |
| Cash in hand | ₹53,949.00 |
| Bank | ₹0.00 — figure not yet supplied |
| Spares stock at cost | ₹27,327.50 across 1,599 units, 80 items |
| Owed to suppliers | ₹49,099.00 — SVS Mobiles ₹2,850, Tulsi ₹46,249 |
| Owed by customers | ₹0.00 |

19 parties: 18 real suppliers and one customer. Document numbering starts at 1, so
the first real job is `JOB/2026-27/0001`.

## Requirements

| # | Requirement | Status |
| --- | --- | --- |
| — | Service modules only, no retail sales | Done — POS hidden, routes redirect to Job Sheets |
| — | Jobsheet → diagnosed → repaired → ready → delivered | Done — four steps: Received → In repair → Ready for delivery → Delivered |
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
| Appsmdass@gmail.com — Master Admin | owner (protected) | Everything, including cost and profit |
| satheesh.ns30@gmail.com — Satheesh | owner | Everything, including cost and profit |
| *(to be created)* | technician | Every module; no cost, margin or profit |

Master Admin is the developer's login and is protected at the database: it cannot
be deleted, deactivated, downgraded, or have its password reset by anyone else.
Satheesh is the shop owner.

No technician login exists yet. Create it from **Users & Roles → Invite staff**,
choosing the Technician role. No email service is configured, so a **one-time
password is shown on screen** to pass on — nothing is emailed. A password can also
be typed rather than generated, and reset later from the row's Reset password
action.

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

**2. Run one real job end to end on the published site** *(owner)* — still the most
important item.
The loop was proven before the data purge, but not after it, and the workflow has
since been rewritten from sixteen statuses to four, altering the status enum, the
transition rules and the delivery path together. That change is compile-verified
only; no authenticated session can be created against the project's Supabase from
the build environment. Book one device from intake through spare issue, the three
status steps, delivery, payment and WhatsApp share before the shop relies on it.

****3. Supply the bank figures** — opening balance as at 1 September, plus bank name,
account number, IFSC and branch. The account exists with those fields blank.

**4. Set reorder levels.** All 80 spares are at 0, so low-stock warnings never fire
and the reorder list stays empty.

**5. Set selling rates.** Every spare is currently priced at cost, so spares bill at
zero margin.

**6. Confirm navigation speed** on the published URL from the shop's own connection.
The 5-second delay was traced to the unpublished dev preview compiling each screen
on click; publishing should resolve it, but that is unconfirmed.

**7. Check the reconciliation report once** — its four problem counts can only be
read from a signed-in screen. Every figure feeding it is clean.

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

Plus three security holes that no build pass surfaced: open self-registration with
attacker-chosen roles, two RPCs answering anonymous callers, and a technician role
holding every permission in the system.

Two further defects came from fixes themselves rather than from features: a reset
dialog that changed someone's password merely by being opened, and an Active
column where "protected" and "switched off" rendered identically on the one row
where that distinction matters most.

The lesson worth keeping: ask for proof by live API call against the real
environment, not a summary of what was implemented.
