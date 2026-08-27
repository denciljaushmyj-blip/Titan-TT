# Brass QC & Brass Audit Full Accept View Modal Assessment

## 1. Issue Summary

The Completed Table eye modal is outcome-agnostic when rendering reject panels. For a `FULL_ACCEPT` / Accepted lot, the backend can still return a `reject_lots` array when historical delink/reject-style structures exist, even if the actual submitted outcome is:

```text
accepted_qty = total_lot_qty
rejected_qty = 0
submission_type = FULL_ACCEPT
```

The frontend then renders every object in `reject_lots` as a reject panel. Because `data.is_full_reject` is false for a Full Accept lot, the panel label becomes:

```text
PARTIAL REJECT
```

So the incorrect Partial Reject section is caused by reject-section rendering being driven by the existence of `reject_lots`, not strictly by the actual lot outcome.

## 2. Brass QC Root Cause

### Backend

Relevant flow:

- `Brass_QC/urls.py`
  - `api/submitted-detail/`
- `Brass_QC/views.py:639`
  - `BrassQCSubmittedDetailAPI`
  - delegates to `get_brass_qc_submitted_detail(lot_id)`
- `Brass_QC/services/selectors.py:344`
  - `get_brass_qc_submitted_detail`

Relevant backend fields:

- `Brass_QC_Submission.submission_type`
- `Brass_QC_Submission.total_lot_qty`
- `Brass_QC_Submission.accepted_qty`
- `Brass_QC_Submission.rejected_qty`
- `Brass_QC_Submission.full_accept_data`
- `Brass_QC_Submission.full_reject_data`
- `Brass_QC_Submission.partial_accept_data`
- `Brass_QC_Submission.partial_reject_data`
- `Brass_QC_Submission.snapshot_data["accepted"]`
- `Brass_QC_Submission.snapshot_data["rejected"]`
- `Brass_QC_Submission.snapshot_data["delinked"]`
- `BrassQC_PartialAcceptLot`
- `BrassQC_PartialRejectLot`

Problem condition:

```python
elif submission.rejected_qty > 0 or delinked_tray_ids:
    reject_snapshot = submission.full_reject_data or submission.partial_reject_data or {}
    trays = _append_delink_rows(..., delinked_tray_ids)
    reject_lots.append(...)
```

This condition can create `reject_lots` for a Full Accept record when `delinked_tray_ids` exists, even when `submission.rejected_qty == 0`.

The returned classification flags are correct:

```python
"is_full_accept": submission.submission_type == "FULL_ACCEPT"
"is_full_reject": submission.submission_type == "FULL_REJECT"
"is_partial_reject": submission.submission_type == "PARTIAL" and bool(reject_lots)
```

But the response can still contain `reject_lots` for `FULL_ACCEPT`, which conflicts with the intended UI outcome.

### Frontend

Relevant template:

- `static/templates/Brass_Qc/Brass_Completed.html`

Relevant functions:

- eye icon: around `static/templates/Brass_Qc/Brass_Completed.html:1266`
- click handler: around `static/templates/Brass_Qc/Brass_Completed.html:4103`
- API fetch: around `static/templates/Brass_Qc/Brass_Completed.html:4320`
- `buildCompletedHistoryHTML`: around `static/templates/Brass_Qc/Brass_Completed.html:4283`
- `renderRejectPanel`: around `static/templates/Brass_Qc/Brass_Completed.html:4254`

Problem rendering condition:

```javascript
(data.reject_lots || []).forEach(function(rejectLot) {
    html += renderRejectPanel(rejectLot, !!data.is_full_reject);
});
```

This renders reject panels whenever `reject_lots` exists. It does not require:

```text
data.is_partial_reject === true
or
data.is_full_reject === true
or
rejectLot.rejected_qty > 0
```

For a Full Accept row, `data.is_full_reject` is false, so `renderRejectPanel` labels the section as `PARTIAL REJECT`.

## 3. Brass Audit Root Cause

### Backend

Relevant flow:

- `BrassAudit/urls.py`
  - `api/submitted-detail/`
- `BrassAudit/views.py:869`
  - `BrassAuditSubmittedDetailAPI`
  - delegates to `get_brass_audit_submitted_detail(lot_id)`
