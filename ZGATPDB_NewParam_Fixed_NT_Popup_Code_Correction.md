# ZGATPDB — New param popup shows fixed NT (Plant) quantity

**Program:** `ZAPO_GATP_ALLOCATION_REPORT`  
**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**Popup:** `gATP Manual & NO Touch Report` (`get_gatprep_data`)  
**Version:** 1.0 — 25/05/2026  

---

## 1. Issue from screenshot

For **new parameter** selection, popup shows a **fixed NT (Plant)** quantity:

| Material | Location | UA Qty seen in grid | Popup MT (Plant) | Popup NT (Plant) | Observation |
|----------|----------|---------------------|------------------|------------------|-------------|
| `B120MA` | `3605` | `15` | `15` | `60` | NT appears as fixed bucket |
| `H110MA` | `3617` | `42` | `42` | `84` | NT appears as fixed bucket |

User observation:

- With **old params**, popup is correct.
- With **new param**, popup shows a **fixed NT** quantity by material.
- So the new-param path is injecting / preserving an SO-based NT bucket that should **not** appear in the popup for this case.

---

## 2. Root cause

There are **two places** in the new-param path that can force a fixed NT value.

### 2.1 `get_data` — custom/fixed NT bucket logic

The standard new-param CHANGE F in AD1 uses:

```abap
<fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.
<fs_output>-no_touch     = <fs_output>-inc_ord_quan - <fs_output>-manual_touch.
```

If any custom code was added to **override** this with SO bucket logic such as:

```abap
<fs_output>-no_touch = lw_so_nt.
<fs_output>-inc_ord_quan = lw_so_in.
```

then popup will show **fixed NT** quantities like **60** / **84**, because `no_touch` is no longer derived from the current row result; it is taken directly from `ZAPO_SO_LIST`.

This matches the symptom very closely.

### 2.2 `get_gatprep_data` — popup always adds `get_zapo_so_list`

Current popup logic:

```abap
go_alloc->get_zapo_so_list( im_output = lt_output ).
gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant.
gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant.
...
```

So even if the selected row in `gt_output` is correct, popup still adds **SO_LIST NT totals** on top.

For new param, that SO add-on behaves like a **fixed NT bucket** by material/location/group customer, which is why:

- `B120MA` shows **60**
- `H110MA` shows **84**

while the grid row itself is showing UA quantities (`15`, `42`).

### 2.3 Why old params are correct

Old-param behavior is effectively driven by the **selected row result**.  
New-param behavior is wrong because popup is taking **fixed SO NT** from `get_zapo_so_list` and/or from earlier custom fixed-NT code.

---

## 3. Required behavior

For **popup in new-param mode**, use the **selected row values** only:

```text
MT popup = SUM( gw_output-manual_touch )
NT popup = SUM( gw_output-no_touch )
```

Do **not** inject a separate fixed NT bucket from `ZAPO_SO_LIST` into popup totals.

`ZAPO_SO_LIST` can still be used for:

- cap / validation
- GP block / DLBL filters
- monthly logic

but not as an extra popup NT add-on in this scenario.

---

## 4. Code correction

## 4.1 `get_data` — revert any fixed SO-bucket NT assignment

If code similar to the below was added earlier:

```abap
READ TABLE lt_so_cvc_ntmt INTO ls_so_cvc ...
lw_so_nt = ls_so_cvc-nt_qty.
lw_so_mt = ls_so_cvc-mt_qty.
lw_so_in = ls_so_cvc-nt_qty + ls_so_cvc-mt_qty.

<fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.
<fs_output>-no_touch     = lw_so_nt.
IF lw_so_in > 0.
  <fs_output>-inc_ord_quan = lw_so_in.
ENDIF.
```

**Remove / revert it** for popup-related new-param logic.

### Replace with

Use the standard dynamic split:

```abap
                      CLEAR lw_mt_adj_sum.
                      READ TABLE lt_mt_adj_sum INTO lw_mt_adj_sum
                           WITH KEY product       = <fs_output>-material
                                    location      = <fs_output>-location
                                    division      = <fs_output>-div
                                    group_cust    = <fs_output>-grp_cust
                                    dist_channel  = <fs_output>-dist_chan.
                      IF sy-subrc = 0.
                        <fs_output>-manual_touch = lw_mt_adj_sum-usr_adj.
                      ELSE.
                        CLEAR <fs_output>-manual_touch.
                      ENDIF.

                      <fs_output>-no_touch = <fs_output>-inc_ord_quan - <fs_output>-manual_touch.

                      IF <fs_output>-no_touch < 0.
                        CLEAR <fs_output>-no_touch.
                      ENDIF.

                      IF <fs_output>-manual_touch < 0.
                        CLEAR <fs_output>-manual_touch.
                      ENDIF.
```

Apply this in all new-param CHANGE F branches where fixed NT code may have been introduced.

---

## 4.2 `get_gatprep_data` — skip `get_zapo_so_list` add-on for new param

This is the **main correction** for the popup.

### Current code

```abap
go_alloc->get_zapo_so_list( im_output = lt_output ).
gw_mtch_tot_plant = gw_mtch_tot_plant + gw_apo_so_mt_tot_plant.
gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant.
gw_mtch_pp_plant  = gw_mtch_pp_plant  + gw_apo_so_mt_pp_plant.
gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant.
...
```

### Corrected code

