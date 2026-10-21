# R-table Formula Columns — Frontend Integration Guide

Backend support for Excel-like formulas in R-table **merged columns** (e.g. percentage `Et × 100 / Nt`).

All URLs below are relative to `/backend/api/v1`. All requests need the usual `Authorization: Bearer <token>` header.

---

## 1. What the feature does

- A **merged column** can now have an optional `formula`, for example `=ROUND([Et]*100/[Nt], 1)`.
- When a column has a formula, the backend calculates each cell's value from that column's merged nodes, per paper and per treatment arm, and returns it in a **new key, `formula_result`**.
- The existing **`result` key is unchanged**: it is always the `data_display_format` text (e.g. `12 / 50`), whether or not the column has a formula.
- On every merged cell, `formula_result` is `null` when the column has no formula.
- Only merged columns can have a formula. Normal (non-merged) columns cannot.
- Values are calculated when the table is read, never stored. They update automatically when extraction values or per-R-table overrides change.
- Manuscript tables and guideline / public website snapshots get `formula_result` in the same data. Any screen that should show computed values there must read `formula_result` too (see §8).

---

## 2. Formula syntax (for the editor and help text)

| Item | Syntax | Example |
|---|---|---|
| Leading `=` | Optional | `=[Et]*100/[Nt]` or `[Et]*100/[Nt]` |
| Node reference | Node name in square brackets. Not case-sensitive; spaces inside the brackets are trimmed. | `[Et]`, `[Number of events]`, `[upper ci]` |
| Operators | `+ - * /` and `^` (power), with `( )` for grouping | `([Et]+[Ec])/([Nt]+[Nc])` |
| Numbers | `12`, `0.5`, `.5`, `1e3` | `[Et]*100` |
| Functions (any case) | `ROUND(x, digits)`, `ABS(x)`, `SQRT(x)`, `LN(x)`, `LOG10(x)`, `EXP(x)`, `MIN(a, b, …)`, `MAX(a, b, …)` | `ROUND(SQRT([x]), 2)` |

**Rules to show users:**
- A formula can only reference nodes **merged into that column**, and must reference at least one.
- If two merged nodes in the column have the same name, that name can't be used in a formula: the reference is ambiguous.
- Use brackets for powers: write `(2^3)^2` instead of `2^3^2`, and `(-[x])^2` or `-([x]^2)` instead of `-[x]^2`. Without brackets, Excel and the backend would disagree, so the backend rejects these.
- `ROUND` rounds half up, like Excel (`ROUND(2.675, 2)` = `2.68`). `digits` must be a whole number from -10 to 10.
- Maximum length: 500 characters.
- Node names containing `[` or `]` can't be referenced.

**Suggested examples for a help tooltip:**
```
Percentage:            =ROUND([Et]*100/[Nt], 1)
Non-events:            =[Nt]-[Et]
Mean change:           =[Mean post]-[Mean baseline]
CI half-width:         =ROUND(([Upper CI]-[Lower CI])/2, 3)
```

---

## 3. API changes

### 3.1 Create merged column — `POST /rtable_nodes/create_rtable_node`

New optional field **`formula`** on each item in `rtable_nodes`:

```json
{
  "project_id": 52,
  "rtable_id": 242,
  "rtable_nodes": [
    {
      "node_name": "Event rate (%)",
      "is_merge": true,
      "separator": "/",
      "merge_node_ids": [6165, 6166],
      "data_display_format": "Et / Nt",
      "formula": "=ROUND([Et]*100/[Nt], 1)",
      "is_highlight": false
    }
  ]
}
```

- `data_display_format` and `separator` are **still required** for merged columns, even when a formula is set. Send the usual default (node names joined by the separator).
- Leaving out `formula`, or sending `null`, `""` or whitespace, creates a normal merged column.
- Sending `formula` on a non-merged column (`"is_merge": false`) returns 400.
- If any column in the request is invalid, **no** columns from that request are saved.

### 3.2 Edit / remove formula — `PUT /rtable_nodes/update_rtable_node/{project_id}`

New optional field **`formula`**:

| Value sent | Effect |
|---|---|
| field left out, or `null` | formula unchanged |
| `"=[Et]+[Nt]"` | formula set or replaced (validated) |
| `""` (or whitespace) | formula removed; the column goes back to `data_display_format` |

```json
{ "id": 1426, "formula": "=ROUND([Et]*100/[Nt], 1)" }
```

It can be combined with the existing fields (`name`, `separator`, `is_highlight`, `child_nodes`). Changing the separator or name does not affect the formula. If the formula is invalid the whole update is rejected, and nothing in the request is saved.

