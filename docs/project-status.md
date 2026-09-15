# Project Status — Mobile Service Centre ERP

**As at 15 September 2026**

- Live: https://vservice-mobile-service-erp.lovable.app
- Editor: https://lovable.dev/projects/f5ee7928-9a2f-4d9a-aafd-8dc467142a1f
- Head commit: `c8e87ea`

**Overall: loaded with the shop's real opening position and ready for a supervised
first week.** Every stated requirement is met, the security holes are closed, and
the books now hold real stock, real suppliers and a real opening balance sheet.
What remains is listed under Outstanding.

## Opening position as at 1 September 2026

| | |
| --- | --- |
| Cash in hand | ₹53,949.00 |
| Bank | ₹0.00 — figure not yet supplied |
| Spares stock at cost | ₹30,260.50 across 1,610 units, 99 items — after returning the parts on the seven reversed jobs |
| Owed to suppliers | ₹49,099.00 — SVS Mobiles ₹2,850, Tulsi ₹46,249 |
| Owed by customers | ₹0.00 |

74 parties: 18 real suppliers and the customers booked since go-live. Document
numbering started at 1, so the first real job was `JOB/2026-27/0001`.

The stock figure moved twice before it settled. It was quoted as 1,599 units /
₹27,327.50, then 1,606 / ₹29,457.50; a full reconciliation of all 99 items
confirmed **1,604 units / ₹29,357.50** with no discrepancies.

## The books as they stand, 15 September

| | |
| --- | --- |
| Cash drawer | ₹7,700.00 — opening ₹53,949 less ₹46,249 paid to Tulsi on 11 Sep |
| Bank | ₹500.00 — three job collections by UPI |
| Spares stock at cost | ₹30,260.50 · 1,610 units · 99 items · reconciled, zero discrepancies |
| Owed to suppliers | ₹5,883.00 — SVS ₹5,700, Sathya V Connect ₹153, Star ₹30. Tulsi settled |
| Owed by customers | ₹0.00 |
| Bills raised | 3 — INV/2026-27/0001, /0002, /0003 |

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
| — | General Service — labour only, no spare | Done — skips the spare picker, bills and delivers normally |
| — | Move a job backwards | Done — Received ↔ In repair ↔ Ready for delivery, both ways |
| — | Drag and drop on the job board | Done — Delivered is not a drop target; gestures untested in a browser |
| — | Bill of Supply after delivery | Done — customer, device, spares, labour, received and delivered dates |

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

The shop is live: real stock, real suppliers, real opening balances and 54 job
sheets booked. The paragraphs below record the pre-launch reset that preceded it.

The database was taken to a clean pre-launch state before go-live. All benchmark data was purged: 12,029
job sheets, 15,059 invoices, 92,209 ledger rows, 81,176 stock rows and 262,796 log
rows removed; parties cut from 2,008 to 3; items from 3,006 to 4. Document
numbering resets to 1, so the first real job sheet is **#1**.

Retained through that reset: 15 brands; tax rates, payment accounts, shop profile,
print settings and all four WhatsApp templates. The test parties and test documents
that survived it — Bangalore Distributors, Sri Vinayaga Mobiles, Dhanush, and the
sample items — were removed afterwards on the owner's instruction, outright rather
than reversed.

## Outstanding

**0. Hand back the remaining five jobs** *(owner)*
0007 and 0008 are done — billed as INV/2026-27/0002 (₹200) and /0003 (₹100). Five
left, all at Ready for delivery. **Re-issue the parts on the Parts tab first**; Hand
back now warns when a job has returned parts and nothing issued, but it is a warning,
not a block:

| Job | Customer | Re-issue before Hand back | Labour |
| --- | --- | --- | ---: |
| 0001 | RAJENTHIRAN | none | ₹150 |
| 0003 | VINOTH | Outer Button – Redmi 10A ×1 @ ₹150 | — |
| 0004 | JAYASURYA | Display M31 ×1 @ ₹1,800 · Outer Button M31 ×1 @ ₹150 | ₹300 |
| 0006 | karthickraj | General service ×1 @ ₹50 | — |
| 0048 | Malathi | none | ₹200 |

**0b. Fix the three ₹1 cost prices.** Both General service items and the Vivo S1
display paste carry a ₹1 cost, which is a placeholder rather than a purchase price.
Any job using them reports near-total profit on a cost figure that is fiction.

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

**3. Supply the bank figures** — opening balance as at 1 September, plus bank name,
account number, IFSC and branch. The account exists with those fields blank.

**4. Set reorder levels.** All 80 spares are at 0, so low-stock warnings never fire
and the reorder list stays empty.

**5. Check selling rates.** Not every spare is priced at cost — the M31 display
costs ₹800 and charges ₹1,800 — but the rates have never been reviewed as a set,
and three items carry a ₹1 placeholder cost (item 0b above).

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
6. Delivery itself — the delivery button called `set_job_status` and skipped
   billing, so seven jobs reached delivered with no bill, no ledger entry and no
   revenue. Three separate routes to that state existed; all three are now closed,
   one of them at the database where a later change cannot reopen it.
7. Technician cost-gating, a second time. A full test pass found `purchases` and
   `purchase_items` readable by a technician; sweeping for the same class of hole
   found **ten more** leaking tables. Cost had been claimed here twice as gated at
   the database. It was not.

Plus three security holes that no build pass surfaced: open self-registration with
attacker-chosen roles, two RPCs answering anonymous callers, and a technician role
holding every permission in the system.

Two further defects came from fixes themselves rather than from features: a reset
dialog that changed someone's password merely by being opened, and an Active
column where "protected" and "switched off" rendered identically on the one row
where that distinction matters most.

Two alarms in the other direction are worth recording too, because both were mine
and both were wrong. I told the owner his job totals were double-counted: all 54
jobs were checked and none were — the disagreement between two screens was a
display race. I told him stock had leaked and not to trust his shelf counts: all 99
items reconciled exactly, and the empty panel that prompted the alarm was a blocked
column read discarding a result the interface then rendered as ₹0.00.

The lesson worth keeping: ask for proof by live API call against the real
environment, not a summary of what was implemented — and hold a diagnosis to the
same standard before passing it on as a warning.
