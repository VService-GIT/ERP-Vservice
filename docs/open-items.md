# Open Items and Risks

## Needs the owner

- **Sign in as `Ramesh Owner` and confirm it works.** Once confirmed, the
  `Master Admin` login should be deleted. It currently holds owner + technician
  roles, meaning **two accounts can see profit figures** where the requirement
  says one. It was retained only as a fallback in case `Master Admin` is the
  account the owner personally uses.

- **End-to-end test on the clean database has not been run.** Creating a job sheet
  requires an authenticated owner session, and the project's Supabase instance
  cannot have a session minted from the build environment. The full
  job → spares → delivery → payment loop was verified before the purge, but not
  after it. This should be done once before real use.

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
