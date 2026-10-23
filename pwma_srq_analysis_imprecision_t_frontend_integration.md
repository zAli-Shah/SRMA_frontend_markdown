# Imprecision t (SRQ and SOF Outcome level) — Frontend Integration Guide

Backend support for setting `is_null_imprecision_t` on a **clinical question (SRQ)** and on a **single SOF outcome**, in addition to the existing **project**-level toggle.

All URLs below are relative to `/backend/api/v1`. All requests need the usual `Authorization: Bearer <token>` header.

---

## 1. What the feature does

- `is_null_imprecision_t` picks the threshold used for the **imprecision** domain of PWMA certainty of evidence:
  - `false` (default): threshold `0.05`.
  - `true`: threshold at the null effect: `0` for risk difference (RD), `1` for ratio measures (RR, OR, …).
- The flag now exists at three levels: **project → SRQ → SOF outcome**.
- Every level stores its own `true` / `false`. The value used for an outcome's calculation is **the outcome's own flag**.
- Changing a higher level **pushes its value down**:

| Toggle changed | Its own flag | Flags pushed down to | Outcomes recalculated |
|---|---|---|---|
| Project | project | every SRQ in the project, plus every recalculated outcome | every selected PWMA outcome in the project |
| SRQ | that SRQ | every recalculated outcome of that SRQ | every selected PWMA outcome of that SRQ |
| SOF outcome | that outcome | — | that outcome only |

- Turning the project or SRQ toggle **overwrites** any outcome-level values below it. Changing an outcome's toggle doesn't affect its SRQ or project.
- Only **selected PWMA** outcomes that have a certainty-of-evidence record are recalculated and get the new flag. Unselected outcomes and NMA outcomes keep their current flag.
- New SRQs start with the project's value. New SOF outcomes start with their SRQ's value.

---

## 2. Where the toggles go

### 2.1 PWMA SOF list view — project toggle + SRQ toggle

Show the **SRQ toggle at the same level as the existing project toggle**, e.g. side by side above the SOF table list.

```
┌───────────────────────────────────────────────────────────────┐
│ Summary of Findings (PWMA)                                    │
│                                                               │
│ Project imprecision t: [● on ]   SRQ imprecision t: [○ off]   │
│                                                               │
│ ▸ Folder A                                                    │
│     SOF table 1 …                                             │
└───────────────────────────────────────────────────────────────┘
```

- **Project toggle:** reads `is_null_imprecision_t` from the list response (unchanged).
- **SRQ toggle:** reads the new `srq_is_null_imprecision_t` from the same list response.
- Both list endpoints return both flags (§3.1), so no extra request is needed.
- Show these toggles in the **PWMA** list only. Imprecision t isn't used for NMA.

### 2.2 SOF table detail view — one toggle per outcome

Show a toggle with **each SOF outcome row**, e.g. next to the imprecision cell or in the row's actions.

```
┌───────────────────────────────────────────────────────────────────────┐
│ Outcome            │ Relative effect │ Imprecision │ Imprecision t    │
├────────────────────┼─────────────────┼─────────────┼──────────────────┤
│ All-cause mortality│ RR 0.82 (…)     │ Serious     │ [● on ]          │
│ Hospitalisation    │ RR 0.91 (…)     │ Not serious │ [○ off]          │
└───────────────────────────────────────────────────────────────────────┘
```

- Each item in `outcomes[]` of the SOF detail response now has `is_null_imprecision_t` (§3.2).

### 2.3 Who can toggle

- Only users with the **PW SOF management** privilege (`is_pw_sof_management_allowed`) can change any of the three toggles. Others get `403`.
- For users without it, show the toggles read-only or hide them, as the project toggle is handled today.

---

## 3. API changes

### 3.1 SOF list endpoints — new `srq_is_null_imprecision_t`

Both PWMA SOF list endpoints now return **`srq_is_null_imprecision_t`** next to the existing `is_null_imprecision_t`:

- `GET /summary_of_finding/all/{project_id}/{clinical_question_id}?is_pwma=true`
- `GET /summary_of_finding/all_with_folders/{project_id}/{clinical_question_id}?is_pwma=true`

```json
{
  "folders": [ ... ],
  "total_sof_table_count": 12,
  "is_null_imprecision_t": false,
  "srq_is_null_imprecision_t": true
}
```

| Key | Meaning |
|---|---|
| `is_null_imprecision_t` | project flag (unchanged) |
| `srq_is_null_imprecision_t` | flag of the clinical question in the URL. Always `true`/`false`. `false` if the clinical question isn't found. |

The public website endpoint (`GET /living_website/all_with_folders/{project_id}/{clinical_question_id}`) returns the same response type. Older published snapshots don't have `srq_is_null_imprecision_t` and return `false` for it.

### 3.2 SOF table detail — per-outcome `is_null_imprecision_t`

`GET /summary_of_finding/{sof_table_id}`

Each item in `outcomes[]` has a new boolean **`is_null_imprecision_t`**. All other keys are unchanged.

```json
{
  "sof_id": 310,
  "name": "Primary outcomes",
  "cq_id": 75,
  "project_id": 45,
  "outcomes": [
    {
      "sof_outcome_id": 19634,
      "outcome_name": "All-cause mortality",
      "is_selected": true,
      "is_null_imprecision_t": true,
      "imprecision": { "...": "..." },
      "overall_imprecision": "Serious",
      "overall_imprecision_reason": "...",
      "overall": "Moderate",
      "...": "..."
    }
  ]
}
```


