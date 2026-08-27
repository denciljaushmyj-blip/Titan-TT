# Nickel Wiping Zone 1 and Zone 2 Accept Tray Top-Partial Capacity Assessment

## 1. Current Behavior

In Nickel Wiping partial rejection, the Accept Tray section is generated from the backend allocation response returned by the shared `nq_action` API.

Observed bad patterns:

- Normal tray accept quantity example: `8, 19, 20, 20...`
- Jumbo tray accept quantity example: `4, 11, 12, 12...`

This means the UI can show two partial Accept tray quantities: the original Top tray quantity plus a second partial remainder created by subtracting the rejected quantity from another tray.

The backend currently accepts this same pattern during submit because submit validation recomputes the same allocation and compares the submitted Accept tray rows against it.

## 2. Expected Behavior

Accept tray quantities after partial rejection should be generated from accepted quantity and the actual Accept tray max capacity:

- Normal max capacity: `20`
- Jumbo max capacity: `12`
- Only the Top tray may be partial.
- Every remaining Accept tray after Top must be full max capacity.
- No Accept tray may exceed max capacity.

Using pure capacity arithmetic:

- Normal `287` with capacity `20` gives `287 % 20 = 7`, so the capacity-shaped distribution is `7, 20, 20...`
- Jumbo `147` with capacity `12` gives `147 % 12 = 3`, so the capacity-shaped distribution is `3, 12, 12...`

If the business expects the first displayed Top value to remain `8` or `4` due to upstream tray identity rules, the core rule still remains: there must not be a second partial `19` or `11`.

## 3. Zone 1 Flow

Frontend:

- `static/templates/Nickel_Inspection/Nickel_PickTable.html`
- `fetchAllocation()` posts `action: 'ALLOCATE'` to `API_BASE + 'action/'`.
- It stores `data.accept_slots`, `data.accept_auto_trays`, `data.reject_slots`, `data.delink_slots`, and renders the allocation.
- `renderAllocation()` renders Accept slots using backend `slot.qty` directly in the `x Qty` label and in the input `data-qty`.
- Submit collects Accept tray rows from `.nq-acc-tray-input` using the same `data-qty`.
- Draft save stores `accept_slots` and `accept_auto_trays`, so a bad allocation can also be restored from draft state.

Backend:

- `Nickel_Inspection/urls.py` maps `api/action/` to `Nickel_Inspection.views.nq_action`.
- `Nickel_Inspection/views.py::nq_action` handles `ALLOCATE`.
- It calculates `accepted_qty = total_qty - rejected_qty`.
- It resolves `orig_cap = _nq_tray_capacity(...)`, but this value is not passed into the allocation builder.
- It resolves reject tray prefix/capacity with `get_nickel_wiping_rejection_tray_allocation(...)`.
- It loads original trays with `_nq_get_original_trays_for_allocation(...)`.
- It calls `build_nq_rejection_allocation(orig_trays, rejected_qty, rej_cap)`.

Submit:

- `Nickel_Inspection/views.py::_nq_do_submit_reject` recomputes the same allocation.
- It validates Accept trays with `normalize_accept_trays(accept_trays, allocation['accept_auto_trays'], original_trays=orig_trays, ...)`.
- It validates only total accepted quantity, not the rule that only Top may be partial and all later Accept trays must be full capacity.
- It persists the accepted rows into `NickelQcTrayId`, `NickelQC_Submission.accept_trays_data`, `NickelQC_PartialAcceptLot.trays_snapshot`, and `NickelWiping_PartialAcceptRecord.accept_trays`.

## 4. Zone 2 Flow

Frontend:

- `static/templates/Nickel_Inspection - Zone_two/Nickel_PickTable_zone_two.html`
- Zone 2 has the same `fetchAllocation()` and `renderAllocation()` pattern as Zone 1.
- It renders `data.accept_slots` directly and submits/saves the same slot quantities.

