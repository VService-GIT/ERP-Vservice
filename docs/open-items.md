# Open Items and Risks

## Needs the owner

- **Run one job end to end on the published site.** This is the main outstanding
  item. The loop was verified before the data purge, but not after it, and the
  workflow has since been rewritten from sixteen statuses to four — touching the
  status enum, the transition rules and the delivery path in one change. That
  rewrite is compile-verified only, because no authenticated session can be created
  against the project's Supabase from the build environment. Book one device from
  intake through the three steps to delivery, with a spare, a service charge and a
  payment, and check the bill, the WhatsApp link and the Invoice tab.

- **Enable leaked-password protection** (see Medium, below).

- **Create the technician login.** Users & Roles → Invite staff, Technician role.
  Nothing is emailed; a one-time password is shown on screen to pass on. Both
  current logins are owners.

- **Supply the bank figures** — opening balance as at 1 September, and the bank
  name, account number, IFSC and branch. The account exists with those fields
  deliberately blank; nothing was invented.

- **Set reorder levels.** All 80 spares sit at 0, so low-stock warnings never fire
  and the reorder list is permanently empty — the feature is present but inert.

- **Set selling rates.** Every spare is priced at cost, so spares bill at zero
  margin and the profit report will show labour only.

- **Confirm navigation speed on the published URL.** The 5-second delay was traced
  to the unpublished dev preview compiling each screen on click. Publishing should
  resolve it, but this has not been confirmed from the shop's own connection.

- **Check the reconciliation report once.** Its four problem counts can only be read
  from a signed-in screen. Every figure feeding it is clean.

## Verification status

Verified by live API calls with real logins (commit `ba90023`, re-confirmed
`df0861f`):

- Technician receives 403 or empty on profit reports, per-job profitability,
  purchase bills, supplier records and all cost-rate fields; owner receives them.
- Over-issuing a spare beyond available stock throws — negative stock is
  impossible.
- Estimate over-run blocks delivery until re-approval is recorded.
- GST-off forces `non_gst` with zero tax, and raw tax input is rejected.
- `wa.me` links normalise 10-digit numbers to +91.
- Job → 2 spares → labour → ready → delivered with split payment: stock updated,
  ledger balanced, invoice settled. Test record reversed via `cancel_sale`.

Verified with a caveat:

- **`finance_reconcile` was not run as the owner after the purge.** The RPC is
  owner-gated and no session could be created, so the equivalent aggregate query
  was run directly instead. It returned zero receivables, zero payables and zero
  mismatches, over-allocated bills, unbalanced journals and unbalanced vouchers.
  With every transactional table empty the result is sound, but it is an
  emulation of the call rather than the call itself.

## Accounts

| Login | Role | Note |
| --- | --- | --- |
| Appsmdass@gmail.com | owner | Developer. Protected at the database — cannot be deleted, deactivated, downgraded, or have its password reset by anyone else. |
| satheesh.ns30@gmail.com | owner | Shop owner. Promoted from technician. |

Both accounts see cost and profit. There is deliberately no technician login yet.

## Opening position as at 1 September 2026

| | |
| --- | --- |
| Cash in hand | ₹53,949.00 |
| Bank | ₹0.00 — pending |
| Spares stock at cost | ₹27,327.50 · 1,599 units · 80 items |
| Owed to suppliers | ₹49,099.00 |
| Owed by customers | ₹0.00 |

Data problems in the owner's source records, flagged rather than guessed, and worth
resolving with the suppliers concerned: Chandan Mobile Shop has no pincode (the
source held the Karnataka state code); RS Communication's second number was
truncated; five records had "Owner" or the business name in place of a contact
name; two city names were corrected against their pincodes.

## Bugs found and fixed during verification

These were reported as complete by the phase that introduced them, and were not.
They are recorded because they show which claims needed independent checking.

1. **Technician had full rights to every module**, including supplier bills and
   purchase rates — the exact leak the role split existed to prevent. Phase A
   reported cost-gating as done; it was not.
2. **Delivery and billing was broken** by a type error in
   `materialise_job_invoice`. The core function of the application did not work
   after Phase C.
3. **Job number collisions** between test and live data. The global uniqueness
   constraint was replaced with a composite `(branch_id, job_no)` index.
4. **GST wording leaked into ledger labels** with GST off ("Sales & output GST").
   Now reads "Sales" / "Purchases".
5. **The reset-password dialog applied the change on open.** Merely opening it to
   look would have broken that person's login. Nothing is applied now until
   confirmed.
6. **The Active column made "protected" look like "switched off".** The Master
   Admin toggle rendered grey because the guard disabled it, which was
   indistinguishable from a deactivated account — on the one row where the
   developer most needs certainty. Now an explicit "Active · Protected" badge.
7. **The invite dialog promised an email that is never sent.** No mail service is
   configured; a one-time password is shown instead. Anyone following the old
   wording would have waited for an invite that never arrives.
8. **Opening balances could not be set on an existing party.** The attempt was
   refused, and the reconciliation counted only supplier bills — so the amount
   would never have appeared as a payable even if it had saved.
9. **Saving an edited job sheet failed** while writing its history line — the
   feature broken on arrival, caught only because the flow was actually run.
10. **The customer lookup on a new job sheet ignored the active flag**, so a
    deactivated customer still surfaced at the counter.
11. **No cash or bank ledger was reachable.** `CashBankBook` sat unwired in the
    codebase while the owner had no way to tally his drawer or reconcile a
    passbook.

## A note on testing against live data

Six test rows — two opening-balance pairs and a receipt with its reversal — were
left visible in the owner's cash ledger, and he found them before we did. Reversing
a test is not cleaning up: it leaves two rows where there should be none. Tests
against a live database must be removed completely, or the inability to remove them
stated plainly rather than left for the owner to discover.

## Known issues

- **Report performance at volume.** Benchmarking showed `report_inventory` at
  roughly 2.1s and `report_finance` at roughly 1.5s of database time against the
  seeded volume, driven by row count rather than per-row helpers. Irrelevant at a
  single shop's real volume; if it ever matters, the cheap fix is lazy per-section
  fetching rather than maintained rollups, which would introduce staleness.

## Deliberately retained

- **Retail sales code and tables.** Hidden and route-blocked, not deleted. The job
  delivery bill posts through the same invoice engine, and retaining the code keeps
  counter sales possible later without a rebuild.
- **`branch_id` throughout the schema.** Invisible in the UI, but load-bearing
  across the ledger and stock tables.
- **The serial/IMEI engine.** Retained in full, now scoped to the customer's device
  on a job sheet rather than to stock.
