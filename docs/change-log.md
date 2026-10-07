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

## Amending a posted purchase

The owner asked for an Edit option on spares purchases, and specifically that a date
change carry through everywhere the original posting went. A posted purchase has
already moved stock into batches and money into a supplier's account, so this was
planned before any code was written — the only feature on this project treated that
way, and it repaid the round several times over.

### What the planning round established

**FIFO does not run on the bill date.** `consume_batches` orders by `received_at,
created_at` — the moment the batch was physically created. So moving a purchase's
date cannot retrospectively change which batch a past job drew from, and cannot
alter a cost already posted. That single fact is what made a date amendment safe.

**The register running backwards is backdating, not a numbering fault.** There is one
`PB/` series and the number is taken at the moment of posting. `/0013` was created on
15 Sept for a 15 Sept bill; `/0023`–`/0028` were created on 16 Sept but dated 8–10
Sept. Numbers must never be reused, so the fix is not renumbering — the register now
**sorts by date, then voucher number**, on screen, in print and in the export, since
the printed copy is the one that matters at assessment time.

**An existing flaw, found by asking.** `ledger_reverse_voucher` stamped reversals
`current_date`. That is wrong for any amendment — a reversal would land in today's
day book against a bill dated weeks earlier. A date-aware variant now exists.

### The three paths, split by risk

| Path | Availability | Effect |
| --- | --- | --- |
| Bill details — bill no., supplier bill date, notes | Always | Saves instantly, no ledger movement |
| Posting date | **Even when stock has been consumed** | Reverses on the old date, re-posts on the new one |
| Supplier, lines, rates, freight | Only while no stock consumed | Re-posts under the same voucher number as revision 1 |

The date path was widened during review. The plan had allowed it only on a bill with
no consumption, which would have gutted the feature — 22 live batches were already
consumed, so the owner would have been refused on most bills, for the one edit he
actually asked for. Since FIFO runs on `received_at`, the narrow path is safe: it
re-dates the postings and leaves batches, receipt order and every issued cost alone.

### The refusals, each tested by live call

> "This stock was issued on 09 Sep 2026 on JOB/2026-27/0004. The purchase cannot be
> dated after that."

Added during review. Without it, dating a purchase after its stock was issued makes
stock-as-on-date reports go negative and shows a phone repaired with a part bought
the following week.

> "That date falls in 2025-26 but this voucher is numbered for 2026-27. Cancel this
> bill and re-enter it in the correct year."

> "Stock from this line has already been used on JOB/2026-27/0004. Quantity, rate and
> item cannot be changed. Raise a purchase return instead, or amend only the date."

> "This bill has ₹1.00 already paid or allocated. The amended total of ₹0.00 would
> leave it over-paid."

Proved on a real bill and rolled back: PB/2026-27/0003 moved 01 Sept → 28 Aug. Stock
in on 01 Sept, reversal out on 01 Sept, new in on 28 Aug; both accounting legs
reversed on 01 Sept and re-posted on 28 Aug. The batch kept `received_at` 09 Sept and
cost ₹800, and the job's issued cost stayed ₹800.

Append-only is intact throughout: no `stock_ledger` or `ledger_entries` row is updated
or deleted, the voucher number never changes, and each amendment carries a revision
counter, a reason and an audit row holding the before and after.

### Four cancelled bills carry mis-dated reversals

Found by asking whether the `current_date` flaw had already bitten. It had:
PB/0001 out by 7 days, PB/0005 by 15, PB/0013 by 1, PB/0014 by 13, in both the
accounting and stock ledgers.

**Left alone, deliberately.** A cancellation genuinely happened on the day it was
cancelled, so dating the reversal there is defensible, and the net effect across the
books is zero — only the individual days fail to net cleanly in the day book.
Rewriting four historical entries to tidy a report costs more than the untidiness.
Recorded here so it is a known position rather than an undiscovered surprise, and
reversible if the owner wants the day book to net per day.

## Warranty terms on the bill and the WhatsApp message

Warranty was already configured per job: at hand back the counter enters service
warranty days, and the system stores the days and an expiry. Eight delivered jobs
carry warranty — one at 365 days, four at 182, three at 30. The owner's Tamil terms
now print on the Bill of Supply and go out with the WhatsApp bill whenever a job
carries warranty.

"Carries warranty" means **service warranty days above zero**. `is_warranty_job` is
a different thing — it marks a free re-repair under an earlier warranty — and no job
uses it.

### Stored exactly, and owner-editable

The terms live in **Settings → Service & WhatsApp**, editable by the owner, beside
the WhatsApp templates. They are never translated, reworded, renumbered or
truncated, and the stored text was verified byte for byte against what the owner
sent: 678 characters, checksum matched. The owner's own numbering and his separate
closing English line are preserved as written.

### Two risks, and how they turned out

**Tamil on thermal printers was a real problem, and it predated this work.** The
bill named no Tamil font at all, so it relied on whatever the PC or phone happened
to have installed — which is exactly how a bill ends up printing boxes. The bill now
carries its own Tamil font and waits for it to load before printing. Rendered in
black-and-white dots at thermal resolution, the Tamil is clear and joins correctly
at both 58mm and 80mm; the warranty text was enlarged and darkened for thermal. On
58mm the terms add roughly 9–10cm of paper.