Backend:

- `nickel_inspection_zone_two/urls.py` maps `api/action/` to the same shared `Nickel_Inspection.views.nq_action`.
- `nickel_inspection_zone_two/views.py` imports `nq_action` from `Nickel_Inspection.views`.
- Therefore Zone 2 uses the same backend allocation and submit validation functions as Zone 1.

This is a shared backend defect, not a separate Zone 2-only implementation issue.

## 5. Relevant Files and Functions

Files involved in the defective flow:

- `Nickel_Inspection/services.py`
  - `get_nickel_wiping_rejection_tray_allocation`
  - `build_nq_rejection_allocation`
  - `_mark_top_by_smallest_qty`
  - `normalize_accept_trays`
  - `validate_original_tray_coverage`
- `Nickel_Inspection/views.py`
  - `_nq_tray_capacity`
  - `_nq_get_original_trays_for_allocation`
  - `nq_action`
  - `_nq_do_submit_reject`
  - `_nq_build_draft_snapshot`
- `nickel_inspection_zone_two/views.py`
  - imports shared `nq_action`
  - defines matching `_nq_tray_capacity` for display/list context
- `nickel_inspection_zone_two/urls.py`
  - routes Zone 2 `api/action/` to shared `nq_action`
- `static/templates/Nickel_Inspection/Nickel_PickTable.html`
  - `fetchAllocation`
  - `renderAllocation`
  - submit payload collection
  - draft save/restore
- `static/templates/Nickel_Inspection - Zone_two/Nickel_PickTable_zone_two.html`
  - same frontend allocation/render/submit/draft behavior as Zone 1

## 6. Tray Capacity Source

Accept tray capacity is available in the view layer:

- `Nickel_Inspection/views.py::_nq_tray_capacity`
- `nickel_inspection_zone_two/views.py::_nq_tray_capacity`

Both functions document and enforce:

- Normal / `NR` / `NB` / `ND`: `20`
- Jumbo / `JB`: `12`
- fallback to `InprocessInspectionTrayCapacity`
- fallback to `TrayType.tray_capacity`

Model-master tray metadata is also available through:

- `Jig_Unloading/tray_utils.py::get_model_master_tray_info`
- `JigUnloadAfterTable.tray_type`
- `JigUnloadAfterTable.tray_capacity`

However, the shared partial-rejection allocation helper receives only `reject_capacity`, not the Accept tray max capacity. The calculated `orig_cap` in `nq_action` is currently unused for Accept slot generation.

## 7. Exact Current Calculation

The defective calculation is in `Nickel_Inspection/services.py::build_nq_rejection_allocation`.

Current algorithm:

1. Clean original tray rows.
2. Set `remaining_reject_qty = rejected_qty`.
3. Iterate through original trays.
4. If the reject quantity fully consumes an original tray, move that tray to `delink_slots`.
5. If the reject quantity partially consumes an original tray, append the accepted remainder as:
   - `row['qty'] - remaining_reject_qty`
6. Append all later original trays as Accept trays unchanged.
7. Call `_mark_top_by_smallest_qty(accept_auto_trays)`.
8. Build `accept_slots` from those leftover accepted rows.

This calculates Accept slots from original tray leftovers, not from `accepted_qty / accept_capacity`.

## 8. Exact Root Cause

The root cause is missing max-capacity redistribution for Accept slots in the shared backend allocation function.

`build_nq_rejection_allocation()` subtracts the rejected quantity from the original tray stream. If the rejected quantity partially consumes a full original tray, that tray becomes a second partial accepted remainder.

For a one-piece rejection:

- Normal full tray `20 - 1 = 19`
- Jumbo full tray `12 - 1 = 11`

If the original lot also has an existing Top tray quantity like `8` or `4`, `_mark_top_by_smallest_qty()` marks `8` or `4` as Top, while `19` or `11` remains as a non-Top Accept tray.

