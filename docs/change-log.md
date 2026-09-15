# Change Log

Three implementation phases, each applied to the Lovable project and committed
there. Commit hashes below refer to the Lovable project repository.

## Phase A — Shop configuration, GST mode, roles, single firm

Commit `8739e10`

- **Tax mode.** Added `gst_enabled` to system settings, defaulting to **off**, with
  a Tax mode switch at the top of Settings, owner-only. With GST off, server-side
  triggers force `invoice_type='non_gst'` and reject any tax passed to the posting
  RPC; HSN/SAC, GSTIN, place of supply, interstate flag, tax-rate pickers and all
  CGST/SGST/IGST columns are hidden; GST reports are removed; bills print as
  **SERVICE BILL** with no tax summary. Turning GST on restores all of it.
- **Two roles.** Collapsed seven roles to **owner** and **technician**. Existing
  users migrated (admin/manager/accountant → owner, counter/viewer → technician).
  Technician reaches every module but never receives `view_cost`.
- **Single firm.** `branch_id` retained in the schema as it is load-bearing across
  the ledger, but made invisible: no branch or warehouse pickers, no stock
  transfer screen, location resolved server-side. Branches replaced by a single
  Shop Profile.
- **Service-only navigation.** Retail Sales/POS removed from navigation and its
  routes redirected to Job Sheets; the code and tables were deliberately retained,
  because the job delivery bill still posts through the invoice engine. Renamed to
  Spares Purchase, Spares Stock, Customers & Ledger.
- **Workshop dashboard** — jobs received, in repair, ready, delivered today, plus
  low-stock spares and cash collected.

## Phase B — Spare parts stock and purchase

Commit `d386851`

- **Item types** — `spare`, `accessory`, `service`. Spares and accessories are
  quantity-tracked only; the serial/IMEI engine is now restricted to the
  customer's device on a job sheet. `service` items carry no stock.
- **Spare masters** gained rack location, compatible phone models (multi-select),
  reorder level, and unit. Purchase rate is owner-only.
- **Spares purchase** simplified — no IMEI capture step, payment mode selector,
  posts stock and supplier liability in one transaction. **Purchase returns**
  added.
- **Job sheet issue** — spare picker shows live available quantity, rack location
  and model compatibility. Negative stock blocked in `consume_batches` inside the
  posting RPC.
- **Spares Stock screen** rebuilt with five tabs: Current stock, Low stock (with
  suggested reorder quantity), Movement history, Adjustments (owner-only,
  mandatory reason, posted as append-only reversal rows), Defective stock.
- **Alerts** — low-stock badge on the navigation item, low-stock widget on the
  dashboard.
- New RPCs: `spares_available`, `spares_stock`, `low_stock_spares`,
  `defective_stock`, `spare_movements`.

## Phase C — Delivery, billing and WhatsApp

Commit `0a1477e`

- **Delivery in one transaction** via `job_deliver_impl`. Bill lines are every
  spare issued plus each service charge as its own line, then discount, round-off
  and total, with the advance adjusted and balance payable shown. One job, one
  bill — a second is refused. Delivery of a job with no charges and no spares is
  refused unless marked warranty or no-charge.
- **Payment at delivery** — split across cash, UPI, card, bank or credit. Any
  balance stays outstanding against the customer and appears in ageing.
- **WhatsApp** — bill share via `wa.me` with +91 normalisation, plus ready-for-
  delivery and payment-reminder messages. All four templates editable by the owner
  in Settings with `{customer}`, `{job_no}`, `{device}`, `{amount}`, `{shop}`
  placeholders.
- **Estimate gate** — a bill exceeding the approved estimate beyond
  `estimate_tolerance_pct` (default 10%) is blocked until re-approval is recorded,
  enforced in the posting RPC.
- **Unclaimed devices** — configurable threshold (`unclaimed_days`, default 30),
  dashboard tile and report tab.
- **Reports reworked** to seven service modules: Job register, Delivery/Collection
  register, Technician performance, Spares consumption, Outstanding, Day book, and
  an owner-only Profit report. All exportable to CSV.
- New RPCs: `report_delivery_register`, `report_technician_performance`,
  `report_spares_consumption`, `report_day_book`, `report_profit`,
  `unclaimed_devices`.