One risk remains and is the owner's to close: some Bluetooth printer apps send plain
text rather than an image, and Tamil prints blank on those. **One real print on the
shop's own printer is still outstanding.**

**The WhatsApp length worry was overstated — by me.** I estimated around 6,000
encoded characters. The real figure is **3,842** for the terms and 4,487 for a full
warranty bill, because roughly 45% of the text is English. The link service accepted
it and rendered the complete text, and still accepted links up to 30,888 characters.
Recorded because the estimate was wrong in the cautious direction, which is still
wrong.

### Term 6 and the warranty card that did not exist

Term 6 originally read *"Warranty Card இல்லாமல் Warranty வழங்கப்படாது"* — no warranty
without a warranty card. **The application issues no warranty card.** It prints an
intake slip and a bill, nothing else. So the term referred to a document the customer
never receives, and as written it read against the shop: a customer could argue no
card was ever given.

Raised with the owner rather than built around. He chose to make the bill the
warranty document, so term 6 became:

> 6. இந்த பில் (Bill of Supply) தான் Warranty ஆவணம். இந்த பில்லை காண்பிக்காமல்
> Warranty வழங்கப்படாது.

Only line 6 changed — verified by checksum against the original with that single
line swapped. Terms 1–5, 7 and the closing line are byte-identical. A shop that has
already rewritten its own terms is **skipped rather than overwritten**, and
re-running the change does nothing; both were tested and rolled back.

### The line that makes the reword work

A bill that is now the warranty document is worthless to the customer if it goes in
the bin, so warranty bills print, bold and centred inside the warranty box:

> இந்த பில்லை Warranty காலம் முடியும் வரை பத்திரமாக வைத்திருக்கவும்.

Bill only. Not in the WhatsApp message, where the text already sits on the
customer's phone and cannot be lost.

### Checking Tamil written by someone who does not speak it

Both new lines were written here, and were checked before saving rather than after
printing. They held: **பில்லை** is the correct object form (பில் + ஐ, with ல்
doubling after a short syllable, as கல் → கல்லை); **பத்திரமாக வைத்திருக்கவும்** is
what a shopkeeper in Tamil Nadu would actually say, where பாதுகாப்பாக would be more
formal; **காண்பிக்காமல்** suits a printed term where காட்டாமல் would be too casual;
and **ஆவணம்** matches the register of the surrounding terms. The bundled font was
confirmed to carry every letter used.

One optional refinement was raised and deliberately not applied: `பில் (Bill of
Supply) தான்` places the bracket between the word and தான். It is common on bills and
reads fine; `இந்த பில் தான் (Bill of Supply) Warranty ஆவணம்.` flows slightly better.
Left as the owner wrote it, and he can change it in Settings.

## Monthly reporting, and the mismatch the owner spotted

The owner reported his totals were out by roughly ₹300 to ₹500 and asked for a
monthly report. He did not say which two screens disagreed, so the instruction was
to find it: build net profit from source vouchers, compare against every report and
dashboard tile, and **report before fixing anything**.

### The ₹500 was on his customers' bills

Five bills printed more spares than they charged for. When a job was handed back,
the bill listed **every spare ever issued to it, including ones that had been
returned**:

| Bill | Job | Lines add to | Total |
| --- | --- | ---: | ---: |
| INV/0002 | 0007 | ₹400 | ₹200 |
| INV/0003 | 0008 | ₹200 | ₹100 |
| INV/0005 | 0006 | ₹100 | ₹50 |
| INV/0007 | 0003 | ₹300 | ₹150 |
| INV/0004 | 0004 | ₹3,900 | ₹1,950 |

The first four come to **exactly ₹500**, which is what he had noticed. Finding that
is what turned a vague complaint into a located bug.

This was never a reporting inconvenience. A customer adding up the lines on their
own bill got a different number from the total they paid. The hand-back now lists
only spares net of returns, and the five existing bills were corrected — totals
unchanged, ledger unmoved, the old lines kept word for word in the audit log, and
the phantom **₹2,450** cleared from the stored lines that some reports read.

### Three more things that pass found

**The Profit report's columns did not add up, by ₹200.** It silently deducted a
discount it never displayed. The profit was right; the presentation was not.
Discount is now its own column.

**Reports counted on different dates.** The dashboard used bill date; Profit,
Deliveries and the Job register used the date the phone came in. Over all of
September both gave ₹87,880, but 17 September read ₹32,550 one way and ₹3,300 the
other. The owner chose bill date everywhere for money. Reports that are genuinely
operational keep the received date and now **say so on the screen** — the failure
was never the choice of date, it was that neither screen said which it used.

**Two reports used the same word for different things.** Job register "Service
revenue" (₹87,880, the whole bill) and Profit report "Service charges" (₹18,900,
labour only). Renamed to "Total billed" and "Labour charges", and three further
collisions were found and fixed.

### Monthly report

