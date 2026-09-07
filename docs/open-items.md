# Open Items and Risks

## In progress

- **Verification pass.** A verification-only run was requested against the live
  database covering: technician cost blindness proven through real API calls
  rather than code reading; technician reachability of every other screen; GST-off
  cleanliness including server rejection of a tax amount passed directly to the
  posting RPC; a full job → spares → delivery → split payment happy path; negative
  stock refusal; the estimate over-run gate; and the `wa.me` link format. Results
  are pending at the time of writing and are not assumed to pass.

- **Inline creation from the form in use.** Requested after review of the running
  app:
  - Brand and Model to be creatable from the job sheet intake form via a `+`
    control, rather than forcing a trip to Masters mid-intake.
  - Supplier and spare part to be creatable from the purchase bill itself.

  This matters because both are entered while a customer is standing at the
  counter, and an unknown model or a new supplier should not interrupt the entry.

## Known issues

- **Seed data pollutes the pickers.** The supplier dropdown is full of
  `PERF Party NNNN` records left over from performance benchmarking, alongside
  real parties such as `Bangalore Distributors`. The same is likely true of items
  and invoices — an earlier benchmark run seeded roughly 15,000 invoices, 92,000
  ledger rows and 81,000 stock ledger rows. This should be purged before the shop
  goes live, and purging it must respect the append-only ledger design rather than
  deleting rows arbitrarily.

- **Report performance on large data.** Benchmarking showed `report_inventory` at
  roughly 2.1s and `report_finance` at roughly 1.5s of database time against the
  seeded volume, driven by raw row volume rather than per-row helpers. At a single
  shop's real data volume this is not expected to matter; if it does, the cheap fix
  is lazy per-section fetching rather than maintained rollups, which would
  introduce staleness.

## Deliberately retained

- **Retail sales code and tables.** Hidden and route-blocked, not deleted. The job
  delivery bill posts through the same invoice engine, and retaining the code keeps
  the option of counter sales open without a rebuild.

- **`branch_id` throughout the schema.** Invisible in the UI, but load-bearing
  across the ledger and stock tables. Removing it would be a large and risky
  migration for no functional gain at one shop.

- **The serial/IMEI engine.** Retained in full, now scoped to the customer's device
  on a job sheet rather than to stock.
