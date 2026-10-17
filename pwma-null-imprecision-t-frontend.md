# PWMA SOF list — null imprecision T (frontend spec)

## Goal

On the **PWMA Summary of Findings table list** (not NMA, not Evidence Map), let users toggle the project setting **`is_null_imprecision_t`**. That triggers a backend recalculation of **imprecision** and **overall COE** for **all selected PWMA SOF outcomes in the entire project** (all clinical questions). The operation can take a long time.

## Where to integrate

| Item | Location |
|------|----------|
| Screen | PWMA SOF list — `SOFCreateBase.tsx` (`isPwma === true`, `isEvidenceMapList === false`) |
| Pattern reference | Bulk “Recalculate COE” in the same file (`useRecalculateCoeBulk`, `selectionExtraActions`) — **but this feature is project-scoped**, not “selected SOF tables” |
| Preferred placement | List **toolbar** (same row as sort / filters / `SofManagementToolbar` or `FileManagerList` header), **visible without selecting SOF rows** |

Do **not** show on NMA SOF list or Evidence Map list.

## API

**Method:** `PUT`  
**URL:** `${Base_Url}/certainty_of_evidence/update_imprecision_t/{projectId}`  
**Auth:** Bearer token (same as other COE hooks)  
**Path:** `projectId` — use numeric project id from `SOFCreateBase` props (`parseInt(projectId, 10)`).

**Request body:**

```json
{ "is_null_imprecision_t": true }
```

| `is_null_imprecision_t` | Effect on `imp_t` (backend) |
|---------------------------|-----------------------------|
| `false` (default) | `0.05` for all summary measures |
| `true` + SM = **RD** | `0.0` |
| `true` + SM = OR/RR/etc. | `1.0` |

**Success response (200):**

```json
{
  "message": "Imprecision t updated successfully",
  "recalculated_sof_outcome_ids": [123, 456],
  "failures": [
    { "sof_outcome_id": 1, "sof_id": 2, "detail": "..." }
  ]
}
```

**Errors:**

| Status | When |
|--------|------|
| `400` | Same flag as current (`detail`: imprecision t already set to the same value) |
| `404` | Project not found |

## UX (required)

### 1. Control

- Use a **labeled toggle** or **two-state button** bound to `is_null_imprecision_t`.
- Label suggestion: **“Use null imprecision T”** with short helper text: *When on, RD uses T=0; other measures use T=1; when off, T=0.05.*
- Initial value: from project payload if the GET project API exposes `is_null_imprecision_t`; otherwise default `false` until first successful PUT.

### 2. Toaster on click (before / when request starts)

**Immediately** when the user confirms the toggle (on click, before or as the request fires), show an **info** toast (use `Notifier.info`):

> **Recalculating imprecision** — This may take a while. The imprecision domain (and overall certainty) will be updated for **all PWMA SOF outcomes across this project**.

Do not block the request on the toast; fire toast then call the API.

### 3. Loading

- Disable the control while the mutation is pending.
- Show inline loading (e.g. spinner on button) — request may run **minutes** on large projects.

### 4. Success / partial success

- **Success:** `Notifier.success` with backend `message` (or a short fixed string).
- **Partial failures:** if `failures.length > 0`, mirror `use_recalculate_coe_bulk.ts` — error toast summarizing count + first few `detail` strings.
- If `recalculated_sof_outcome_ids.length === 0` and no failures, still show success but mention nothing was recalculated (edge case).

### 5. Cache / refresh

- Mutation `meta`: `{ cacheEvents: 'coe.changed' }` (same as `use_recalculate_coe_bulk.ts`).
- Invalidate/refetch SOF list and any open COE/SoF table views for this project/CQ after success.

## Implementation checklist

1. Add `src/hooks/SOF/use_update_imprecision_t.ts` — `useMutation`, PUT handler, error mapping from `response.data.detail`.
2. Wire into **PWMA** `SOFCreateBase` toolbar; pass `projectId` and `token`.
3. On toggle change → **info toaster** → `mutate({ is_null_imprecision_t: newValue })`.
4. Handle `400` duplicate flag: show error toast, revert toggle UI to previous value.
5. Manual QA: RD test outcome should change imprecision when toggling; many OR outcomes may **not** change label (expected).

## Out of scope

- NMA SOF list, single-outcome recalc, bulk recalc by selected SOF ids (`recalculate_coe_bulk`).
- Changing `imp_t` per outcome (project flag only).

## Backend reference

Service: `update_imprecision_t_service` in `SRMA/app/services/certainty_of_evidence.py`  
DB: `projects.is_null_imprecision_t` (migration `91dea4c2103d_imprecision_flag_added`).