Day by day plus a month total. The owner's formula was *Total Sales (Service Revenue
+ Spares Margin) − Cost spares − expenses*, which **double-subtracts the cost of
spares**: if total sales already uses margin, the cost has been taken out once.
Implemented instead with every line visible so the arithmetic can be audited:

```
Service revenue + Spares revenue − Discount = Total sales
− Spares cost = Gross profit
− Expenses    = Net profit
```

Spares margin appears as a derived column marked as already inside gross profit, so
it can never be subtracted twice.

September: total sales ₹87,880.00, spares cost ₹38,046.90, gross ₹49,833.10,
expenses ₹3,057.00, **net ₹46,776.10**.

## Monthly summary — a one-page A4

Opening and closing stock and cash, the trading figures between them, each
percentage labelled with its base.

**The trap designed around:** the owner's layout puts Total Purchases directly above
Gross Margin, and any reader will assume margin = sales − purchases. It is not —
margin uses the cost of spares **issued**, purchases are what was **bought**. That
is the same confusion that produced his mismatch, so purchases sits outside the
margin chain, in the stock movement block, labelled as money spent on stock rather
than cost of sales.

**The page proves itself.** Opening stock ₹27,927.50 + purchases ₹37,903.00 −
issued ₹38,046.90 ± nil = **₹27,783.60**, and the stock books agree. That figure was
also cross-checked lot by lot, worked out independently. Cash likewise, per account.

One page at real A4 — 243 mm of 277 mm usable. Checked in greyscale, not assumed:
the first attempt had the stock and cash headings rendering as near-identical grey,
so the cash heading was darkened and the sections given different borders so they
stay distinct with no colour at all.

## The cash figures, and a warning I gave that was wrong

Building the summary turned up a cash double-count. Entering an opening balance on a
money account writes it into the books as an opening entry — and two reports then
added the same figure again on top.

**I told the owner his dashboard had been overstating his cash by ₹53,949 every
day, and that he might have made a buying decision on it. That was wrong.** The bad
figure was being calculated behind the dashboard, but no dashboard tile displays a
cash or bank balance at all. I passed on a bad formula as a bad number on his screen
without checking the screen.

A real fault was found in the same place: the dashboard **had been failing to load**,
still asking for two job states the four-step workflow removed. Job counts arrived by
another route, which is why it looked like it worked, while "Cash collected today",
the alerts, the month trend and top technicians came through blank.

### The day book was inverted, and swapping the columns would not have fixed it

Money in and out were the wrong way round on drawer and bank lines. The cause was
not a label: the day book read the drawer side **backwards** *and* also picked up the
customer side of every payment, counting it twice. September showed ₹140,286 in and
₹137,229 out, neither tying to anything. Flipping the labels would have preserved
the double count.

It now reads only the money-account side: in is money arriving, out is money leaving.

### Three screens, one answer

| September | Opening | In | Out | Closing |
| --- | ---: | ---: | ---: | ---: |
| Drawer — Cash & bank book | ₹53,949.00 | ₹69,480.00 | ₹52,056.00 | ₹71,373.00 |
| Drawer — Day book | ₹53,949.00 | ₹69,480.00 | ₹52,056.00 | ₹71,373.00 |
| Drawer — Monthly summary | ₹53,949.00 | ₹69,480.00 | ₹52,056.00 | ₹71,373.00 |
| Bank — all three | ₹0.00 | ₹18,750.00 | ₹0.00 | ₹18,750.00 |

An opening balance is not a receipt, so it is now counted as opening on all three
rather than as money arriving on one. The edge cases were tested rather than
assumed: a range starting **after** the opening entry brings forward ₹53,482 with
nothing leaking in from the day before; a range before either account existed
returns a real zero; an empty October carries the balances through rather than
showing blank; and every row's running balance was checked to walk from the new
opening to the closing without a break.

## Stock counts locked down

A security finding said a count line could be edited without the row being
re-checked afterwards, so it could be moved onto another branch's count or one
already closed.

Sweeping 33 edit rules across 32 tables found the same shape **three more times**:
the count header (a count could be closed by editing it, skipping the Close button
and its stock posting), and purchase orders and purchase returns (a draft could be
flipped to posted with no stock or ledger entry behind it).

**A closed count is now final for everyone** — owner included, and a database admin
is refused too. There is no reopen, by design: if a closed count was wrong, the fix
is a new count or a stock adjustment, and either leaves its own record. Ordinary
counting still works, proved by editing a line 10 → 7, adding a line, and closing a
count with the Close button.

Three gaps were found and deliberately left, recorded so they are known rather than
forgotten: a posted payment's or expense's amount can be edited directly; a
technician can hand a job to someone else (never take one); and a count's system
quantity is typed by the screen rather than read from stock.

**Leaked-password protection is a paid-plan feature.** It had been recorded here and
repeated to the owner for weeks as a two-minute dashboard toggle. Supabase's
documentation says otherwise: on the free plan the switch is shown but greyed out.

## Profit taken out

The owner draws profit monthly and wanted somewhere to record it.

**It is a drawing, not an expense**, and that distinction mattered more than the
feature. Booked as an expense, a month where he draws ₹40,000 would read as
near-breakeven, and the books would understate the shop every month from then on.
Profit is earned first; drawing happens out of profit already made.

