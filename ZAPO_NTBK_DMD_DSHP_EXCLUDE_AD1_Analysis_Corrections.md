# ZAPO_NTBK_DMD_DSHP_EXCLUDE — AD1 Live Analysis & Code Corrections

**System:** AD1 (`http://DEVSCMAD1.ril.com:8000`)  
**Function group:** `ZAPO_NTBK`  
**Function module:** `ZAPO_NTBK_DMD_DSHP_EXCLUDE`  
**Include:** `SAPLZAPO_NTBK`  
**Related FM:** `ZAPO_NTBK_PROD_DSHP_EXCLUDE`  
**Source verified:** AD1 via MCP `get_function_source_mcp` — 29-Jun-2026

---

## Executive summary

| # | Observation | Root cause in **current AD1 code** | Correction priority |
|---|-------------|----------------------------------|---------------------|
| 1 | `NTBK_QTY` shows DC **20+25** even when input has only `20` or only `25`; with `20,25` qty appears **double** | Partial ZVTWEG filter on `lt_prod_filtered` only; **no per-row `zvtweg` match** when collecting; possible **zvtweg format mismatch** (`'20'` vs `'020'`); demand hash built **without** `lt_vtweg` filter | **P1** |
| 2 | `NTBK_QTY` should be **RM qty = FG qty × BOM ratio** (e.g. × 0.991) | **BOM multiplication removed** in CD8085121 refactor — code sets `ntbk_qty = prod_qty` only; `lt_log_bom_2` / `req_qty_pmt_matnr_out` **not used** in output loop | **P1** |
| 3 | `NTBK_LOCFR = 40N5,40N6` → **no output**; should sum both plants | `ntbk_locfr` compare uses **string mismatch** (external `40N5` vs internal loc ID); `COLLECT` keeps `ntbk_locfr` in key → no plant rollup; RM path still has `SPLIT <lfs_out>-ntbk_locfr` **before** `lfs_out` exists | **P1** |

---

## Current AD1 code pattern (verified)

The deployed FM was refactored (CD8085121) to use `lt_prod_filtered`, `lt_output_agg`, and hashed demand keys. Core output logic is now:

```abap
LOOP AT lt_prod_filtered INTO lw_prod_dshpex_op.
  " ... demand / mat / loc validation via lt_dmd_h_* ...
  lw_output_agg = lw_output.
  CLEAR: lw_output_agg-zvtweg, lw_output_agg-ntbk_qty.
  lw_output_agg-ntbk_qty = lw_prod_dshpex_op-ntbk_qty.    " ← NO BOM multiply
  COLLECT lw_output_agg INTO lt_output_agg.
ENDLOOP.

LOOP AT lt_output_agg INTO lw_output_agg.
  lw_output_agg-zvtweg = <lfs_input>-zvtweg.
  APPEND lw_output_agg TO lt_output.
ENDLOOP.
```

`lt_prod_filtered` is built once per input row:

```abap
LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op
  WHERE pdh1 = <lfs_input>-pdh1 AND pdh2 = <lfs_input>-pdh2.
  IF lt_vtweg_h IS NOT INITIAL.
    READ TABLE lt_vtweg_h WITH TABLE KEY vtweg = lw_prod_dshpex_op-zvtweg.
    IF sy-subrc NE 0. CONTINUE. ENDIF.
  ENDIF.
  APPEND lw_prod_dshpex_op TO lt_prod_filtered.
ENDLOOP.
```

---

## Observation 1 — Distribution channel double-counting

### Symptoms

- Input `ZVTWEG = 20` → output qty looks like **20 + 25** combined.
- Input `ZVTWEG = 20,25` → qty appears **doubled** vs expected.

### Root causes in AD1

**1a. ZVTWEG format mismatch on filter**

Input split uses `CONDENSE` on `lt_vtweg`, but `lw_prod_dshpex_op-zvtweg` is **not normalized** before `READ TABLE lt_vtweg_h`. If prod has `'020'` and input has `'20'`, filter may fail open or behave inconsistently.

**1b. Demand hash ignores `lt_vtweg`**

When building `lt_dmd_h_mat_plant` / `lt_dmd_h_mat_loc_plant`, the loop over `lt_dmdvsup` applies division filter but **not** distribution-channel filter:

```abap
LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup>.
  " ... div filter only ...
  INSERT ls_dmd_h_mat_plant INTO TABLE lt_dmd_h_mat_plant.  " all DCs included
ENDLOOP.
```

This allows prod rows for plants/demand combinations outside the intended DC scope to pass validation.

**1c. COLLECT sums all filtered prod DC rows**

For input `ZVTWEG = 20,25`, summing both DC prod rows is **correct**.  
For input `ZVTWEG = 20` only, if filter works, only DC 20 prod rows should be in `lt_prod_filtered`. If qty still includes DC 25, the filter is failing (1a) or `ZAPO_NTBK_PROD_DSHP_EXCLUDE` returns pre-aggregated cross-DC qty.

**1d. Possible duplicate processing**

RM-only block (when `pdh1`/`pdh2` initial) **and** `ZAPO_NTBK_COMB_PREPARE` loop both append to `lt_output` for the same input index — can produce duplicate lines.

### Correction — Observation 1

**Step 1:** Normalize ZVTWEG on prod rows when building `lt_prod_filtered`:

```abap
LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op
  WHERE pdh1 = <lfs_input>-pdh1
    AND pdh2 = <lfs_input>-pdh2.

  DATA(lv_prod_vtweg) = lw_prod_dshpex_op-zvtweg.
  CONDENSE lv_prod_vtweg NO-GAPS.

  IF lt_vtweg_h IS NOT INITIAL.
    READ TABLE lt_vtweg_h WITH TABLE KEY vtweg = lv_prod_vtweg
      TRANSPORTING NO FIELDS.
    IF sy-subrc NE 0.
      CONTINUE.
    ENDIF.
  ENDIF.

  lw_prod_dshpex_op-zvtweg = lv_prod_vtweg.
  APPEND lw_prod_dshpex_op TO lt_prod_filtered.
ENDLOOP.
```

**Step 2:** Add ZVTWEG filter when building demand hash:

```abap
IF lt_vtweg_h IS NOT INITIAL.
  READ TABLE lt_vtweg_h WITH TABLE KEY vtweg = <lfs_dmdvsup>-zvtweg
    TRANSPORTING NO FIELDS.
  IF sy-subrc NE 0.
    CONTINUE.
  ENDIF.
ENDIF.
```

**Step 3:** When collecting, match prod row DC explicitly (defence in depth):

```abap
" Only collect prod qty for DCs in input — already in lt_prod_filtered
" After BOM fix (Obs 2): ntbk_qty = prod_qty * bom_ratio per prod row
```

**Step 4:** Prevent double append — use `ELSE` so only **one** path runs per input:

```abap
IF <lfs_input>-ntbk_matnr IS INITIAL AND ... RM-only conditions ...
  " RM-only processing
ELSE.
  CALL FUNCTION 'ZAPO_NTBK_COMB_PREPARE' ...
  " COMB processing
ENDIF.
```

---

## Observation 2 — RM requirement (BOM ratio) missing

### Expected business rule

```
NTBK_QTY = FG_Production_Qty × BOM_RM_Ratio

BOM_RM_Ratio = req_prnt_qty ÷ pds_out_qty
  (from ZAPO_OPT_LOG_BOM for NTBK_RMMATNR, FG, plant, bucket)
```

Example: BOM RM = 99.1% of FG → `NTBK_QTY = FG_Qty × 0.991`.

### Root cause in AD1

The refactored code **removed** the original formula:

```abap
" ORIGINAL (correct intent):
lw_output_1-ntbk_qty = lw_prod_dshpex_op-ntbk_qty
                       * <lfs_dmdvsup>-req_qty_pmt_matnr_out.

" CURRENT AD1 (incorrect):
lw_output_agg-ntbk_qty = lw_prod_dshpex_op-ntbk_qty.
```

`lt_log_bom_2` is still populated earlier in the FM but **never read** in the output section.

### Correction — Observation 2

Add FORM `get_bom_rm_ratio` and use it in every prod collect loop.