## Phase D — Inline creation and permission correction

Commit `df0861f`

- **Technician permissions corrected.** The verification pass had over-trimmed the
  technician to Jobs, Spares-view, Customers, Billing and Payments, with a
  "restricted" notice on Purchase and Expenses. That is the wrong shape: the
  requirement is every module except profit. Access was restored to all modules
  and the gating moved down to **column level at the database** — purchase rates,
  bill amounts, discounts, stock valuation and spare cost are denied to
  non-owners, with owners reading them through `purchase_money`,
  `purchase_item_money` and `purchase_return_money` (`SECURITY DEFINER`). The
  technician opens a purchase bill and sees supplier, date, spare and quantity
  received; the money columns are absent, not blanked.
- **Inline creation.** `+` beside Brand and Model on job intake (Model pre-fills
  the selected brand and becomes the highlighted action when a brand has no
  models); `+` beside Supplier and Spare on the purchase bill, with a created
  spare dropping into the line being typed and focus moving to quantity. No typed
  data is lost. The purchase-rate field stays hidden from the technician in the
  inline spare dialog.
- **Delivery arithmetic confirmed.** The ₹2,300 / ₹1,800 discrepancy raised in
  review was a ₹500 advance correctly adjusting the bill. Not a defect.

## Phase E — Pre-launch data reset

Commit `e250264`

Benchmark seed data purged as one all-or-nothing migration.

| Table | Before | After |
| --- | --- | --- |
| Job sheets and children | 12,029 | 0 |
| Sales invoices / items | 15,059 / 35,126 | 0 / 0 |
| Purchases / items | 6,009 / 12,013 | 0 / 0 |
| Payments | 25,005 | 0 |
| Ledger entries | 92,209 | 0 |
| Stock ledger / batches / serials | 81,176 / 6,020 / 40,039 | 0 / 0 / 0 |
| Expenses / journals / day close | 14 / 2 / 1 | 0 / 0 / 0 |
| Activity log | 262,796 | 0 |
| Parties | 2,008 | 3 |
| Items | 3,006 | 4 |
| Spare categories | 21 | 16 |
| Branches / warehouses / payment accounts | 2 / 10 / 5 | 1 / 5 / 3 |
| Logins | 4 | 3 |

Kept: parties Bangalore Distributors, Sri Vinayaga Mobiles, Mohan dass; items
Redmi 13C 4/64, Redmi 13C Display, Refurb Battery R13C, USB-C Cable; all 15
brands; tax rates, Main Branch payment accounts, shop profile, GST mode off,
print settings and all four WhatsApp templates. Duplicate spare categories
collapsed to Batteries, Displays, Chargers & Cables, Covers & Glass, Other
Spares. Document numbering counters reset to 1.

**Note.** The two real suppliers were sitting on the `Test Service Centre` branch
rather than Main Branch. Deleting that branch without checking would have removed
them. They were repointed to Main Branch before the branch was dropped — the
reason the instruction required configuration to be confirmed before deletion.

Logins remaining: Ramesh Owner (owner), Master Admin (owner + technician),
Suresh Technician (technician). Perf Bench removed.

## Performance — dev preview diagnosis and publication

Commits `b4e494c`, `ab79e5f`

The owner reported 5–10 second navigation. The first pass measured against a warm
local dev server on the build machine and reported 0.6–0.9s, which did not match
what he experienced and should not have been relayed as a fix.

Measured properly — phone viewport, throttled mobile link, dev preview:

| | Total | Code fetching | Data | Code requests |
| --- | --- | --- | --- | --- |
| Dashboard → Spares Stock | 4.3s | 3.44s | 0.32s | 27 |
| Dashboard → Spares Purchase | 2.7s | 2.25s | 0.36s | 29 |

Roughly 80% of every click was code delivery; the database was not involved. On
Spares Stock the data request did not begin until 3.6s in. Immediately after any
edit the preview recompiles and the same click took **18.2s across 93 requests**.

Production build, same conditions: Spares Stock **2.3s** on phone and **0.52s** on
desktop; Spares Purchase **0.67s** / **0.70s**, with zero on-demand code requests.