### 3.3 Read columns — `GET /rtable_nodes/get_rtable_nodes/{project_id}/{rtable_id}`

Merged columns now include `formula` (`string` or `null`):

```json
{
  "id": 1426,
  "name": "Event rate (%)",
  "is_merge": true,
  "separator": "/",
  "data_display_format": "Et / Nt",
  "formula": "=ROUND([Et]*100/[Nt], 1)",
  "concatenated_names": "Et/Nt",
  "merge_node_ids": [{ "node_id": 6165, "node_name": "Et" }, { "node_id": 6166, "node_name": "Nt" }],
  "rank": 5, "is_highlight": false, "is_ai_assisted": false, "rtable_id": 242
}
```

Use this endpoint to pre-fill the formula editor and to find which columns are formula columns.

> ⚠️ **Known backend bug, not caused by this feature:** `GET /rtable_nodes/get_rtable_node/{project_id}/{rtable_node_id}` currently returns **500 for every merged column**, with or without a formula. The service returns `merge_nodes` but the response schema expects `merge_node_ids`. Until it's fixed, read single merged columns from the list endpoint above.

### 3.4 Table data — `GET /rtable/rtable_data/{project_id}/{rtable_id}` (one new key)

Each entry in a merged column's `results` now also has **`formula_result`**. Every other key, including `result`, is exactly as before:

```json
{
  "rtable_node_id": 1426,
  "name": "Event rate (%)",
  "is_merge": true,
  "results": [
    {
      "result": "41 / 65",
      "formula_result": "63.1",
      "subgroup_label": null,
      "treatment_arm_label": "Primary",
      "result_by_node_id": [
        { "node_id": 6165, "node_name": "Et", "result": "41", "is_resolved": false, "...": "..." },
        { "node_id": 6166, "node_name": "Nt", "result": "65", "is_resolved": false, "...": "..." }
      ],
      "is_paper_selected": true,
      "is_subgroup_treatment_arm_selected": false
    }
  ]
}
```

| Key | Value |
|---|---|
| `result` | **Unchanged.** The `data_display_format` text (e.g. `"41 / 65"`), or `null` as before. |
| `formula_result` | `null` if the column has no formula. Otherwise the computed number as a **string**, e.g. `"24"`, `"63.1"`, `"0.0733333333"`, `"-12"`, or `"-"` when it can't be computed (see §4). |
| `result_by_node_id` | Unchanged. The **input** values, useful for a tooltip such as "Et = 41, Nt = 65". |

- `formula_result` is on **every** merged-column cell, so `formula_result !== null` tells you the column is a formula column.
- Non-merged columns (`is_merge: false`) have no `formula_result` key.
- `formula` (the formula text) is not included in `rtable_data`. Get it from the column list (§3.3), e.g. for a header tooltip.
- `is_paper_selected` (default paper selection in multi-paper trials) is still based on `result`, as before. The formula doesn't affect it.

---

## 4. When `formula_result` is `-`

The backend returns `"-"` rather than guessing when:

| Case | Example |
|---|---|
| An input value is missing or not resolved yet | Nt not extracted for that arm |
| An input isn't a plain number | `NR`, `N/A`, `12 (24%)`, `200 (31.2)`, `12-15`, `<0.001`, `24%` |
| The result is undefined | division by zero, `SQRT` of a negative number, `LN(0)` |
| A referenced node was renamed or deleted in the I-table/outcome | formula `[Nt]` after Nt was renamed |

Accepted input numbers: `12`, `-3.5`, `.5`, `+3`, `1,234.5`, `1.2e-5`, and `−5` (the PDF minus sign).

**Suggested UI:** show `-` as now. Optionally show a tooltip on formula cells: *"Couldn't calculate: an input is missing or isn't a plain number"*, plus the input values from `result_by_node_id`.

Combined values like `200 (31.2)` are common in the data. Users must extract events and totals as **separate nodes** for formulas to work.

---

## 5. Number display

- No trailing zeros, at most 10 decimals: `24`, `1.5`, `0.3333333333`.
- To show a fixed number of decimals, users should wrap the formula in `ROUND(…, n)`. The backend still removes trailing zeros (`ROUND(24.0, 1)` shows `24`), the same as Excel's default number format.

---

## 6. Validation errors (HTTP 400)

All errors are `{"detail": "<plain string>"}`. Show `detail` as-is next to the formula input; the messages are written for end users.