```abap
*&---------------------------------------------------------------------*
*& Form get_bom_rm_ratio
*&  Returns req_prnt_qty / pds_out_qty for FG+plant+bucket+RM
*&---------------------------------------------------------------------*
FORM get_bom_rm_ratio
  USING
    iv_sessionname TYPE zapo_sessionname
    iv_fg_matnr    TYPE /sapapo/matnr
    iv_fg_locno    TYPE /sapapo/locno
    iv_bucket      TYPE zapo_bucket
    iv_rm_matnr    TYPE zapo_matnr_out
  CHANGING
    cv_ratio       TYPE zapo_req_inp_qty
    ct_log_bom     TYPE STANDARD TABLE.

  DATA: lv_bucket TYPE zapo_bucket,
        ls_bom    TYPE lty_log_bom_1.

  CLEAR cv_ratio.
  lv_bucket = iv_bucket.
  CONDENSE lv_bucket NO-GAPS.

  READ TABLE ct_log_bom INTO ls_bom
    WITH KEY sessionname = iv_sessionname
             fg_matnr    = iv_fg_matnr
             fg_locno    = iv_fg_locno
             bucket      = lv_bucket
             matnr_out   = iv_rm_matnr
    BINARY SEARCH.

  IF sy-subrc = 0 AND ls_bom-pds_out_qty NE 0.
    cv_ratio = ls_bom-req_prnt_qty / ls_bom-pds_out_qty.
  ENDIF.

ENDFORM.
```

**Replace collect assignment** (in both RM-only and COMB_PREPARE loops):

```abap
DATA: lv_bom_ratio TYPE zapo_req_inp_qty,
      lv_rm_qty    TYPE zapo_ntbk_qty.

PERFORM get_bom_rm_ratio USING
  is_session-sessionname
  lw_prod_dshpex_op-ntbk_matnr          " FG matnr from prod output
  lw_prod_dshpex_op-ntbk_locfr          " FG plant
  lw_prod_dshpex_op-ntbk_mth            " use bucket from plandrv if stored on prod row
  <lfs_input>-ntbk_rmmatnr
CHANGING lv_bom_ratio lt_log_bom_2.

IF lv_bom_ratio IS INITIAL.
  CONTINUE.    " or derive bucket from ntbk_mth via lt_plandrv
ENDIF.

lv_rm_qty = lw_prod_dshpex_op-ntbk_qty * lv_bom_ratio.

lw_output_agg = lw_output.
CLEAR: lw_output_agg-zvtweg, lw_output_agg-ntbk_locfr, lw_output_agg-ntbk_qty.
lw_output_agg-ntbk_qty = lv_rm_qty.
COLLECT lw_output_agg INTO lt_output_agg.
```

> **Note:** Map `iv_bucket` from `lw_prod_dshpex_op` bucket field if available, or reverse-map `ntbk_mth` → bucket using `lt_plandrv` (same as demand processing). Verify field availability on `zapo_ntbk_detail` from `ZAPO_NTBK_PROD_DSHP_EXCLUDE`.

---

## Observation 3 — Multiple plants (`NTBK_LOCFR = 40N5,40N6`)

### Symptoms

- Input `NTBK_LOCFR = 40N5,40N6` → **no rows** in `ZAPO_NTBK_DETAIL`.
- Expected: **one row**, `NTBK_LOCFR = 40N5,40N6`, `NTBK_QTY` = sum(40N5) + sum(40N6).

### Root causes in AD1

**3a. Direct string compare fails (COMB loop)**

```abap
IF <lfs_out>-ntbk_locfr IS NOT INITIAL
AND lw_prod_dshpex_op-ntbk_locfr NE <lfs_out>-ntbk_locfr.
  CONTINUE.
ENDIF.
```

- `lw_prod_dshpex_op-ntbk_locfr` = internal APO location ID.
- `<lfs_out>-ntbk_locfr` = external plant (`40N5`) or combined string (`40N5,40N6`).
- Compare **always fails** → all prod rows skipped → **no output**.

**3b. No plant summation in COLLECT**

Only `zvtweg` is cleared before COLLECT; `ntbk_locfr` remains in the COLLECT key → separate rows per plant (if compare worked), not one combined row.

**3c. RM-only path bug (still in AD1)**

```abap
SPLIT <lfs_out>-ntbk_locfr AT ',' INTO TABLE lt_loc.   " lfs_out undefined here
```

Must be `<lfs_input>-ntbk_locfr`.

**3d. `ZAPO_NTBK_COMB_PREPARE` may not split multi-plant**

If `COMB_PREPARE` returns one row with `ntbk_locfr = '40N5,40N6'`, plant filter (3a) blocks all rows.

### Correction — Observation 3

**Step 1:** Split plants from **input** at start of input loop:

```abap
IF <lfs_input>-ntbk_locfr IS NOT INITIAL.
  SPLIT <lfs_input>-ntbk_locfr AT ',' INTO TABLE lt_loc.
  LOOP AT lt_loc ASSIGNING FIELD-SYMBOL(<ls_loc_ref>).
    CONDENSE <ls_loc_ref> NO-GAPS.
    TRANSLATE <ls_loc_ref> TO UPPER CASE.
  ENDLOOP.
ENDIF.
```

**Step 2:** Add FORM `plant_matches` using `it_loc`:

```abap
FORM plant_matches
  USING
    iv_prod_locfr  TYPE /sapapo/locno
    iv_ref_locfr   TYPE zapo_ntbk_locno
    it_loc_tab       TYPE zapo_ntbk_loc_t
    it_loc_list      TYPE STANDARD TABLE OF zapo_ntbk_locno
  CHANGING
    rv_match TYPE abap_bool.

  DATA: ls_loc TYPE zapo_ntbk_loc,
        lv_ref TYPE zapo_ntbk_locno.

  CLEAR rv_match.

  IF iv_ref_locfr IS NOT INITIAL.
  IF iv_prod_locfr = iv_ref_locfr.
    rv_match = abap_true. RETURN.
  ENDIF.
  READ TABLE it_loc_tab INTO ls_loc WITH KEY locno = iv_ref_locfr.
  IF sy-subrc = 0 AND ls_loc-locid = iv_prod_locfr.
    rv_match = abap_true. RETURN.
  ENDIF.
  ENDIF.

  IF it_loc_list IS NOT INITIAL.
    LOOP AT it_loc_list INTO lv_ref.
      IF iv_prod_locfr = lv_ref.
        rv_match = abap_true. RETURN.
      ENDIF.
      READ TABLE it_loc_tab INTO ls_loc WITH KEY locno = lv_ref.
      IF sy-subrc = 0 AND ls_loc-locid = iv_prod_locfr.
        rv_match = abap_true. RETURN.
      ENDIF.
    ENDLOOP.
  ENDIF.

ENDFORM.
```

**Step 3:** Replace direct locfr compare in COMB loop:

```abap
DATA lv_plant_ok TYPE abap_bool.

IF <lfs_out>-ntbk_locfr IS NOT INITIAL OR lt_loc IS NOT INITIAL.
  PERFORM plant_matches USING
    lw_prod_dshpex_op-ntbk_locfr
    <lfs_out>-ntbk_locfr
    it_loc
    lt_loc
  CHANGING lv_plant_ok.
  IF lv_plant_ok = abap_false.
    CONTINUE.
  ENDIF.
ENDIF.
```

**Step 4:** Sum plants in COLLECT — clear `ntbk_locfr`; set combined value on flush:

```abap
CLEAR: lw_output_agg-zvtweg, lw_output_agg-ntbk_locfr, lw_output_agg-ntbk_qty.

" On flush:
lw_output_agg-zvtweg    = <lfs_input>-zvtweg.
lw_output_agg-ntbk_locfr = <lfs_input>-ntbk_locfr.   " 40N5,40N6
```

**Step 5:** Update `flush` loop signature:

```abap
LOOP AT lt_output_agg INTO lw_output_agg.
  lw_output_agg-zvtweg     = <lfs_input>-zvtweg.
  lw_output_agg-ntbk_locfr = <lfs_input>-ntbk_locfr.
  APPEND lw_output_agg TO lt_output.
ENDLOOP.
```

---

## Complete FORM routines (paste before `ENDFUNCTION`)