So on the Monthly summary it sits **below** net profit:

```
Net profit − Profit taken out = Profit retained in the business
```

with the cumulative retained figure alongside. Proved with a ₹40,000 test drawing,
then rolled back: total sales, gross margin, expenses and net margin all **identical
before and after**; only the drawer moved, ₹71,373 → ₹31,373.

Each drawing gets its own voucher number, posts against Drawings (owner), and is
cancellable but never deletable. It refuses a withdrawal that would take an account
below zero — including a **backdated** one that would make any later day negative,
which was claimed and then actually tested: a ₹30,000 drawing dated 01 September was
refused because the drawer dipped to ₹6,293 on the 14th. It warns, without blocking,
when the amount exceeds the month's profit, distinguishing a draw against earlier
retained profit from one against money never earned.

### What the technician sees

Hiding drawings initially closed the Cash & bank book and the day book to
technicians altogether. That contradicted the original requirement — technician sees
everything except profit — so the owner was asked rather than it being settled by
default. He chose to give the books back.

The rule is now: **he can see the cash leaving, he cannot see what it means.** The
books open, the drawing shows as a plain money-out line, the running balance is
right. Refused by real call: the drawings list (no rows), drawing totals, recording,
pre-checking, cancelling, the Monthly summary, the Monthly report, the profit report,
and every cost column.

## The delivery date — settable at hand back, correctable afterwards

The owner sometimes hands a device back and records it later. Two changes: a date
picker on Hand back, defaulting to today, and an owner-only correction on a job
already delivered.

The second is a financial amendment, not a field edit. Every money report now runs
on bill date, so moving a delivery date moves revenue between days and between
months.

### It reused the purchase machinery, with two departures worth recording

Reused as-is: the same bill number, a revision counter, a reason, an audit row
holding before and after, and the date-aware reversal that lands both legs on the
original day rather than stamping `current_date`.

**The advance.** Reversing the whole job record would have dragged the customer's
advance along with the bill, because both sit on the same record. The reversal now
takes a bill number and moves only that bill's lines, so **an advance keeps the date
it was actually received**. Not something that had been anticipated when the
instruction was written.

**The bill.** Purchases cancel and re-post. That cannot work here: one job can only
ever have one bill, so the re-post would be refused by a rule built earlier in this
project. The date is corrected in place instead, with the old copy in the audit row
and the revision counter incremented — the same way purchase details are corrected.

**The payment takes the bill's date**, on the bill and in the cash book, so it moves
with it and the customer balance and day book stay in step.

### Proved by moving a real job across a month boundary

JOB/0110, INV/0096, ₹1,700 cash, 29 September → 1 October, then rolled back:

| | Before | After |
| --- | --- | --- |
| September — sales / spares cost / net | ₹87,880 / ₹38,046.90 / ₹29,276.10 | ₹86,180 / ₹37,488.90 / ₹28,134.10 |
| October — sales / spares cost / net | ₹32,550 / ₹15,155 / ₹17,395 | ₹34,250 / ₹15,713 / ₹18,537 |

The Monthly summary matched the Monthly report on both months, before and after.

The three cash screens still agreed: September's drawer closed at ₹69,673, 29
September closed at ₹69,523, and 1 October opened at ₹69,673, took ₹1,700 and closed
at ₹71,373 — chaining exactly into the Cash & bank book.

**A point about proving a ledger in an append-only book.** Debits and credits both
rose by ₹6,800, so the raw totals did change. They must: the reversal and the
re-post are new rows. What has to hold is the **gap** between them, and it stayed at
₹4,850. A test that demanded unchanged totals here would have been testing the wrong
thing.

### The refusals, each tried for real

> "Combo (LCD) – Oppo A17 was issued to JOB/2026-27/0086 on 03 Oct 2026. The job
> cannot be handed back before that, on 30 Sep 2026."

> "Current Account would hold ₹-350.00 on 30 Sep 2026 if this bill's ₹1,600.00
> collection moved to 01 Oct 2026…"

Plus: future date; before the job was received; across the financial year; no reason
given; the same date as now; a later payment against the bill dated before the new
date; and a closed day. The cash check tests **every day in between**, not only the
target date — the same shape as the backdated drawing check.

**Access:** a technician can set the date at hand back, since that is only recording
when he handed the device over. Moving a posted bill afterwards is refused — *"Only
the owner can change the date of a delivered job."* Both proved by real call.

### Two consequences that are correct, not faults

A move shows in the old month as money **out** on the original date rather than that
month's receipts shrinking — the same two-line pattern already agreed for
cancellations.

After a move, the September summary shows ₹558 of spares **issued in September but
billed in October**. That is what actually happened.

## The spare-issue date, and a guard that blocked real work

The delivery-date refusal — "the job cannot be handed back before the spares were issued to it" — stopped the owner working. His actual flow is to create the job, issue the spare and hand back all on one day in the system, then correct the delivery date back to when he really handed the phone over. The spare's issue date was still today, now *after* the delivery date, so the guard refused.

That issue date was never real. It was when he typed it in, not when the part went into the phone.