**Root cause: the unpublished development preview compiling and shipping the app
piece by piece on every click.** The app was published to
`https://vservice-mobile-service-erp.lovable.app`, which resolves it.

Genuine application fixes made alongside: preloading now triggers on touch
(`pointerdown`/`touchstart`) rather than hover, which does nothing on Android;
permissions resolve once per session; masters cache for ten minutes. Background
route warming is enabled only in production — it was measured making the dev
preview *worse* by starving the screen the user had tapped, and was disabled there
rather than shipped because it sounded reasonable.

Route weight was checked and found not to be a problem — both screens are 6–9 KB
with no charts or export libraries — so nothing was split.

## Security audit

Commits `8ef0c75`, `87e3dd6`

Audited against the live database with real logins, not by reading code.

### Critical — fixed

1. **Open self-registration with attacker-chosen roles.** The signup handler took
   the role from user-supplied metadata, so any stranger could have created an
   **owner** account on a publicly reachable ERP. Demonstrated by registering a
   working technician account. Sign-up is now refused at the database; a login can
   only exist if the owner first adds it in Users & Roles, and that entry sets the
   role. Re-tested: registration as "owner" is rejected, no account created.
2. **Two RPCs answered unauthenticated callers** — the dashboard summary and the
   WhatsApp templates were readable from the open internet. `EXECUTE` is now
   revoked from `anon`/`public` across the schema.

### High — fixed

3. **The technician role held all 70 permissions**, including delete, approve,
   posting payments and expenses, and `view_cost`.
4. **Purchase cost was readable off the items table** by any staff login,
   bypassing report-layer masking. Now denied at column level; the owner reads it
   through a protected route.

### Verified sound, unchanged

Technicians cannot self-grant roles, write permission overrides, read other
profiles, delete a customer, post a payment or expense, post a stock adjustment,
cancel an invoice or change settings — each attempted live and refused. Ledgers
cannot be written directly. Device lock PINs and patterns are genuinely encrypted
with a vault-held key. Delivery OTPs are hashed and time-limited. Job photos are
in a private bucket; an unauthenticated URL guess returns nothing. Document
numbering locks the row. No admin key is present in the browser bundle.

### Medium — owner action outstanding

Leaked-password protection is disabled. Supabase dashboard → project →
Authentication → Sign In / Providers → Password → enable **Prevent use of leaked
passwords**, set minimum length to 10.

### Note on the fix for finding 3

The audit's first fix cut the technician to 13 permissions, removing Spares
Purchase, Payments and Expenses entirely — the same over-trim as Phase D, and
against the stated requirement. Corrected to: **view broadly, create/edit
narrowly, delete/approve/view_cost never** — 32 permissions. Purchase money
columns (rate, discount, taxable value, tax, freight, paid, total) are now refused
at the database rather than merely omitted by the screen, so the technician sees
that a part arrived, with quantity and date, and no rates even via direct API
calls.

## Accounts, staff management and workflow simplification

Commits `e1bbc68`, `6c2bacc`, `084b9ff`, `46ddfc2`, `e085072`

### Roles and accounts

- **Invite dialog offered only Technician.** It was hardcoded during the two-role
  scoping. Now offers **Owner and Technician**, each with a one-line explanation,
  Technician preselected. The role still comes from the owner-written
  `signup_invites` row, never from anything the invited person types — verified
  that a technician cannot write an invite row, self-promote, or call the invite
  function (403), so adding the Owner option did not reopen the self-registration
  hole.
- **Corrected the dialog's wording.** It claimed "They receive an email invite".
  No mail service is configured and nothing was ever sent — the app shows a
  one-time password instead. An owner following the old wording would have waited
  for an email that never arrives.
- **Account roles settled.** `Appsmdass@gmail.com` (Master Admin) is the
  developer's login; `satheesh.ns30@gmail.com` (Satheesh) is the shop owner. He
  had been created as a technician before the role picker existed and was promoted
  to owner, keeping his profile, phone and password.

### Master Admin protection

Enforced by database triggers, matched by email at runtime rather than by UUID, so
it survives the account being recreated. The account cannot be deleted,
deactivated, stripped of its owner role, downgraded, have permission overrides
applied, or **have its password reset by anyone else** — the last of these matters
because password reset is an account-takeover primitive that would otherwise let
another owner walk past every other guard.