```abap
    READ TABLE gt_zapoparam TRANSPORTING NO FIELDS
         WITH KEY param1 = 'GATP'.

    IF sy-subrc <> 0.
      " Old params: keep existing SO_LIST add-on logic
      go_alloc->get_zapo_so_list( im_output = lt_output ).
      gw_mtch_tot_plant = gw_mtch_tot_plant + gw_apo_so_mt_tot_plant.
      gw_ntch_tot_plant = gw_ntch_tot_plant + gw_apo_so_nt_tot_plant.
      gw_mtch_pe_plant  = gw_mtch_pe_plant  + gw_apo_so_mt_pe_plant.
      gw_ntch_pe_plant  = gw_ntch_pe_plant  + gw_apo_so_nt_pe_plant.
      gw_mtch_pp_plant  = gw_mtch_pp_plant  + gw_apo_so_mt_pp_plant.
      gw_ntch_pp_plant  = gw_ntch_pp_plant  + gw_apo_so_nt_pp_plant.
      gw_mtch_pvc_plant = gw_mtch_pvc_plant + gw_apo_so_mt_pvc_plant.
      gw_ntch_pvc_plant = gw_ntch_pvc_plant + gw_apo_so_nt_pvc_plant.
      gw_mtch_elstmr_plant = gw_mtch_elstmr_plant + gw_apo_so_mt_elstmr_plant.
      gw_ntch_elstmr_plant = gw_ntch_elstmr_plant + gw_apo_so_nt_elstmr_plant.

      gw_mtch_tot_depot = gw_mtch_tot_depot + gw_apo_so_mt_tot_depot.
      gw_ntch_tot_depot = gw_ntch_tot_depot + gw_apo_so_nt_tot_depot.
      gw_mtch_pe_depot  = gw_mtch_pe_depot  + gw_apo_so_mt_pe_depot.
      gw_ntch_pe_depot  = gw_ntch_pe_depot  + gw_apo_so_nt_pe_depot.
      gw_mtch_pp_depot  = gw_mtch_pp_depot  + gw_apo_so_mt_pp_depot.
      gw_ntch_pp_depot  = gw_ntch_pp_depot  + gw_apo_so_nt_pp_depot.
      gw_mtch_pvc_depot = gw_mtch_pvc_depot + gw_apo_so_mt_pvc_depot.
      gw_ntch_pvc_depot = gw_ntch_pvc_depot + gw_apo_so_nt_pvc_depot.
      gw_mtch_elstmr_depot = gw_mtch_elstmr_depot + gw_apo_so_mt_elstmr_depot.
      gw_ntch_elstmr_depot = gw_ntch_elstmr_depot + gw_apo_so_nt_elstmr_depot.
    ELSE.
      " New param: popup should use selected-row MT/NT only
      " Do NOT add fixed SO_LIST NT/MT buckets here
    ENDIF.
```

This change makes popup behavior in **new-param mode** follow the **selected row output**, which is what the user says is correct in old-param mode.

---

## 4.3 Optional safer variant — add method flag instead of using `gt_zapoparam`

If you want a cleaner design, extend method signature:

```abap
METHODS: get_zapo_so_list
  IMPORTING im_output LIKE gt_output
            im_popup_add TYPE abap_bool DEFAULT abap_true.
```

Then in `get_gatprep_data`:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.
  go_alloc->get_zapo_so_list(
    EXPORTING
      im_output    = lt_output
      im_popup_add = abap_true ).
ELSE.
  go_alloc->get_zapo_so_list(
    EXPORTING
      im_output    = lt_output
      im_popup_add = abap_false ).
ENDIF.
```

But for immediate correction, the simple `IF sy-subrc <> 0.` guard in `get_gatprep_data` is sufficient.

---

## 5. Expected results after fix

For popup in **new-param mode**:

| Material | UA Qty | Popup MT | Popup NT |
|----------|--------|----------|----------|
| `B120MA` | `15` | `15` | **dynamic row result** (not fixed `60`) |
| `H110MA` | `42` | `42` | **dynamic row result** (not fixed `84`) |

Most importantly:

- popup must **not** always show `60` for `B120MA`
- popup must **not** always show `84` for `H110MA`
- old-param behavior remains unchanged

---

## 6. Test plan

| ID | Test | Expected |
|----|------|----------|
| T1 | Old params, `B120MA` popup | Same as before |
| T2 | New param, `B120MA`, UA qty `15` | MT = `15`, NT not fixed at `60` |
| T3 | New param, `H110MA`, UA qty `42` | MT = `42`, NT not fixed at `84` |
| T4 | Compare popup before/after with same row selection | Only new-param popup changes |
| T5 | GP / DLBL filter scenarios | No regression in SO-list filtering logic |

---

## 7. Recommendation

This issue looks like a **follow-on effect of earlier fixed-NT popup logic**.  
So the safest correction is:

1. **Revert any custom code that sets `no_touch` from SO NT bucket** in `get_data`.
2. **Stop adding `get_zapo_so_list` popup totals when new param is active**.

That keeps:

- **old params** unchanged
- **new param popup** aligned with the selected row result
- SO-list logic available for other non-popup use cases

---

## 8. Related files

| Topic | File |
|-------|------|
| GP / DLBL filter | `ZGATPDB_SORG1020_DLBL_NT_MT_Code_Correction.md` |
| Daily `PRIME_BCKU` save | `ZGATPDB_PRIME_BCKU_Daily_ABAP_Code_Change.md` |
| Earlier popup fixed-bucket idea | `ZGATPDB_Popup_NT_Reduced_UA_Order_Code_Correction.md` |

---

*Reference source checked from AD1 include `ZAPO_GATP_ALLOCATION_F005`: `get_gatprep_data` around `go_alloc->get_zapo_so_list( im_output = lt_output )` and the new-param CHANGE F NT/MT split blocks.*
