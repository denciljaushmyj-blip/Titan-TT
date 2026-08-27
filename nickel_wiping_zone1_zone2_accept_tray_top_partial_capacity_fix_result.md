# Nickel Wiping Zone 1 and Zone 2 Accept Tray Top-Partial Capacity Fix Result

## 1. Root Cause Fixed

The shared Nickel Wiping partial-rejection allocation logic was deriving Accept tray quantities from original tray leftovers.

That could create a second partial Accept tray, for example:

- Normal: `8, 19, 20, 20...`
- Jumbo: `4, 11, 12, 12...`

The fix changes the shared backend allocation path so Accept tray quantities are generated from:

- final `accepted_qty`
- configured Accept tray max capacity

Only the Top Accept tray can now be partial.

## 2. Files Modified

- `Nickel_Inspection/services.py`
- `Nickel_Inspection/views.py`

No frontend files were modified.

## 3. Functions Modified

`Nickel_Inspection/services.py`

- `build_nq_rejection_allocation`
- `normalize_accept_trays`

Added small internal helpers:

- `_build_accept_slot_quantities`
- `_sort_accept_candidates_for_capacity`
- `_apply_accept_capacity_shape`
- `_validate_accept_capacity_shape`

`Nickel_Inspection/views.py`

- `nq_action`
- `_nq_do_submit_reject`

## 4. New Allocation Logic

The Accept tray allocation now follows:

```text
accepted_qty = final accepted quantity
capacity = configured Accept tray maximum capacity
remainder = accepted_qty % capacity

if remainder > 0:
    Top Tray = remainder
    all remaining trays = capacity
else:
    all trays = capacity
```

This prevents a non-Top Accept tray from having a partial quantity.

## 5. Zone 1 and Zone 2 Coverage

Both Nickel Wiping Zone 1 and Zone 2 use the shared backend action:

```text
Nickel_Inspection.views.nq_action
```

Zone 2 routes to the same shared backend flow, so no separate Zone 2 backend change was required.

## 6. Validation Results

Normal capacity `20`:

```text
Accept Qty 287
Result: 7, 20, 20, 20, ...
No 19 second partial tray.
```

Jumbo capacity `12`:

```text
Accept Qty 147
Result: 3, 12, 12, 12, ...
No 11 second partial tray.
```

Exact divisible:

```text
Normal 280 -> 20, 20, 20, ...
Jumbo 144 -> 12, 12, 12, ...
```

Below capacity:

```text
Normal 8 -> 8
Jumbo 4 -> 4
```

Capacity plus partial:

```text
Normal 27 -> 7, 20
Jumbo 15 -> 3, 12
```

Submit validation:

```text
Second partial Accept tray -> rejected
Non-Top tray over max capacity -> rejected
Valid capacity-shaped allocation -> accepted
```

## 7. Verification Commands Run

```text
python -m py_compile Nickel_Inspection\services.py Nickel_Inspection\views.py
git diff --check -- Nickel_Inspection\services.py Nickel_Inspection\views.py
```

Both checks passed.

## 8. Preserved Behavior

Preserved:

- Reject tray quantity validation
- Accept tray total validation
- Rejection tray prefix validation
- Delink tray reuse behavior
- Original tray coverage validation
- Tray occupancy validation
- Full accept logic
- Draft payload structure
- Frontend response contract

## 9. Unrelated Files

No unrelated source files were modified for this fix.

No migration files, test files, management commands, debug scripts, or backup files were created.