Each attack was fired directly at the database, bypassing the app and RLS, and
each was refused with a clear message. The password-reset guard was tested by
capturing the real reset request from the browser and replaying it against the
protected account's id. Ordinary use is unaffected: the account edits its own
name, phone and password normally, and managing other staff is not restricted.

### Staff management

- **Edit action** on each row — name, phone and role, owner-only and enforced in
  the server function, respecting both the last-owner safeguard and the Master
  Admin guard.
- **Reset password** with an editable field, a Generate button, Copy, and a
  show/hide toggle. Nothing is applied until confirmed — the first implementation
  reset the password the instant the dialog opened, so merely looking would have
  broken someone's login. Sessions are invalidated on change.
- **Password field on invite**, so a login can be created with a chosen password;
  left blank, one is generated. Minimum length validated **server-side**, not only
  in the browser.
- **The Active column was misleading.** Master Admin's toggle rendered grey
  because the protection disabled it, which was indistinguishable from the account
  being switched off — on the one row where certainty matters most. It now shows
  an **"Active · Protected"** badge with a lock; the current user's own row shows
  "Active · You".

### Job workflow reduced to four steps

Sixteen statuses suited a large service centre with separate diagnostic, repair and
QC staff. This is a two-person shop where one person does all three.

**Received → In repair → Ready for delivery → Delivered**, with Cancelled as an
owner-only exception. Existing rows migrated; board columns, mobile tabs, filters,
badges, dashboard counters and reports all updated. History entries keep their
original wording rather than being rewritten.

Preserved deliberately while simplifying, rather than lost with the statuses that
carried them:

- **Estimate over-run confirmation** — moved into the delivery dialog. A bill past
  the quoted figure beyond the tolerance still cannot go out until an owner
  confirms the customer agreed.
- **QC checklist** — folded into the hand-back tab as optional, not deleted.
- **Refusal to deliver an unbilled job** unless warranty or no-charge.
- **One job, one bill.**

Delivery now generates the bill and immediately presents it with a prominent
**Send bill on WhatsApp** button, and the fourth tab becomes **Invoice** once
delivered — bill number and date, spare lines, service charges, discount, advance
adjusted, total and balance, with A4 / 80mm / 58mm printing and WhatsApp share.

**Not behaviour-verified.** The agent could not create an authenticated session
against the project's own Supabase, so this change is compile-verified only. It
altered the status enum, the transition rules and the delivery path together, so a
real job should be booked through to delivery before the shop relies on it.

## Going live — real shop data

Commits `23ac551`, `897f0c0`, `acd9faf`, `93e0e45`, `6b8c252`, `56eb8d8`, `73ac3c6`,
`c1df366`

### Opening stock

The owner's stock report — 80 rows, all reconciling exactly (`Price × Qty = TOTAL`
on every line) — was loaded as **80 spare items, 1,599 units, ₹27,327.50**, dated
**1 September 2026**.

Two decisions were his, not assumed: each of his tray lots stays a **separate
item** (23 rows of CC Pin V8 remain 23 items, because that is how he counts them),
and **selling rate = cost** for now.

Normalised on the way in, and told to him rather than done silently: `SS` and `SAM`
→ Samsung; mixed casing unified; dates embedded in model names stripped
(`Y22 02/08/2025` → `Y22`); 16 categories created from his own vocabulary rather
than forced into generic buckets; lot numbers kept in the item name.

Posted as an **opening balance, not a purchase** — so it created no supplier
liability and did not touch the party ledger.

### Suppliers

**18 real suppliers** loaded from photographs of his records, with contact person,
both phones, email, full address, city, state and pincode. The parties record
gained `contact_person`, `alt_phone`, `address_line1`, `address_line2`, `city` and
`pincode`, and a unique index on `(branch_id, lower(name))` so imports cannot
create duplicates.

Data problems were flagged rather than guessed:

- Chandan Mobile Shop's pincode read `29` — the Karnataka **state code**, not a
  pincode. Left blank.
