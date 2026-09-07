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
