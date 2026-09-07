# Requirement Review — Mobile Service Centre

## The requirement

A single mobile service centre. Service only, no retail sales.

Daily loop: customer arrives → job sheet created → product diagnosed → repair
completed and ready to deliver → delivered, billed for spare cost plus service
charge.

Stated requirements:

1. Service spares stock maintenance
2. Party ledger maintenance
3. Settings switch for **with GST** or **without GST**; when without GST, the whole
   app optimised for it
4. Delivery invoice shareable on WhatsApp
5. Spares purchase adds to stock; spares used on a job sheet reduce stock

Constraints:

- One firm, no branches
- Two users only: **Owner** (sees everything) and **Technician** (sees everything
  except the profit report)

## What the project already had

The project was built as a *mobile retail plus service* ERP for India, and was
substantially more complete than its brief suggested. Against the requirement:

| Requirement | State before review |
| --- | --- |
| Job sheet lifecycle | Present — 16-state workflow, QC gate, delivery OTP, service warranty |
| Delivery produces a bill | Present — `sales_invoices` carried `source='job'` and `job_id` |
| Spares stock | Present — append-only stock ledger, batches, low stock, ageing, valuation |
| Spares purchase adds stock | Present — purchase entry posted directly to the stock ledger |
| Spares reduce stock on issue | Present — `job_parts` with issue / return / faulty_return / reversal |
| Party ledger | Present — running balance, bill-wise settlement, ageing, credit limit and days |
| WhatsApp share | Partial — existed for the job acceptance slip only, not the bill |
| Billing without GST | Partial — `invoice_type` supported `non_gst` per invoice, but nothing more |

Architecture worth preserving, and preserved: money as `numeric(14,2)`, quantities
as `numeric(14,3)`, posting logic in `SECURITY DEFINER` RPCs running in a single
transaction, append-only `stock_ledger` and `ledger_entries` with reversal rows for
corrections, posted vouchers cancelled rather than deleted, RLS on every table,
roles in a separate `user_roles` table behind `has_permission()`, atomic document
numbering, and HSL design tokens.

## Gaps found

1. **No shop-level GST switch.** A per-invoice `non_gst` type existed, but there was
   no global setting, and nothing was optimised for GST-off — GSTIN, HSN/SAC,
   place of supply, CGST/SGST/IGST columns and GST reports all still rendered.
2. **Seven roles, not two** — owner, admin, manager, accountant, technician,
   counter, viewer.
3. **Retail Sales/POS was a first-class module** — counter billing, IMEI phone
   sales, exchange value, sales returns. Out of scope for a service centre.
4. **Multi-branch and multi-warehouse throughout the UI** — branch pickers,
   warehouse selectors, stock transfers.
5. **No WhatsApp share on the delivery bill** — only on the intake slip.
6. **Dashboard was retail-oriented**, not workshop-oriented.
7. **Spares were modelled like devices** — a battery or a display was pushed
   through the same serial/IMEI machinery as a phone.

## Added beyond the stated requirement

These were not asked for. They were added because the workflow is incorrect or
exploitable without them.

- **Spare masters separated from device masters.** Spares are quantity-tracked
  only, with reorder level and rack location. Requiring an IMEI for a screen is
  friction with no payoff.
- **Technician cost blindness, not just report hiding.** Hiding a "Profit Report"
  page is not enough — a technician who can see spare purchase rates, purchase
  bills or stock valuation can derive margin. Cost is gated at the database, not
  in the UI.
- **Negative stock made impossible** in the posting RPC, not merely discouraged in
  the UI.
- **Estimate over-run re-approval.** If the final bill exceeds the approved
  estimate beyond a configurable tolerance (default 10%), delivery is blocked
  until a re-approval is recorded. This is the most common source of disputes at
  the counter.
- **One job, one bill**, enforced server-side. A second bill for the same job is
  refused.
- **Delivery of an unbilled job refused** unless explicitly marked warranty or
  no-charge, so nothing leaves the shop unbilled by accident.
- **Defective spare bucket.** A spare that failed leaves sellable stock into a
  defective bucket pending return to the supplier, rather than silently vanishing.
- **Purchase returns** to send wrong or defective spares back to the supplier.
- **Unclaimed device tracking** — devices sitting past the ready-for-delivery
  threshold, with a WhatsApp reminder.
- **Day book** so the owner can tally the cash drawer at closing.
- **WhatsApp templates for all four touch points** — intake, ready for delivery,
  bill, and payment reminder — editable by the owner rather than hard-coded.
