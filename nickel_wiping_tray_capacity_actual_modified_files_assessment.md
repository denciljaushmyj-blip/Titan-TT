# Actual Modified Files Assessment

## 1. Git Working Tree Summary

- Staged files: none
- Modified tracked files: 21
- Untracked files at inspection time: 3
- Actual source files modified for the latest Nickel Wiping tray-capacity fix: 2
- Latest fix stayed within intended source scope: yes

## 2. Files Reported by Git

`git diff --name-only` reported 21 tracked files.

`git diff --cached --name-only` reported no staged files.

Tracked files reported by Git:

```text
BrassAudit/selectors.py
BrassAudit/views.py
Brass_QC/services/selectors.py
Brass_QC/views.py
IQF/views.py
Jig_Unloading/tray_utils.py
Nickel_Audit/views.py
Nickel_Inspection/services.py
Nickel_Inspection/views.py
adminportal/global_scan.py
adminportal/module_registry.py
adminportal/urls.py
static/templates/BrassAudit/BrassAudit_Completed.html
static/templates/Brass_Qc/Brass_Completed.html
static/templates/IQF/Iqf_PickTable.html
static/templates/Nickel_Audit - Zone_two/NickelAudit_Completed_zone_two.html
static/templates/Nickel_Audit - Zone_two/NickelAudit_PickTable_zone_two.html
static/templates/Nickel_Audit/NickelAudit_Completed.html
static/templates/Nickel_Audit/NickelAudit_PickTable.html
static/templates/Nickel_Inspection - Zone_two/NI_Completed_zone_two.html
static/templates/Nickel_Inspection/NI_Completed.html
```

## 3. Actual Files Belonging to Latest Fix

| File | Functions/Hunks Changed | Why It Belongs |
|------|--------------------------|----------------|
| `Nickel_Inspection/services.py` | `_build_accept_slot_quantities`, `_sort_accept_candidates_for_capacity`, `_apply_accept_capacity_shape`, `_validate_accept_capacity_shape`, `build_nq_rejection_allocation`, `normalize_accept_trays` | Implements accepted-qty/capacity-based Accept tray allocation and submit validation so only Top can be partial. |
| `Nickel_Inspection/views.py` | `nq_action`, `_nq_do_submit_reject` | Passes `accepted_qty` and `_nq_tray_capacity(...)` into the shared allocation and validation path. |

## 4. Mixed Files

| File | Relevant Changes | Unrelated Changes |
|------|------------------|-------------------|
| `Nickel_Inspection/services.py` | New Accept slot capacity shaping and validation logic. | Existing diff also includes `plating_stk_no` / ModelMaster resolution for rejection tray type and prefix, which is not part of this latest Accept tray capacity fix. |
| `Nickel_Inspection/views.py` | Passes `accept_capacity=orig_cap` and `accepted_qty=accepted_qty` in allocation/submit; passes same values to `normalize_accept_trays`. | Existing diff also passes `plating_stk_no` into rejection tray series/allocation checks, unrelated to the latest Accept tray capacity fix. |

## 5. Unrelated Existing Changes

| File | Why It Is Not Part of This Fix |
|------|--------------------------------|
| `BrassAudit/selectors.py` | Brass Audit accepted/rejected modal data filtering. |
| `BrassAudit/views.py` | Brass Audit tray count/view logic. |
| `Brass_QC/services/selectors.py` | Brass QC submitted detail filtering. |
| `Brass_QC/views.py` | Brass QC tray count/view logic. |
| `IQF/views.py` | IQF Input Screening rejected tray validation. |
| `Jig_Unloading/tray_utils.py` | Delinked tray reuse conflict handling. |
| `Nickel_Audit/views.py` | Nickel Audit hold/release behavior. |
| `adminportal/global_scan.py` | Global scan permission/routing logic. |
| `adminportal/module_registry.py` | Module registry column ordering. |
| `adminportal/urls.py` | Network ping API view routing. |
| `static/templates/BrassAudit/BrassAudit_Completed.html` | Brass Audit completed modal rendering. |
| `static/templates/Brass_Qc/Brass_Completed.html` | Brass QC completed modal rendering. |
| `static/templates/IQF/Iqf_PickTable.html` | IQF reject tray frontend validation. |
| `static/templates/Nickel_Audit - Zone_two/NickelAudit_Completed_zone_two.html` | Nickel Audit completed table column/status layout. |
| `static/templates/Nickel_Audit - Zone_two/NickelAudit_PickTable_zone_two.html` | Nickel Audit S.No, hold/release info icon, tooltip layout. |
| `static/templates/Nickel_Audit/NickelAudit_Completed.html` | Nickel Audit completed table layout/status changes. |
| `static/templates/Nickel_Audit/NickelAudit_PickTable.html` | Nickel Audit S.No, hold/release info icon, tooltip layout. |
| `static/templates/Nickel_Inspection - Zone_two/NI_Completed_zone_two.html` | Newline-only/template EOF change. |
| `static/templates/Nickel_Inspection/NI_Completed.html` | Nickel Inspection completed status indicator change. |

## 6. Untracked / Generated Files

| File | Classification |
|------|----------------|
| `brass_qc_brass_audit_full_accept_view_modal_assessment.md` | `ASSESSMENT_OR_DOCUMENTATION`, unrelated to latest Nickel Wiping fix. |
| `nickel_wiping_zone1_zone2_accept_tray_top_partial_capacity_assessment.md` | `ASSESSMENT_OR_DOCUMENTATION`, related assessment document. |
| `nickel_wiping_zone1_zone2_accept_tray_top_partial_capacity_fix_result.md` | `ASSESSMENT_OR_DOCUMENTATION`, related result document. |

No untracked `test_*.py`, `*_test.py`, management command, debug script, temp script, migration, or backup file was found during inspection.

## 7. Unexpected Files Modified by Latest Fix

No unexpected source files were modified by the latest Nickel Wiping tray-capacity fix.

Files outside intended scope appear in Git, but their hunks are unrelated existing changes from prior work.

## 8. Final Actual Fix File List

Actual source files modified for this fix:

```text
Nickel_Inspection/services.py
Nickel_Inspection/views.py
```

## 9. Conclusion

Actual source files modified for this fix: 2

```text
Nickel_Inspection/services.py
Nickel_Inspection/views.py
```

Other files appearing in git diff: 19

```text
BrassAudit/selectors.py
BrassAudit/views.py
Brass_QC/services/selectors.py
Brass_QC/views.py
IQF/views.py
Jig_Unloading/tray_utils.py
Nickel_Audit/views.py
adminportal/global_scan.py
adminportal/module_registry.py
adminportal/urls.py
static/templates/BrassAudit/BrassAudit_Completed.html
static/templates/Brass_Qc/Brass_Completed.html
static/templates/IQF/Iqf_PickTable.html
static/templates/Nickel_Audit - Zone_two/NickelAudit_Completed_zone_two.html
static/templates/Nickel_Audit - Zone_two/NickelAudit_PickTable_zone_two.html
static/templates/Nickel_Audit/NickelAudit_Completed.html
static/templates/Nickel_Audit/NickelAudit_PickTable.html
static/templates/Nickel_Inspection - Zone_two/NI_Completed_zone_two.html
static/templates/Nickel_Inspection/NI_Completed.html
```

Unexpected source files modified by this fix: NO

New source/test/management files created by this fix: NO