- RS Communication's second number read `9894` — truncated. Left blank.
- Five records had `Owner` or the business name as the contact person. Left blank,
  because a contact called "Owner" looks filled in and is not.
- `chenai` → Chennai and `CUDDALUR` → Cuddalore, each confirmed by the pincode on
  the same record.

The **CSV importer** now accepts that shape, reports failures by spreadsheet row
with a reason, and refuses duplicates. Header:
`Business name,Contact person,Phone,Alternate phone,Email,Address line 1,Address line 2,City,State,Pincode,Party type`

### Opening balances

**Payables ₹49,099.00** — SVS Mobiles ₹2,850.00 and Tulsi Mobile & Electronics
₹46,249.00, both credit, dated 1 September.

This exposed a real defect. Setting an opening balance on an **existing** party was
refused outright, and `finance_reconcile` counted only supplier bills — so even had
it saved, the amount would never have appeared as a payable. Parties now carry an
**opening as on** date; changing the amount later posts a correcting entry (old
reversed, new carried in); and opening amounts count towards payables and
receivables net of anything already paid on account. Verified by posting a ₹1,000
payment against SVS and watching the balance fall correctly.

**Cash Drawer opening ₹53,949.00** as at 1 September. The bank account remains at
zero pending the owner's figure.

### Money accounts

Reduced to **Cash Drawer** and one bank account. The separate UPI/QR account was
deleted: UPI settles into the bank automatically, so a UPI balance would only ever
be something to forget to clear.

**UPI is a payment mode, not an account** — UPI and bank transfers increase the
bank balance, cash goes to the drawer, and the day book still splits the day by
mode so the owner can see how money came in. **Card removed** (no machine);
**Cheque removed** on request. Modes are **Cash · UPI · Bank · On credit**, each
removal enforced in the database so there is no dead option posting to nothing.

Opening balances on accounts can be set **after** creation, posting properly and
correcting rather than doubling — proved by setting ₹5,000, correcting to ₹1,200,
and confirming the balance read ₹1,200.

### Cash & bank ledger

The owner asked where it was. It did not exist: Payments had only Register,
Outstanding and Party ledger, and a `CashBankBook` component sat unwired in the
codebase. Reports → Day book covered a single day, not a running account — so
there was no way to tally the drawer at closing or reconcile against a passbook.

Added as a **Cash & bank** tab: account selector, date range, opening balance,
every movement in date order with a **running balance per row**, closing figure,
click-through to the voucher, CSV export. The orphaned duplicate under Expenses was
removed rather than left to be found and trusted later.

### Spare creation template

The quick-create dialog did not match the loaded stock: free-text name, **optional**
category, and unit defaulting to **BOX**. Within a month the spares list would have
been half structured and half free text, with category reports wrong and at least
one item counted by the carton.

Reshaped to the stock sheet: **Category** (required, `+` to add) → **Brand** (`+`
to add, blank allowed) → **Model** → name auto-composed as `Category - Brand Model`
and still editable. Unit defaults to **Nos**. MRP added as optional, matching his
unused MOP column. Applied to both the counter dialog and the Masters form.

### Test data

All test documents and parties were removed: the test job sheet, purchase, payment,
Sri Vinayaga Mobiles and Dhanush deleted outright with no orphans, and numbering
reset so the first real job is `JOB/2026-27/0001`.

**A lesson recorded rather than buried.** Six of our own test rows — two
opening-balance pairs and a receipt with its reversal — were left visible in the
owner's cash ledger, and he found them before we did. Reversing a test is not
cleaning up: it leaves two rows where there should be none. Tests against a live
database must be removed completely, or the inability to remove them stated plainly.

## Reports, general service and the delivery bypass

Five report screens were reviewed against the shop's real data. Four of the five
complaints turned out to share one cause.

### Everything assigned to Satheesh

One technician works here, so "Unassigned" was never a real state — it was a
column that could only ever be wrong. Every existing job was assigned to Satheesh
and new job sheets default to him, so the technician report stops reporting on a
person who does not exist.

### General Service

A service that consumes no spare had no way to be recorded: the job sheet expected
parts. Added as a service type that takes labour only, skips the spare picker
entirely, and still bills, delivers and posts like any other job.