**Spare issue dates now follow the delivery date — by clamping, not by reassigning.** A part genuinely issued on the 20th for a phone delivered on the 25th keeps the 20th, because that is real history. Only a date that would fall *after* the delivery is pulled back to it. Proved on job 0005: two spares moved from 5 Oct to 28 Sept (2 moved, 0 kept), then moving the same job forward to 29 Sept left both on the 28th (0 moved, 2 kept).

One refusal had to survive, and did:

> "Combo (LCD) – Motorola G34 fitted on JOB/2026-27/0005 arrived in the shop on 28 Sep 2026 (PB/2026-27/0073). It cannot have gone into the phone on 27 Sep 2026. If the purchase date is wrong, correct the purchase first."

**It also fixed the cost-matching problem** recorded in the previous section: spares issued but not billed now reads ₹0 in both months, because cost lands in the month of the revenue it earned. Stock ties with a ₹0 difference in September and October, and no item goes below zero on any day.

## Advances that never reached the drawer

At intake the system wrote only "customer paid ₹X". The balancing step then invented a matching line against a placeholder called **"Job work"** — so the voucher balanced, every check passed, and the cash never arrived anywhere. A balanced voucher pointing at a placeholder is invisible to every test we had.

Two jobs were affected: JOB/0066 ₹500 and JOB/0106 ₹200. The owner confirmed he physically took the money, so his drawer was reading **₹700 light**.

| Job | Drawer, job date | Drawer, delivery date | Total | Customer |
| --- | --- | --- | --- | --- |
| 0066 | ₹500 on 18 Sep | ₹500 on 21 Sep | ₹1,000 | ₹0 → ₹0 |
| 0106 | ₹200 on 29 Sep | ₹400 on 29 Sep | ₹600 | ₹0 → ₹0 |

The double-count risk flagged beforehand did not materialise: hand back had already credited each customer with only the balance due, so both were at zero before and after. Cash Drawer ₹89,123 → ₹89,823, bank unchanged, both amounts landing in September.

Intake now has an **"Advance received into"** account, defaulting to Cash Drawer.

## The books did not balance, twice

### ₹4,850 — the opening capital that was never written

Total debits and credits differed by ₹4,850. The first explanation offered — that handing back an unpaid bill widened it — was wrong, and the agent corrected itself: its test had rolled back *before* the deferred balancing step ran, producing a false ₹1,300. All 344 numbered vouchers balanced to the paisa.

The whole gap was the 1 September opening balances, posted **without a voucher number**. The balancing step only runs on numbered vouchers, so the owner's-capital side was never written. Nothing the owner could see was affected; the missing line was capital, which no screen shows.

**The reconciliation report said clean** because it grouped by voucher and ignored unnumbered rows — blind to exactly the rows that were wrong. It now also checks total debits = total credits including unnumbered rows, and would have caught this.

### ₹2,850 — and an explanation worth rejecting

A second gap appeared. The agent proposed that the capital figure had been "entered ₹2,850 short" and asked the owner what his opening capital should be.

That was refused, because **₹2,850 is exactly SVS Mobiles' opening balance** — the supplier openings were ₹49,099, Tulsi ₹46,249 plus SVS ₹2,850, and ₹49,099 is what the capital line was derived from. Too precise to be coincidence. Three possibilities were put in order, cheapest first: is the check miscounting, was a row removed, or did the re-dating run touch an opening row.

The answer was none of them. **A row had been added.** On 6 October the owner zeroed SVS's opening balance, which correctly posted a ₹2,850 correction against SVS and left the capital line stale. He confirmed the zeroing was deliberate, so capital was the figure that needed correcting.

The lesson is the method, not the number: asking for the cheapest explanation to be ruled out first is what made the rest of the answer trustworthy, and it stopped the owner being asked to invent a capital figure he had no way of knowing.

**The real defect behind it:** changing a party's opening balance posted the party side and left capital behind. It now posts both in the same transaction, and the sweep found the same hole on money accounts — unhit only because the Cash Drawer's capital line happened to have been posted by hand.

Opening position now reads end to end: SVS ₹0, Tulsi ₹46,249, cash ₹53,949, capital ₹7,700, debits ₹56,799 = credits ₹56,799.

## Cancelled purchases, and a decision reversed

The owner reported cancelled purchases showing in his figures. The audit found **nothing counted twice** — all four were cancelled with full reversals and SVS's closing balance was correct at ₹38,270.

What made it look wrong: each reversal landed on the day of cancellation, not the bill's date. Between 1 and 8 September the running balance genuinely carried a cancelled ₹600.

**But there was a real fault underneath, and it was a decision recorded earlier in this document.** When the date-aware reversal was built for amendments, the question of applying it to cancellations was raised and answered: leave it, because a cancellation genuinely happens on the day it is done. That reasoning holds for a day book and fails for a monthly report. Two September bills cancelled in October — PB/0081 ₹700 and PB/0058 ₹900 — put **₹1,600 of September purchases and closing stock into the wrong month**.

Cancelling now reverses on the document's own date, the existing mis-dated cancellations were re-dated with 32 correcting money entries and 14 stock entries, and the earlier decision is withdrawn.

## Books check