```abap
  FORM get_bom_rm_ratio
    USING
      iv_sessionname TYPE zapo_sessionname
      iv_fg_matnr    TYPE /sapapo/matnr
      iv_fg_locno    TYPE /sapapo/locno
      iv_bucket      TYPE zapo_bucket
      iv_rm_matnr    TYPE zapo_matnr_out
    CHANGING
      cv_ratio TYPE zapo_req_inp_qty
      ct_log_bom TYPE STANDARD TABLE.

    DATA ls_bom TYPE lty_log_bom_1.
    DATA(lv_bkt) = iv_bucket.
    CONDENSE lv_bkt NO-GAPS.
    CLEAR cv_ratio.

    READ TABLE ct_log_bom INTO ls_bom
      WITH KEY sessionname = iv_sessionname
               fg_matnr    = iv_fg_matnr
               fg_locno    = iv_fg_locno
               bucket      = lv_bkt
               matnr_out   = iv_rm_matnr
      BINARY SEARCH.
    IF sy-subrc = 0 AND ls_bom-pds_out_qty NE 0.
      cv_ratio = ls_bom-req_prnt_qty / ls_bom-pds_out_qty.
    ENDIF.
  ENDFORM.

  FORM plant_matches
    USING
      iv_prod_locfr TYPE /sapapo/locno
      iv_ref_locfr  TYPE zapo_ntbk_locno
      it_loc_tab    TYPE zapo_ntbk_loc_t
      it_loc_list   TYPE STANDARD TABLE OF zapo_ntbk_locno
    CHANGING
      rv_match TYPE abap_bool.

    DATA: ls_loc TYPE zapo_ntbk_loc, lv_ref TYPE zapo_ntbk_locno.
    CLEAR rv_match.

    IF iv_ref_locfr IS NOT INITIAL.
      IF iv_prod_locfr = iv_ref_locfr.
        rv_match = abap_true. RETURN.
      ENDIF.
      READ TABLE it_loc_tab INTO ls_loc WITH KEY locno = iv_ref_locfr.
      IF sy-subrc = 0 AND ls_loc-locid = iv_prod_locfr.
        rv_match = abap_true. RETURN.
      ENDIF.
    ENDIF.

    LOOP AT it_loc_list INTO lv_ref.
      IF iv_prod_locfr = lv_ref.
        rv_match = abap_true. RETURN.
      ENDIF.
      READ TABLE it_loc_tab INTO ls_loc WITH KEY locno = lv_ref.
      IF sy-subrc = 0 AND ls_loc-locid = iv_prod_locfr.
        rv_match = abap_true. RETURN.
      ENDIF.
    ENDLOOP.
  ENDFORM.
```

---

## Change checklist (AD1 transport)

| Step | Change | Fixes |
|------|--------|-------|
| 1 | Restore `ntbk_qty = fg_prod_qty × bom_ratio` via `get_bom_rm_ratio` | Obs 2 |
| 2 | Normalize `zvtweg` on prod; filter demand hash by `lt_vtweg_h` | Obs 1 |
| 3 | `CLEAR ntbk_locfr` in COLLECT; flush with `<lfs_input>-ntbk_locfr` | Obs 3 |
| 4 | Replace `mfg_plant NE ntbk_locfr` with `plant_matches` + `it_loc` | Obs 3 |
| 5 | Split `<lfs_input>-ntbk_locfr` (not `<lfs_out>`) | Obs 3 |
| 6 | RM-only vs COMB_PREPARE — `IF/ELSE` to avoid duplicate `lt_output` | Obs 1 |
| 7 | Keep sort only: `SORT et_output BY ntbk_index ntbk_mth ...` | General |

---

## Test scenarios (AD1)

| # | Input | Expected `ZAPO_NTBK_DETAIL` |
|---|-------|----------------------------|
| T1 | `ZVTWEG=20` only | Qty = FG₂₀ × BOM; **no** DC 25 component |
| T2 | `ZVTWEG=20,25` | **One row**; `ZVTWEG=20,25`; qty = (FG₂₀×BOM) + (FG₂₅×BOM) |
| T3 | BOM 99.1% | `NTBK_QTY ≈ FG_Qty × 0.991` (verify vs `ZAPO_OPT_LOG_BOM`) |
| T4 | `NTBK_LOCFR=40N5,40N6` | **One row**; locfr = `40N5,40N6`; qty = sum both plants |
| T5 | `NTBK_LOCFR=40N5` only | Qty for 40N5 only |
| T6 | Multiple months | Separate rows per month (not collapsed) |

---

## Related documents

- `ZAPO_NTBK_DMD_DSHP_EXCLUDE_ZVTWEG_Sum_Correction.md` — earlier ZVTWEG summation design  
- `Code_Correction_ZAPO_NTBK_DMD_DSHP_EXCLUDE.md` — `ZAPO_NTBK_PROD_DSHP_EXCLUDE` ST22 overflow  
- `ABAP Dump_ZAPO_NTBK_DMD_DSHP_EXCLUDE.xls` — runtime dump reference

---

**Author:** mahesh pathak  
**CD:** 8085121 follow-up  
**AD1 source read:** MCP `get_function_source_mcp` — 29-Jun-2026
