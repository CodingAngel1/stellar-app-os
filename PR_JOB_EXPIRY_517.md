# feat(escrow): Planter job expiry — auto-cancel unaccepted jobs after 14 days (#517)

Closes #517.

## Problem

The single-donor `tree-escrow` flow let a sponsor deposit funds for a
farmer, but there was no upper bound on how long the funds could sit in
the `Funded` state waiting for the planter to begin work. If the
planter never started, the donor's funds were effectively locked. The
sponsor only had two recourses:

* call `refund()` themselves (admin-gated)
* watch the funds remain indefinitely

## Solution

Add a configurable deadline mechanism that any off-chain cron may crank
to refund the donor after 14 days. The "first `verify_progress`" call
is treated as planter acceptance — once planting has begun, the
deadline no longer applies.

## What changed

### `contracts/tree-escrow/src/lib.rs`

1. **New constant** — `FOURTEEN_DAYS_SECS = 60 * 60 * 24 * 14`
   (= 1 209 600 sec).

2. **New storage key** — `DataKey::JobDeadline(Address)`. Deliberately
   stored as a **separate key** rather than a field on `EscrowRecord`:
   Soroban XDR serialisation of `#[contracttype]` structs is fixed at
   deploy time, so adding a field would break deserialisation of
   previously-deployed storage rows. A separate key is shape-stable
   across redeployments.

3. **Deadline recorded at deposit time** — both `deposit_internal`
   (called by `deposit` and `sponsor_as_gift`) and `batch_deposit`
   write `JobDeadline(farmer) = now + FOURTEEN_DAYS_SECS` immediately
   after the token transfer.

4. **New function `expire_job(env, farmer)`** — anyone can call.

   ```
   pub fn expire_job(env: Env, farmer: Address) {
       let deadline = load(&JobDeadline(farmer))?;
       let rec     = load(&Escrow(farmer))?;

       if rec.status == Refunded      → panic!("job already refunded");
       if rec.status != Funded        → panic!("job already accepted (planting in progress)");
       if env.ledger().timestamp() < deadline
                                     → panic!("job not expired yet");

       token::transfer(contract → rec.donor, rec.total_amount);
       rec.status = Refunded;
       save(&Escrow(farmer), rec);
       remove(&JobDeadline(farmer));
       events.publish(("jobexp", farmer), (donor, amount, deadline));
   }
   ```

   * **No `require_auth`**: anyone-crankable so off-chain keepers /
     bots can run it. The only effect is a refund to the original
     donor.
   * **CEI ordering**: storage loaded → checks → token transfer →
     storage written → deadline key removed. If the transfer panics,
     no state is mutated.
   * **Refund routes to `rec.donor`** (the sponsor). For
     `sponsor_as_gift` flows, the `gift_recipient` is the NFT/credit
     beneficiary, not the funds source — so donor is still the correct
     refund target.

5. **Two new getters**

   * `get_job_deadline(env, farmer) -> Option<u64>` — for off-chain
     UIs to display the deadline.
   * `get_job_expiry_secs(env) -> u64` — returns the 14-day constant
     so SDK clients don't hard-code it.

6. **Symmetric cleanup in `refund()`** — manual refund by the admin
   now also removes the `JobDeadline` key to avoid orphaned storage
   rent. Brings the manual and automatic paths into symmetry.

## Backwards compatibility

* `EscrowRecord` schema is **unchanged**, so existing on-chain escrow
  rows are still readable. New deposits write the new key; old deposits
  have no deadline and `expire_job` will panic with
  `"no job deadline on file for farmer"` if invoked against them (the
  funds remain recoverable through `refund()`).
* `sponsor_as_gift` inherits the change automatically — it delegates
  to `deposit_internal`.
* No new `HarvestaError` codes were added; panics use distinct,
  stable string literals (`"job not expired yet"`,
  `"job already refunded"`, `"job already accepted (planting in progress)"`,
  `"no job deadline on file for farmer"`, `"no escrow for farmer"`).

## Tests

Eleven new unit tests under
`#[cfg(test)] mod tests` in `contracts/tree-escrow/src/lib.rs`:

| Test                                                              | What it covers                                                  |
|-------------------------------------------------------------------|-----------------------------------------------------------------|
| `test_deposit_records_deadline_at_now_plus_14_days`               | Single deposit → deadline exactly `ts + 14d`.                    |
| `test_expire_job_refunds_donor_after_14_days`                     | Donor refunded, contract balance restored, status `Refunded`, deadline key removed. |
| `test_expire_job_panics_one_second_before_deadline`              | Off-by-one boundary: 14d−1s → panic.                            |
| `test_expire_job_panics_at_exact_deadline_minus_one`             | Tiny advance (1s) still insufficient.                            |
| `test_expire_job_panics_after_first_progress_update`             | After first `verify_progress` (status → `Planted`), panic.     |
| `test_expire_job_panics_when_called_twice`                        | Idempotent: second call panics with `"job already refunded"`.   |
| `test_expire_job_panics_after_admin_refund`                       | Manual admin refund then `expire_job` → panic with same msg.   |
| `test_expire_job_panics_for_unknown_farmer`                      | No deadline on file → panic.                                    |
| `test_expire_job_scoped_to_one_farmer_other_escrows_unaffected`  | Expiring `farmer1` leaves `farmer2`'s `Funded` escrow + deadline intact. |
| `test_get_job_expiry_secs_is_fourteen_days`                      | Public constant matches `14 * 24 * 60 * 60`.                    |
| `test_batch_deposit_records_per_farmer_deadlines`                | Batch deposit writes one deadline per slot.                     |

The tests use self-contained `setup_expiry_test()` /
`setup_funded_expiry_test()` helpers that avoid the existing
double-`initialize()` quirks in `setup()`. They are CI-isolated and
do not touch legacy fixtures.

## Event surface (new)

* `( "jobexp", farmer )` → `( donor, total_amount, deadline_secs )`

Off-chain indexers should listen for this topic to surface
"Planter job expired — refund issued" notifications in the dashboard.

## Out of scope / future work

* The co-funded flow (`register_tree`/`contribute`/`release_proportional`)
  already pre-assigns a farmer at registration, so "open job postings"
  with a deadline don't apply. If we ever add an "open posting" flow
  (#494, #517's sibling), the same `JobDeadline` pattern can be lifted
  into that contract.
* An admin-configurable deadline length (rather than the hard-coded
  14 days) is a candidate for a follow-up issue; the current shape
  keeps the constant out of contract storage on purpose.

## Risk

* **Cosmetic**: existing escrow rows in production have no deadline;
  `expire_job` against them panics cleanly. Funds remain recoverable
  via `refund()`.
* **CEI**: token transfer precedes storage writes — if the transfer
  fails, state is unchanged.
* **Auth**: `expire_job` is anyone-crankable. Mitigated by the fact
  that the only effect is a refund to the recorded donor.

## CI

```bash
cargo test -p tree-escrow --all-targets
```

Not exercised in this PR's environment (no rust toolchain); CI on
GitHub Actions is the source-of-truth validator and should pass clean
given the self-contained test harness.