A page under Reports, owner only, that runs every integrity check and explains in plain words what a non-zero figure means. It found the ₹4,850 and then the ₹2,850, each within days of occurring. Both would otherwise have surfaced at year end, when nobody can remember what happened.

Three additions, all built:

- **The opening entries, listed with their totals.** They carry no voucher number and no other screen can see them, which is exactly why two imbalances hid there.
- **The gap and its cause on the dashboard** — silent when there is nothing wrong, because a tile that is always present teaches people to ignore it. It names the cause ("opening entries out by ₹2,850") rather than only the amount, since a number alone sends the owner to ask rather than to look.
- **A log of every opening-balance change** — who, when, old figure, new figure. This turns "why don't my books balance" into "Satheesh changed SVS Mobiles on 6 October".

Each was proved twice: reading correctly with the books balanced, and reading correctly with the balance deliberately broken inside a rolled-back test. A diagnostic that is itself wrong costs more than the fault it was meant to catch.

Recorded honestly: the "cause not identified" branch has **never fired in a test**. Every entry falls into one of three named groups, so it should not appear. It is a safeguard, not a tested path — and if it ever shows, that in itself means something unexpected exists.

## Reporting and exports

**Day-wise / month-wise** on the Monthly report rather than a separate screen. Built as a toggle deliberately: a second report computing the same figures its own way is how two screens come to disagree, which is the fault that took a day to find. The month row is the sum of the day rows, so they cannot drift.

The owner's column list — Date, Total Sales, Total Expenses, Net Profit, Cash, UPI — had no spares cost in it. Computed as sales minus expenses, September would have read ₹84,823 against a real net of ₹46,776. A spares cost column was added so the row adds up left to right, rather than deducting ₹38,046 invisibly.

**Cash and UPI are collections, not sales**, and the page says so: a credit sale, an advance or a payment against an older bill all break the equality, and every one is legitimate.

**CSV and PDF on every stock tab**, plus a **"Hide items with no stock"** toggle defaulting on — 98 of 188 spares are empty. The file states what it is showing: "90 items with stock (98 with no stock hidden)". An export that silently differs from the screen is worse than no export, so the toggle drives both.

It also found and fixed CSV amounts exporting as text (`"₹1,234.00"`) rather than numbers, so Excel can sum them.

## Duplication audit

Asked for after a run of date moves, each writing a reversal and a repost. Worth noting that **the books balancing is not evidence against duplication** — a duplicated pair balances perfectly — so the check was for specific shapes.

Nothing is double-counted: 812 ledger lines with 60 reversals and 328 stock movements with 19 reversals all reconciled, with no orphaned reversals, nothing reversed twice, and no identical rows. Batch quantities match stock movements; each bill's party amount equals its total exactly once.

Found and left for the owner's decision: **11 spare parts existing as 23 records**, and **3 genuine duplicate customers** (identical name and phone, all at ₹0). No duplicate suppliers — the 39 purchases sharing a "bill number" were run down item by item and are separate purchases on the same day, the numbers being dates like `14092026`.

The zero-stock toggle incidentally tidies the duplicates: three parts hold stock in a single record and show once; the other eight are empty and vanish.

## Editing a posted bill

The owner asked whether invoices could be edited — purchase and sales alike — covering date, amount and items. Purchases already could. Sales could not.

Both now amend **in place, keeping the same number**, which was the owner's explicit choice over cancel-and-reissue. A bill number that a customer is holding should not change because a quantity was wrong. Each amendment bumps a revision counter and leaves its reversal and repost in the ledger, so the number stays stable while the history stays honest.

On a sales bill the editable surface is quantities, rates, adding and removing spares, service charges and the discount. Stock follows the edit — a removed spare goes back, an added one comes out, FIFO batches reconsumed in order — and the party ledger, the day book and both cash screens restate against the bill's own date, not today's.

**Payment method is not editable.** It was offered and not taken up. Worth saying why it is not a cosmetic field: switching a bill from cash to UPI moves money between the drawer and the bank, so the day's cash count stops matching the books until someone recounts. It needs its own posting, not a dropdown.

## Where did this part come from

The owner opened Movement history on a Vivo Y20 combo and asked the plainest possible question: where was this bought. The screen could not answer it. It showed four rows with types and dates and no document behind any of them.

Every row now names its source: a purchase gives the PB number, the supplier and the supplier's own bill number; an issue gives the job number and the customer; opening stock says "on hand when the books started"; adjustments, transfers and returns each carry their own label. The job number opens the job. The purchase number opens the bill **only for someone allowed to see costs** — a technician gets the text, not the link. CSV and PDF gained Supplier/customer and Detail columns.

The four rows resolved to one purchase on 1 September and one issue, re-dated from 6 October to 30 September by the spare-issue clamp. Correct, and now legible as such.

Checked: the labels on screen, and a technician seeing the supplier with the rate column blank on every row while the owner saw ₹600. Not checked at the time of writing: the click-throughs, the new download columns, and the labels for reversed purchases, returns, adjustments, transfers and opening stock, none of which had been rendered against real data.

## The profit figure, and where it was actually wrong