- `BrassAudit/selectors.py:96`
  - `get_brass_audit_submitted_detail`

Relevant backend fields:

- `Brass_Audit_Submission.submission_type`
- `Brass_Audit_Submission.total_lot_qty`
- `Brass_Audit_Submission.accepted_qty`
- `Brass_Audit_Submission.rejected_qty`
- `Brass_Audit_Submission.full_accept_data`
- `Brass_Audit_Submission.full_reject_data`
- `Brass_Audit_Submission.partial_accept_data`
- `Brass_Audit_Submission.partial_reject_data`
- `Brass_Audit_Submission.snapshot_data["accepted"]`
- `Brass_Audit_Submission.snapshot_data["rejected"]`
- `Brass_Audit_Submission.snapshot_data["delinked"]`
- `BrassAudit_PartialAcceptLot`
- `BrassAudit_PartialRejectLot`
- `BrassAuditTrayId.delink_tray`
- `BrassTrayId.delink_tray`

Problem condition:

```python
elif submission.rejected_qty > 0 or delinked_tray_ids:
    reject_snapshot = submission.full_reject_data or submission.partial_reject_data or {}
    trays = _append_delink_rows(..., delinked_tray_ids)
    reject_lots.append(...)
```

Brass Audit has the same issue as Brass QC, with one extra source of false reject-panel data: when `snapshot_data["delinked"]` is empty, it also falls back to physical mirror rows:

```python
BrassAuditTrayId.objects.filter(lot_id=submission.lot_id, delink_tray=True)
BrassTrayId.objects.filter(lot_id=submission.lot_id, delink_tray=True)
```

That means a Full Accept lot can receive `reject_lots` purely because delinked tray history exists, even though the actual outcome is not reject/partial.

### Frontend

Relevant template:

- `static/templates/BrassAudit/BrassAudit_Completed.html`

Relevant functions:

- eye icon: around `static/templates/BrassAudit/BrassAudit_Completed.html:820`
- click handler: around `static/templates/BrassAudit/BrassAudit_Completed.html:2239`
- API fetch: around `static/templates/BrassAudit/BrassAudit_Completed.html:2428`
- `buildBrassAuditCompletedHistoryHTML`: around `static/templates/BrassAudit/BrassAudit_Completed.html:2391`
- `renderBrassAuditRejectPanel`: around `static/templates/BrassAudit/BrassAudit_Completed.html:2362`

Problem rendering condition:

```javascript
(data.reject_lots || []).forEach(function(rejectLot) {
    html += renderBrassAuditRejectPanel(rejectLot, !!data.is_full_reject);
});
```

Like Brass QC, this renders a reject card based only on `reject_lots` existence. It does not gate rendering by actual business outcome.

## 4. Backend vs Frontend Responsibility

### Brass QC

Classification:

```text
Both backend and frontend
```

Reason:

- Backend can return `reject_lots` for a Full Accept outcome due to `delinked_tray_ids`.
- Frontend renders `reject_lots` without checking `is_full_reject`, `is_partial_reject`, `submission_type`, or positive `rejected_qty`.

### Brass Audit

Classification:

```text
Both backend and frontend
```

Reason:

- Backend can return `reject_lots` for a Full Accept outcome due to `delinked_tray_ids` or mirror `delink_tray=True` rows.
- Frontend renders `reject_lots` without checking the actual outcome.

## 5. Current Rendering Flow

### Brass QC

```text
Eye icon clicked in Brass QC Completed table
-> static/templates/Brass_Qc/Brass_Completed.html click handler runs
-> fetch('/brass_qc/api/submitted-detail/?lot_id=...')
-> BrassQCSubmittedDetailAPI calls get_brass_qc_submitted_detail()
-> backend builds accept_lots from full_accept_data / partial_accept_data
-> backend may build reject_lots if rejected_qty > 0 OR delinked_tray_ids exists
-> frontend buildCompletedHistoryHTML renders all accept_lots
-> frontend buildCompletedHistoryHTML renders all reject_lots
-> reject panel receives isFullReject=false
-> panel label displays PARTIAL REJECT
```

### Brass Audit