### 3.4 Update toggle — `PUT /certainty_of_evidence/update_imprecision_t/{project_id}`

Same endpoint as the project toggle, with two new **optional** body fields. Which ids you send decides the level:

| Body | Level |
|---|---|
| `{ "is_null_imprecision_t": true }` | project (unchanged) |
| `{ "is_null_imprecision_t": true, "clinical_question_id": 75 }` | SRQ |
| `{ "is_null_imprecision_t": false, "sof_outcome_id": 19634 }` | SOF outcome |

- `is_null_imprecision_t` is **required** and must be `true` or `false`. `null` or a missing value returns `422`.
- If both `sof_outcome_id` and `clinical_question_id` are sent, the **SOF outcome** level is used. Send only one.

**Response (all levels):**

```json
{
  "message": "Imprecision t updated successfully",
  "recalculated_sof_outcome_ids": [19634, 19635, 19636],
  "failures": [
    { "sof_outcome_id": 19640, "sof_id": 311, "detail": "Certainty of evidence record not found" }
  ]
}
```

- `recalculated_sof_outcome_ids`: outcomes whose imprecision and overall certainty were recalculated. These now have the new flag.
- `failures`: outcomes that couldn't be recalculated. They **keep their previous flag and values**. `detail` is a plain string you can show to the user.
- The request succeeds (`200`) even when `failures` isn't empty.

**Errors:**

| Status | `detail` | When |
|---|---|---|
| 400 | `Imprecision t is already set to the same value` | **SOF outcome level only:** the outcome already has that value |
| 403 | `You don't have sufficient permissions to perform this action` / `…privileges…` | not a member, or no PW SOF management privilege |
| 404 | `Project not found` | |
| 404 | `Clinical question not found` | clinical question doesn't exist, is deleted, or belongs to another project |
| 404 | `Sof outcome not found` | outcome doesn't exist, is deleted, isn't selected, is NMA, or belongs to another project |
| 422 | validation error | `is_null_imprecision_t` missing or `null` |

> Changed behaviour: the **project** level (and the SRQ level) no longer returns `400` when the value is unchanged. Sending the same value pushes it down again and recalculates. This is how you reset outcomes that were changed individually.

---

## 4. Frontend flow

### 4.1 Project / SRQ toggle (PWMA SOF list)

1. Disable the toggle and show a loader. This call **recalculates synchronously**. It takes about 1 s for 10–15 outcomes and 7 s for about 90. Large projects (400+ outcomes) can take **30 s or more**, so use a generous request timeout and don't let the user click again.
2. `PUT /certainty_of_evidence/update_imprecision_t/{project_id}` with `{ is_null_imprecision_t }`, plus `clinical_question_id` for the SRQ toggle.
3. On `200`:
   - Update the toggle from the value you sent.
   - **Project toggle:** set the SRQ toggle to the same value, because the project pushes its value to every SRQ.
   - Re-fetch any open SOF table detail, since outcome imprecision, overall certainty and outcome flags changed.
   - If `failures` isn't empty, show a non-blocking warning, e.g. "Imprecision could not be recalculated for N outcome(s)", with each `detail`.
4. On error: revert the toggle and show `detail`.

### 4.2 SOF outcome toggle (SOF table detail)

1. Disable that row's toggle and show a loader for that row.
2. `PUT /certainty_of_evidence/update_imprecision_t/{project_id}` with `{ is_null_imprecision_t, sof_outcome_id }`.
3. On `200`: re-fetch the SOF table detail (`GET /summary_of_finding/{sof_table_id}`), or at least that outcome's row, to show the new imprecision and overall certainty.
4. On `400 already set to the same value`: your local state is stale. Re-fetch the detail and sync the toggle.
5. On other errors: revert the toggle and show `detail`.

### 4.3 Things to handle

- **Toggle state comes from the server.** After a project or SRQ change, outcome toggles in the detail view can all change. Always re-fetch instead of guessing.
- **Unselected outcomes** keep their own flag. If one is selected again later, it uses that stored flag. Show whatever `is_null_imprecision_t` the detail returns.
- **Overall certainty follows the new imprecision:** whenever a toggle changes, `overall` is recalculated from the **newly computed** imprecision, even if a reviewer had manually set the imprecision rating (`is_overall_imprecision_updated_by_reviewer: true`). This is intended. The other domains (risk of bias, inconsistency, publication bias, indirectness) keep the reviewer's value where one is set. Expect `overall` to change after a toggle, and always re-fetch it.
- **NMA:** no toggles. NMA endpoints and responses are unchanged.

---

## 5. Quick test checklist (QA / frontend)

- [ ] PWMA SOF list shows project and SRQ toggles side by side, with values from `is_null_imprecision_t` / `srq_is_null_imprecision_t`.
- [ ] Switching SRQ changes the SRQ toggle only; the project toggle stays the same.
- [ ] Switching the project also switches the SRQ toggle to the same value.
- [ ] Changing to another SRQ in the list loads that SRQ's own value.
- [ ] SOF detail shows a toggle per outcome matching `is_null_imprecision_t`.
- [ ] Switching one outcome updates only that row's imprecision / overall; other rows stay the same.
- [ ] After switching the SRQ, every outcome toggle in that SRQ's tables shows the SRQ value, including ones changed individually before.
- [ ] Toggling an outcome to the value it already has (stale UI) re-syncs instead of showing a broken state.
- [ ] `failures` in the response shows a warning, not an error, and those outcomes keep their old values.
- [ ] Users without PW SOF management can't change the toggles.
- [ ] Toggles are not shown in NMA views.