The hide-cancelled sweep was sent to look at cancellations. Walking every screen under the 360 rule, it found something unrelated and worse: the Finance profit-and-loss was reading spares cost as **₹0**, showing September net profit of **₹105,073** against a real **₹48,754.10**.

**Root cause, not a patched number.** Three places worked profit out from the cost recorded on each bill line. Job bill lines never carry a cost — all 267 lines since the first bill on 11 September record ₹0. The real cost sits on the job's issued parts, which is where the monthly report had always read it. One shared spares-cost rule now feeds every profit figure, the monthly report included.

It was **not** a permission failure returning zero instead of refusing. That was worth ruling out explicitly: a cost gate that answers ₹0 rather than "not allowed" would be a quiet, repeatable way to put a wrong number in front of someone. A technician gets a blank, confirmed by a real call.

September, before and after: profit-and-loss ₹105,073 → ₹48,754.10 (spares cost ₹0 → ₹56,318.90); sales report gross margin ₹125,980 → ₹69,311.10; October ₹26,430 → ₹16,349. The monthly report, day-wise, month-wise, monthly summary, profit report and drawings profit read ₹48,754.10 both before and after — they were always right. Every job's margin summed gives ₹56,318.90 spares cost, matching.

**Where the owner was actually exposed.** Not the profit-and-loss: that page was taken off the screens on 7 September, four days before the first bill, so nobody ever saw ₹105,073. The live fault was the **"Cost · Profit" line on the bill screen**, which since 11 September has shown every job bill at cost ₹0 and profit equal to the whole bill — INV/0134 read cost ₹0, profit ₹1,500 where the truth is cost ₹600, profit ₹900. That is the one to tell him about, and the distinction was worth getting right: a ₹56,000 error on a hidden page and a per-bill error on a page he uses every day are different conversations.

**A cost of the fix, stated:** job bill lines are not linked to a spare, so per-item and per-brand margin columns now show blank rather than a false full margin.

## Money due from customers

Dashboard ₹6,550 against reconciliation ₹6,700. Traced to one document rather than guessed at: **Prakash, INV/2026-27/0130, ₹150**, paid on 30 September through receipt RCP/2026-27/0002 which was never matched to the bill. His ledger correctly showed ₹0 owed, so the dashboard was right and the reconciliation was counting a paid bill as unpaid. A receipt not tied to a bill now settles what that customer owes. Both read ₹6,550; suppliers agree at ₹59,731; the Books check gap is ₹0.00.

His bill still shows "Balance ₹150" on the bill screen — matching that receipt to the bill is the owner's action, so nothing was changed there.

A ₹5 movement in the dashboard figure between two readings (₹6,555 → ₹6,550) is **not yet traced**.

## A purchase cost leak, found by asking the right question

The movement-history round ended with the agent volunteering that a technician can reach the purchases page on his own — which would make hiding the link on a movement row cosmetic. Chased, and it was half right.

The purchases page itself is closed at the database: totals, line amounts and per-line money all refused, leaving supplier, bill number, date, items and quantities. But the **purchase report handed a technician September's total (₹58,701), every bill's total and per-item purchase values** — ₹4,630 for a OnePlus frame among them. The page is not on the screens, but anyone signed in could ask the database for it directly. Now refused: "You do not have permission to view purchase cost". The owner still gets ₹58,701.

This is the fifth gap of the same shape. The pattern is consistent enough to state as a rule: a screen being off the menu is not a permission.

## SVS Mobiles, ₹38,270 to ₹37,650

Noticed and left untraced in one round, traced in the next rather than written off as "probably real shop activity": PB/2026-27/0061 amended from ₹1,600 to ₹750 at 10:28, and PB/2026-27/0091 entered at ₹230 at 10:43. ₹38,270 − ₹850 + ₹230 = ₹37,650. Both real.

## The cancelled part that came back on the wrong day

The owner tried to move JOB/2026-27/0160 from 7 October to 30 September and was refused: "would leave -1.000 in stock on 30 Sep 2026. Check the purchase date of this part."

I guessed the purchase date was wrong — the familiar story of a part taken in September and billed in October. **That guess was wrong, and the agent disproved it rather than confirming it.** There is one purchase, PB/2026-27/0061, bill dated 23 Sep, supplier bill number 23092026, keyed in on 28 Sep. Nothing about it needed changing.

What it found instead: **when a job is cancelled, the money is reversed on the bill's own date, but the spare goes back into stock dated today.** One repair for Masilamani had been booked three times — fitted 28 Sep and paid ₹1,600, cancelled this morning to correct a rate, re-billed as JOB/0159, cancelled as a duplicate, re-billed as JOB/0160. The cancellation returned the money to 28 September and the part to 7 October, so for nine days the books showed the part consumed with no sale behind it. Stock on 30 September read 0. The guard was right about the books; the books were wrong about the shop.

**This was the remainder of my own mistake.** I took the position that cancellation reversals could sit on today's date, we found out it was wrong when ₹1,600 of September purchases landed in October, and I fixed it for purchases only. Job parts were the same rule in a second place. The pattern is worth stating: when a dating rule is wrong once, the fix has to be chased everywhere the rule applies, not only where it was caught. The agent took that on itself for future work without being asked to.