### Bill of Supply

After delivery the owner needs a document to hand over. Produced with customer
details, device details, spares and labour itemised, and **both dates** — received
and delivered — so the customer can see how long the repair took. Titled "Bill of
Supply" with GST off and "Tax Invoice" with GST on, matching the mode rather than
carrying tax wording into a non-GST shop.

### The delivery bypass — three holes, all closed

The Deliveries tab read 0, the dashboard said 53 booked / 6 delivered, and My Jobs
disagreed with both. One cause underneath all of it: **the delivery button called
`set_job_status`, not `job_deliver_impl`.** It moved the job to delivered and
skipped billing altogether. Seven jobs were marked delivered with no bill, no
ledger entry and no revenue — which is why the delivery report was empty while the
job list showed deliveries.

Three ways existed to reach delivered without a bill, and all three are now closed:

1. The delivery button, repointed at `job_deliver_impl`.
2. The status dropdown, which allowed delivered as an ordinary transition.
3. A direct data edit, closed at the database — `guard_job_status` now refuses
   `delivered` unless an invoice draft exists, with the message *"Use Hand back to
   deliver this job — that is what creates the bill."*

The first two are application fixes and could be undone by a future change. The
third cannot: the rule lives in the database, so billing is now a property of the
data rather than a habit of the interface.

### Profit report

Repaired, and it reads honestly: labour only, because every spare is still priced
at cost. That is a data gap, not a report bug, and it is listed for the owner.

### Two alarms I raised that were wrong

Recorded because the corrections came from checking, not from review.

**"Your totals are inflated and wrong twice over."** They were not. All 54 jobs
were checked and **zero** carried a double-counted charge. The disagreement between
the dashboard and My Jobs was a display race — two panels reading at different
moments — not bad data.

**"Stock has leaked; do not trust your shelf counts."** It had not. All 99 items
reconciled with **zero discrepancies**: book stock **1,604 units, ₹29,357.50**.
Both Oppo A15 spares were correctly issued to job 0050 and stock had correctly
reduced. The empty parts panel that prompted the alarm was a permissions fault: the
query asked for `cost_rate`, which is revoked at column level, and the block
discarded the whole result — so a job with ₹100 of parts and ₹100 of labour
rendered as an empty job worth ₹0.00. The panel now reads through the `job_costs`
RPC and shows an error when a read fails instead of showing an empty job.

Correcting the stock figure as well: it was quoted as 1,599 units / ₹27,327.50, then
1,606 / ₹29,457.50, before the reconciliation settled it at **1,604 units /
₹29,357.50**.

### Hand back

**Labour is charged per job sheet, not per part.** It had been collected per line,
which would have multiplied a single service charge by the number of spares used.
An **Edit** option was added to Hand back so a mistake can be corrected before the
bill is raised rather than cancelled after.

### Customer names need not be unique

A supplier import had added a uniqueness index on party name. It then blocked the
counter from saving two customers called the same thing — which at a phone shop is
routine. The index was dropped.

**The phone number is the identifier**, and even it does not block a save: entering
a number already on file shows the matching customers with their last visible job
and offers *Use existing customer* or *Save as new*, so a shared family handset
still books. The job-sheet lookup now returns every match rather than the first
five.

One consequence, stated rather than buried: **duplicate suppliers can now be
created by hand too.** The same rule was applied to both party types rather than
splitting the behaviour, because a rule that holds for one kind of party and not
the other is a rule nobody remembers. CSV import still refuses duplicate suppliers,
both within a batch and against existing records, reporting them as skipped rows.

All 74 parties were preserved through the change. Typecheck and production build
both pass.

## Reversing the seven bypass deliveries

The seven jobs that reached Delivered without a bill were put back to Ready for
delivery so the owner can hand each one back properly and get a real bill.

They were not where either of us expected. The owner described them as sitting in
"In repair"; they were still marked **Delivered**. The four genuinely in In repair
(0005, 0009, 0040, 0041) are unrelated and were left alone. Checking before writing
is what caught that — the instruction was to stop and report on any mismatch rather
than move whatever was found.

### The reversal path

`guard_job_status` refuses `delivered → ready_for_delivery`, correctly. Rather than
weaken it, an owner-only `job_reverse_delivery()` was added that:

- refuses anyone who is not an owner;
- **refuses any job that already has a posted bill** — a billed job can only be
  undone by cancelling its invoice, never by a status flip;
- writes a visible history line, so a reversal is never silent.

That second rule is what protects job 0050 and every correctly billed job from here
on. Both refusals were tested live and the test block rolled back.

### Stock

Book stock **1,604 units / ₹29,357.50 → 1,610 units / ₹30,260.50**.

| Part | Job | Cost | Charged |
| --- | --- | ---: | ---: |
| Outer Button – Redmi 10A | 0003 | ₹50 | ₹150 |
| Displays – Samsung M31 | 0004 | ₹800 | ₹1,800 |
| Outer Button – Samsung M31 | 0004 | ₹50 | ₹150 |
| General service | 0006 | ₹1 | ₹50 |
| Display paste – Vivo S1 | 0007 | ₹1 | ₹200 |
| General service – Samsung A35 | 0008 | ₹1 | ₹100 |
| **6 units** | | **₹903.00** | **₹2,450.00** |

Stock rises by cost, ₹903, not by the ₹2,450 charged. The ₹1,547 difference is
mark-up and was never stock value. Every return was appended as a new movement;
nothing was deleted or edited.

Labour was left alone: 0001 ₹150, 0004 ₹1,100, 0048 ₹200.

### Two corrections

**Hand back does not issue parts.** It creates the bill; parts leave stock when they
are issued on the Parts tab. Clearing the part lines therefore means the owner must
re-pick them before each hand back or the bill comes out short. This was my error
and the agent caught it before the change went in, which is why the owner got a
per-job re-issue list rather than a surprise.

**Spares are not all priced at cost.** It was recorded here that every spare bills
at zero margin; the M31 display cost ₹800 and charged ₹1,800 disproves it. Margin
does exist on job lines, so the profit report holds more than was claimed.

Worth the owner's attention: three items carry a **₹1 cost** — both General service
lines and the Vivo S1 display paste. ₹1 is a placeholder, not a purchase price, so
any job using them reports near-total profit on a cost figure that is fiction.

### 0048's blank amount

The register showed no figure against Malathi's job while the database held ₹200
labour. Both were right: that column shows the **estimate**, which is empty. The
labour is real.

## Drag and drop on the job board

Cards move between Received, In repair and Ready for delivery by dragging.
`@dnd-kit/core`, chosen for a touch sensor with a long-press delay so an ordinary
finger swipe still scrolls the board on the owner's Android rather than picking up
a card by accident.

**Delivered is not a drop target**, and that is the point. The job board is exactly
where someone would try to shove a card into Delivered, which is the hole that cost
this shop seven bills. There is no code path in the board that sets `delivered`.
The column stays a drop zone only so it can refuse out loud: it goes dashed red
while dragging, reads "Not a drop target — use Hand back", and releasing there
opens a dialog with an **Open Hand back** button.

Dragging *out* of Delivered calls `job_reverse_delivery()`. A technician cannot
pick up a delivered card at all. Every other move goes through the normal status
change, so `guard_job_status` still has the last word — the board only decides
which columns to light up. Columns a job cannot legally reach dim and stop
accepting drops. A refused move snaps the card back and shows the database's own
wording rather than an invented message.

The status control on the job sheet is unchanged, so dragging is a shortcut and
never the only way. One side effect worth noting: the mobile job board now scrolls
sideways across four columns instead of using tabs.

## Full test pass, and the cost leak it found

A complete test against the live database, every write inside a transaction that
was rolled back. Row counts before and after were identical on every table, so the
owner's books carry no residue — the rule set after six of our test rows were found
in his cash ledger.

### What passed

The whole loop, end to end: job booked, spare issued, three status steps, Hand back,
bill, payment. Stock fell by the issued quantity, the bill carried parts plus labour
with zero tax as a non-GST shop, ledger debits equalled credits, the customer's
balance returned to zero and the invoice settled.

Every guard held. All three routes to `delivered` without a bill refused, including
a direct `UPDATE`. Over-issuing beyond stock refused. A second bill from one job
refused. Estimate over-run blocked until re-approval. `job_reverse_delivery` refused
a billed job and refused a non-owner.