```text
Eye icon clicked in Brass Audit Completed table
-> static/templates/BrassAudit/BrassAudit_Completed.html click handler runs
-> fetch('/brass_audit/api/submitted-detail/?lot_id=...')
-> BrassAuditSubmittedDetailAPI calls get_brass_audit_submitted_detail()
-> backend builds accept_lots from full_accept_data / partial_accept_data
-> backend may build reject_lots if rejected_qty > 0 OR delinked_tray_ids exists
-> backend may also collect delinked_tray_ids from BrassAuditTrayId / BrassTrayId mirrors
-> frontend buildBrassAuditCompletedHistoryHTML renders all accept_lots
-> frontend buildBrassAuditCompletedHistoryHTML renders all reject_lots
-> reject panel receives isFullReject=false
-> panel label displays PARTIAL REJECT
```

## 6. Expected Rendering Flow

The modal should render sections from the actual submitted outcome:

```text
FULL_ACCEPT
-> render accepted/full-accept panel only
-> ignore reject_lots/delink history for this modal outcome

PARTIAL
-> render accepted/partial-accept panel when accepted_qty > 0
-> render rejected/partial-reject panel when rejected_qty > 0 or valid partial delink/reject details are part of the partial outcome

FULL_REJECT
-> render rejected/full-reject panel only
```

A record with:

```text
accepted_qty = total_lot_qty
rejected_qty = 0
submission_type = FULL_ACCEPT
```

must not display a reject section, even if historical tray/delink rows exist.

## 7. Recommended Minimal Fix

### Backend

In both submitted-detail builders, gate reject-lot construction by the submitted outcome:

- `Brass_QC/services/selectors.py::get_brass_qc_submitted_detail`
- `BrassAudit/selectors.py::get_brass_audit_submitted_detail`

Recommended condition:

```text
Only build reject_lots when:
- submission.submission_type == "FULL_REJECT" and rejected_qty > 0
or
- submission.submission_type == "PARTIAL" and the partial reject section is valid
```

For `FULL_ACCEPT`, return:

```text
reject_lots = []
is_partial_reject = False
is_full_reject = False
```

Do not remove accepted/full-accept data.

### Frontend

In both completed templates, add a defensive render gate:

- `static/templates/Brass_Qc/Brass_Completed.html::buildCompletedHistoryHTML`
- `static/templates/BrassAudit/BrassAudit_Completed.html::buildBrassAuditCompletedHistoryHTML`

Recommended condition:

```text
Render reject_lots only when:
data.is_full_reject === true || data.is_partial_reject === true
```

This prevents future backend payload drift from reintroducing the Full Accept reject-card bug.

## 8. Regression Risks

The eventual fix must preserve:

- Full Accept accepted tray display
- Partial Accept accepted section
- Partial Reject rejected tray section
- delinked tray display for genuine partial outcomes
- rejection reason chips/details
- Full Reject rejected tray display
- completed-history snapshots instead of live tray occupancy
- old completed rows that rely on snapshot fallback

Main risk:

- Over-filtering `reject_lots` could hide valid delink/reject details for real `PARTIAL` or `FULL_REJECT` submissions. The condition should be outcome-based, not simply "hide all delinks".

## 9. Files That Would Need Modification

Files requiring modification:

1. `Brass_QC/services/selectors.py`
2. `BrassAudit/selectors.py`
3. `static/templates/Brass_Qc/Brass_Completed.html`
4. `static/templates/BrassAudit/BrassAudit_Completed.html`

Files inspected but not requiring modification for the minimal fix:

1. `Brass_QC/views.py`
2. `BrassAudit/views.py`
3. `Brass_QC/urls.py`
4. `BrassAudit/urls.py`
5. `Brass_QC/services/submission_service.py`

## 10. Final Conclusion

The screenshot issue is caused by both:

```text
incorrect backend modal payload composition
and
incorrect frontend rendering condition
```

The backend should not return a reject-lot structure for a `FULL_ACCEPT` outcome just because historical delink/tray records exist. The frontend should also not render reject panels based only on `reject_lots` truthiness.

The minimal safe fix is to gate rejection-section creation and rendering by actual submitted outcome:

```text
FULL_ACCEPT -> accepted section only
PARTIAL -> accepted + partial reject/delink sections as applicable
FULL_REJECT -> full reject section
```