**Fixed** at the cancel path, plus an outside-repair charge reversal and an unused older reversal tool that had the same default. Two late returns were re-dated by reverse-and-repost, nothing deleted: JOB/0085 from 7 Oct to 28 Sep, JOB/0031 from 17 Sep to 16 Sep.

**Deliberately still on today's date,** because each is a real event on the day it happens: handing back an unused part, logging a customer's faulty part, sending or receiving an outside repair, receiving a stock transfer, and a bounced cheque.

September closing stock ₹28,709.60 → ₹30,309.60, being the unit that was genuinely on the shelf. The "issued but not billed" mismatch went from ₹1,600 in September and −₹1,600 in October to **₹0 in both**. Purchases, profit and the Books check gap did not move. No spare is negative on any day from 1 April to today.

Proved by running the owner's real action and undoing it, not by reasoning: JOB/0160 to 28 Sep allowed, JOB/0064 to 10 Sep allowed, and as a control JOB/0003 to 1 Sep still correctly refused.

**The list, built so the owner meets this once rather than one dialog at a time:** 18 delivered jobs cannot move all the way back to their booking date, each with the earliest date it can legally take. Most are legitimate — the part was not there yet, or earlier units had gone to other jobs.

## Correcting how a bill was paid

Asked for after the sales bill became editable: a bill is sometimes keyed as cash when the customer paid by UPI. The total is right and the money is in the wrong place.

Cash, UPI, a split of the two, and the amount received — which covers fully paid, part paid and not paid. The amount was included deliberately rather than split into a second feature: a mis-keyed payment is nearly always wrong method *or* wrong amount, and separating them would mean editing the same bill twice.

**Only the leg that changes is reversed and re-posted, on the bill's own date.** In a split bill where only the UPI part moves, the cash part is untouched. **Advances are left completely alone** — an advance carries its own receipt number and date, and only entries carrying the bill's number are touched.

Four directions proved on real bills inside tests that were then removed. INV/0067, 21 Sep, cash → UPI ₹500: day book cash out 80 → 580 and UPI in 300 → 800; drawer closing 65,643 → 65,143 and bank 17,750 → 18,250; September month-wise 87,580 / 34,500 → 87,080 / 35,000; monthly summary closing 88,423 / 12,640 → 87,923 / 13,140. Sales and net profit did not move, which is the point. INV/0132, paid → unpaid, left the customer owing ₹1,200 and the ledger showed it.

Refusals proved by real calls, not by reading the code: cancelled bill, overpayment, card payment, missing reason. A technician was refused at the database on the amend action, on the inner steps called directly, on the change log, and on writing to the payment records — four ways in, all closed, because a hidden button is not a permission.

Row counts identical before and after on all four tables: no residue.

**Not checked:** the screen itself, the printed bill, the WhatsApp message and the report downloads. All of them need a signed-in owner view, which this project still cannot produce.

## Two service items that were pretending to be stock

Found while clearing the date-move blocks: "Rework service Charge" was set up as a stocked spare, so a job carrying it was refused a date move like any other part. Five items turned out to be in that state. The owner converted two — **General service** and **General service - Samsung A35**, both ₹1 placeholders — and deliberately left three alone: Rework service Charge, Software - Apple 6s and Software - Samsung M01 core carry a real bought-in cost of ₹2,100 that today correctly reaches the job's margin. As plain services that cost would stop reaching it and reported profit would rise by ₹2,100 unless booked as an expense instead. The price of leaving them as stock is that JOB/0116 cannot be billed before 3 October — which is right, since that is the day the rework was bought.

**The thing worth proving was not the conversion.** It was that the owner can still charge ₹50 for a general service afterwards. A conversion that tidied the item list and broke his ability to bill labour would be worse than the problem it fixed. Proved by adding General service ₹50 to a real job and delivering it: the bill came out at ₹550 with a "service ₹50" line and no stock touched, the same shape as the bills already printed. The test job was returned to Ready for delivery and the bill removed.

**History kept.** All three movements on each item are untouched — bought on PB/2026-27/0006 and /0008 from Sathya V Connect on 1 Sep, issued, returned the next day. Services drop out of the spares picker, so buttons were added beneath it to open their past movements rather than stranding them.

**Loose ends closed in the same pass:** count sheets, stock adjustments and transfers no longer offer service items, and issuing a service to a job as a spare is now refused with "Add it as a labour or service charge." Low-stock warnings and dead stock were checked and unaffected.

**A ₹2 disagreement this created, stated rather than buried:** the two ₹1 units still exist, so the Current stock page (which drops services) reads ₹29,268.60 while the monthly summary and stock reports still count them at ₹29,270.60. Cancelling the two placeholder purchases closes it. Left for the owner.

**It also corrected the outstanding-work list in these docs.** JOB/0006 was recorded here as waiting to be handed back with General service ×1 at ₹50. It was delivered on 15 September and billed as INV/2026-27/0005, the ₹50 charged as labour rather than as the item. Neither that bill nor JOB/0008's ever carried the item, so nothing printed changes — but a list of outstanding work is worth no more than its last check against the data.