That is why the UI shows:

- `8, 19, 20, 20...`
- `4, 11, 12, 12...`

The frontend is not the authoritative source of the bug. It renders and preserves the backend response. The backend submit validation also accepts the bad allocation because it recomputes the same defective expected rows.

## 9. Frontend vs Backend Responsibility

This is primarily a backend business-logic defect.

Frontend involvement:

- The templates render backend `accept_slots`.
- They preserve slot quantities in `data-qty`.
- They submit and draft-save those values.

Backend involvement:

- The shared allocation helper creates the invalid two-partial distribution.
- The submit path validates against that same invalid distribution.
- The final persisted records can therefore contain the bad Accept tray split.

The fix should be backend-first at the shared allocation/validation point. Frontend should remain a renderer of backend allocation data.

## 10. Save, Restore, and Submission Impact

Submission impact:

- Bad Accept quantities can be persisted in `NickelQcTrayId`.
- Bad Accept quantities can be persisted in `NickelQC_Submission.accept_trays_data`.
- Bad Accept quantities can be persisted in `NickelQC_PartialAcceptLot.trays_snapshot`.
- Bad Accept quantities can be persisted in `NickelWiping_PartialAcceptRecord.accept_trays`.
- The child accepted `JigUnloadAfterTable` row gets the correct total `accepted_qty`, but tray-level quantities can be incorrectly split.

Draft impact:

- `_nq_build_draft_snapshot()` preserves `accept_slots` and `accept_auto_trays`.
- Both templates save those arrays in draft payload.
- A draft can preserve and later restore the bad generated split unless allocation is rebuilt with corrected logic when the rejected quantity is recalculated/resumed.

## 11. Shared or Separate Defective Logic

Zone 1 and Zone 2 share the defective backend logic.

- Zone 1 uses `Nickel_Inspection.views.nq_action`.
- Zone 2 imports and routes to the same `Nickel_Inspection.views.nq_action`.
- Both paths call `Nickel_Inspection.services.build_nq_rejection_allocation`.

The frontend files are separate but structurally duplicate the same render/submit behavior.

The minimal fix can be applied safely through the shared backend service and shared backend submit validation. Frontend changes should only be necessary if the response shape changes; ideally it should not.

## 12. Minimal Recommended Fix

Modify the shared allocation contract so Accept slots are generated from final accepted quantity and Accept max capacity.

Recommended backend changes:

1. Extend `build_nq_rejection_allocation(...)` to receive `accept_capacity` or `accepted_qty`.
2. In `nq_action` `ALLOCATE`, pass the already-resolved `_nq_tray_capacity(...)` value into the allocation helper.
3. In `_nq_do_submit_reject`, pass the same Accept capacity into the allocation helper.
4. Generate Accept slot quantities using:
   - `accepted_qty = sum(original_tray_qty) - rejected_qty` or the view-calculated `accepted_qty`
   - `remainder = accepted_qty % accept_capacity`
   - if `remainder > 0`, first slot is Top with `remainder`
   - all remaining slots are `accept_capacity`
   - if exactly divisible, all slots are `accept_capacity`; decide whether first slot or lowest tray ID remains marked Top, but no partial slot exists.
5. Ensure `normalize_accept_trays()` validates against the corrected expected quantities.
6. Preserve existing original-tray coverage and delink/reuse validation.

The fix should not change:

- reject tray prefix validation
- reject tray capacity validation
- delink/reuse behavior
- Day Planning
- full accept flow
- existing tray-ownership checks
- frontend-only business logic

Potential implementation detail:

- Keep the existing response fields `accept_slots` and `accept_auto_trays` so both templates continue to render without structural changes.
- Rebuild the quantities in those fields so only one Top partial can exist.
- Continue using original tray IDs for required coverage, but do not let original leftover quantities create non-Top partial Accept slots.

## 13. Files Requiring Modification for the Fix

Likely required:

- `Nickel_Inspection/services.py`
  - update `build_nq_rejection_allocation`
  - update/strengthen `normalize_accept_trays` if needed to enforce corrected slot quantities
- `Nickel_Inspection/views.py`
  - pass Accept capacity into `build_nq_rejection_allocation` in `ALLOCATE`
  - pass Accept capacity into `build_nq_rejection_allocation` in `_nq_do_submit_reject`

Likely not required if response shape is preserved:

- `static/templates/Nickel_Inspection/Nickel_PickTable.html`
- `static/templates/Nickel_Inspection - Zone_two/Nickel_PickTable_zone_two.html`
- `nickel_inspection_zone_two/views.py`

Do not modify:

- Day Planning
- model schema
- unrelated tray modules
- new test/management/debug files

## 14. Regression Risks

Risks to validate carefully:

- Existing delinked/reused tray behavior: reject-reused original trays must still remove the required delink input.
- Original tray coverage validation: Accept, Delink, and reused Reject trays must still cover the original lot tray IDs exactly.
- Exact-divisible accepted quantities: no partial Top slot should be generated, and Top selection must remain deterministic.
- Below-capacity accepted quantities: one Top slot only.
- Draft resume: old draft payloads may carry previously generated bad `accept_slots`; the corrected backend should revalidate/rebuild when allocation is refreshed.
- Zone 2 routing: because Zone 2 uses the shared `nq_action`, a shared fix affects both zones at once.
- Historical completed records: existing persisted bad records should not be rewritten by this fix unless explicitly requested.

Security/performance impact:

- The fix should stay in backend service logic and continue using ORM-backed existing models.
- No new raw SQL is needed.
- No additional broad queries are required if capacity is passed from the existing view context.

## 15. Verification Scenarios

Validate both Zone 1 and Zone 2.

Normal capacity `20`, Accept Qty `287`:

- Expected capacity-shaped slots: `7, 20, 20, ...`
- Only the first/Top slot is partial.
- No `19` slot appears.

Jumbo capacity `12`, Accept Qty `147`:

- Expected capacity-shaped slots: `3, 12, 12, ...`
- Only the first/Top slot is partial.
- No `11` slot appears.

Exact divisible:

- Normal `280` with capacity `20` should render only `20` quantity slots.
- Jumbo `144` with capacity `12` should render only `12` quantity slots.
- No partial Top quantity is generated.

Below one capacity:

- Normal `8` with capacity `20` should render one Top slot `8`.
- Jumbo `4` with capacity `12` should render one Top slot `4`.

Capacity plus partial:

- Normal `27` should render `7, 20`.
- Jumbo `15` should render `3, 12`.

Partial rejection Qty `1`:

- A one-piece rejection must not create a second partial Accept tray.
- Normal should not show `..., 19, ...` as a non-Top Accept tray.
- Jumbo should not show `..., 11, ...` as a non-Top Accept tray.

Submit validation:

- Submitting any non-Top Accept tray with quantity below max capacity should be rejected.
- Submitted Accept tray total must still equal `accepted_qty`.
- Submitted Reject tray total must still equal `rejected_qty`.
- Delink/reuse coverage must still pass only when original trays are fully accounted for.

Draft validation:

- Save draft after corrected allocation.
- Resume draft and confirm Accept `x Qty` remains corrected.
- Recalculate rejected quantity and confirm backend regenerates corrected Accept slots.

## 16. Conclusion

The defect is caused by shared backend allocation logic in `Nickel_Inspection/services.py::build_nq_rejection_allocation`.

It is not a Zone 1-only or Zone 2-only UI issue. Both zones consume the same shared backend action and both templates render whatever `accept_slots` the backend returns.

The minimal safe fix is to update the shared backend allocation and submit-validation path so Accept tray quantities are generated from `accepted_qty` and Accept max capacity, while preserving the existing original-tray coverage and delink/reuse rules.