Books reconciled: **1,610 units / ₹30,260.50 across 99 items, zero discrepancies**
item by item; all five reconciliation problem counts at zero.

### The cost leak — ten tables, not one

The test found a technician could read cost straight out of `purchases` and
`purchase_items`. That alone contradicted what this documentation had claimed twice:
that cost is gated at the database rather than hidden in the screens.

Rather than patch the two tables named, the same class of hole was swept for
everywhere. **Ten more were leaking**: stock batches, stock movements, serial units,
sale invoice lines, sale return lines, purchase returns and their lines, purchase
orders and their lines, supplier bills, stock adjustments and their lines, stock
transfer lines, and stock-in-transit.

All now return `permission denied` to a technician on a raw call, verified by
testing as a real technician rather than by reading the code. Owner access is
intact — batch cost, purchase totals, adjustment values, stock value, ageing, job
profit and sale margin all return normally, and the purchase screens, stock
valuation and profit report still work.

Found on the way: the **pay a supplier screen was reading a bill total it had no
permission for**, so it would have shown nothing. Fixed to read through the
protected lookup.

The lesson is the instruction, not the fix: asking for a sweep of the same class of
hole found ten times more than fixing the table that was named.

### GST off now refuses a tax value

It had been silently storing zero. The money was right, but a quietly discarded
value is how a real error hides — an import or integration sending tax would have
gone unnoticed until the returns disagreed. It now raises: *"This shop bills without
GST, so no tax can be recorded on a job line."* Four spares still carry an 18%
setting on the item master; that is ignored rather than blocking the counter over a
setting nobody sent deliberately.

### Not run

**The drag gestures themselves were never exercised in a browser.** No test session
can be minted against the owner's own Supabase, so the preview bounced the agent to
the sign-in page. Everything underneath the interaction was tested and passes. This
is recorded rather than glossed, because a passing report that quietly includes an
untested item is how this project has gone wrong before.

## Moving jobs backwards, and a parts display that lied

Both found by the owner using the app, not by a review.

### Backwards

He could drag a card forward but not back. That was not a drag bug: backward moves
were refused at the database, exactly as the test pass had reported
(*Ready → Received: invalid job status change*). The rule was wrong for a workshop —
a device marked Ready that turns out to be faulty has to go back to In repair.

**Received ↔ In repair ↔ Ready for delivery now move in both directions**, by drag
or by the status buttons, technician included, each move writing a history line.

**Delivered is unchanged and deliberately so.** Still not reachable by dropping;
still leaves only through the owner-only reversal; still refuses a billed job.
Re-tested after the change: a direct `delivered → ready_for_delivery` is still
refused, and skipping straight from Received to Delivered is still refused.

### A return that looked like a charge

After the seven jobs were reversed, job 0004's Parts tab read:

```
Displays – Samsung M31 · Issued 09 Sept · 0 × ₹1,800.00 = ₹0.00 · "1 of 1 put back into stock"
Displays – Samsung M31 · Returned 15 Sept · 1 × ₹1,800.00 = ₹1,800.00
```

Arithmetically correct and append-only, and completely misleading at a counter: a
returned part showing a positive ₹1,800 reads as a charge, and an issue line reading
`0 × ₹1,800 = ₹0.00` reads as nonsense. The owner saw it and concluded the parts
were issued while Hand back said ₹0 — he was reading the screen correctly; the
screen was wrong.

Now: **Currently issued — this is what will be billed** comes first and is the only
billable set. Everything settled sits below under **History — nothing here is
billed**, with a fully returned issue struck through, greyed and marked *Returned*,
and the return itself showing as money coming off (− ₹1,800). No row is hidden or
deleted. With history but nothing out, the tab says so plainly.

### The short bill it nearly caused

Hand back now warns, before billing, on any job with returned parts and nothing
currently issued — naming each part and quantity so they can be re-picked. A
warning, not a block, so a genuinely labour-only job still hands back freely.

This was not hypothetical. Job 0004 sat at Ready for delivery with ₹1,950 of
returned parts and ₹300 labour. Handed back as it stood, it would have billed ₹300.