| `detail` | Cause |
|---|---|
| `Formula is empty` | only `=` or spaces |
| `Formula is not a valid expression` | syntax error, e.g. `=[Et]*` |
| `Formula has an unmatched '[' or ']'` | `[Et` |
| `Unknown name 'Et'. Put node names in square brackets, e.g. [Et]` | node name without brackets |
| `Formula references 'Ec', which is not one of this column's merged nodes` | reference outside the column |
| `Formula reference 'Event' is ambiguous: more than one merged node has that name` | duplicate node names in the column |
| `Formula must reference at least one merged node, e.g. [Events]` | e.g. `=100` |
| `Unsupported function. Allowed functions: ROUND, ABS, SQRT, LN, LOG10, EXP, MIN, MAX` | e.g. `SUM(...)`, `IF(...)` |
| `Wrong number of arguments for ROUND` | e.g. `ROUND()` |
| `Formula may only contain numbers, [node] references, operators and functions` | text or strings in the formula |
| `Formula contains an unsupported expression` | comparisons, `%`, `//`, etc. |
| `Add brackets to chained powers, e.g. (2^3)^2` | `2^3^2` |
| `Add brackets to a negative power, e.g. (-[x])^2 or -([x]^2)` | `-[x]^2` |
| `Formula must be at most 500 characters` | too long |
| `Formulas are only supported on merged columns` | formula on a non-merged column |

Other status codes are unchanged: `401` (no or invalid token), `403` (no R-table edit privilege), `404` (column or R-table not found), `422` (wrong type, e.g. `"formula": 123`).

There is **no separate "validate" endpoint**. Validation happens on create and update. If you want instant feedback, a light client-side check is enough: balanced brackets, and `[names]` that are in the column. The backend stays the source of truth.

---

## 7. Suggested UI flow

1. **Create / edit merged column dialog:** add an optional **"Formula"** input below "Data display format".
   - Show chips for the column's merged nodes. Clicking a chip inserts `[Node name]` at the cursor.
   - Help text with the syntax and examples from §2.
   - Show the backend's `detail` message below the input on 400.
   - A "Remove formula" action sends `"formula": ""`.
2. **Table header:** mark formula columns, e.g. an ƒx icon with the formula in a tooltip, using `formula` from §3.3.
3. **Cells:** for merged columns, show `formula_result` when it isn't `null`, otherwise `result` as today:
   ```js
   const cellText = cell.formula_result ?? cell.result;
   ```
   Optionally show `result` (the inputs, e.g. `41 / 65`) as a tooltip or secondary line under the computed value.
4. **Editing a formula cell:** the computed value **cannot be edited directly**; there is no endpoint to store it. To change it, users edit the **input** values: the existing per-R-table override (`PUT /rtable/update_rtable_extraction/{project_id}/{rtable_id}`) on the merged nodes in `result_by_node_id`. Refetch `rtable_data` afterwards and the formula recalculates using the override.
5. **Sorting:** sorting happens on the frontend. For formula columns, sort by `formula_result` **numerically** (`parseFloat`) with `"-"` last; text sorting would put `"100"` before `"24"`.
6. **After an I-table/outcome node is renamed:** formulas still reference the old name and show `-`. Consider warning the user to update formulas that use that node.

---

## 8. Other screens and backend behaviour

- **Manuscript tables and guideline / public website pages** receive the same R-table data from the backend, so their cells also carry `formula_result`. ⚠️ Wherever those screens render R-table cells, apply the same `formula_result ?? result` rule, or they'll keep showing the display text. Snapshots that were already published don't change.
- **Copy R-table** copies formulas (backend).
- **Delete column** works as before.
- Existing R-tables without formulas render exactly as before; the only difference is `formula_result: null` on merged cells (verified on all 112 R-tables in the dev database).

---

## 9. Quick test checklist for frontend QA

- [ ] Create a merged column with `=ROUND([Et]*100/[Nt], 1)` → cells show percentages (`formula_result`), and `result` still holds `Et / Nt` text.
- [ ] A merged column without a formula → `formula_result` is `null` and cells look exactly as before.
- [ ] Create with `=[Et]*` → error "Formula is not a valid expression" shown, column not created.
- [ ] Create with `=[Something]` → error naming `Something`.
- [ ] Edit the formula → table updates after refetch.
- [ ] Remove the formula (`""`) → column shows display-format text again.
- [ ] Override an input value (e.g. Et) for one cell → the formula cell recalculates.
- [ ] An input of `NR` → cell shows `-`.
- [ ] Sort the formula column → numeric order, `-` last.
- [ ] Copy the R-table → the formula column is copied and shows the same values.
- [ ] Open a manuscript and a guideline page that embed the R-table → computed values appear (after those screens use `formula_result`).
