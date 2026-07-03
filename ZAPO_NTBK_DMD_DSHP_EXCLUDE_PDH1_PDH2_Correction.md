# ZAPO_NTBK_DMD_DSHP_EXCLUDE — PDH1 / PDH2 Correction (vs ZAPO_NTBK_DMD)

**System:** AD1 (`SAPLZAPO_NTBK` / function group `ZAPO_NTBK`)  
**Date:** 03-Jul-2026  
**Issue:** Rule `TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)` produces **no `NTBK_QTY`** in `ZAPO_NTBK_DETAIL` when `PDH1` and `PDH2` are maintained in `ZAPO_NTBK_REF`.  
**Reference (working):** Rule `TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG` via FM `ZAPO_NTBK_DMD` — same PDH values calculate correctly (see screenshot: ETHYLENE / 3925 / PPCOPOL. / ICP.).

> **AD1 note:** MCP was unavailable at document creation time. Analysis is based on AD1 source read on 29-Jun-2026 plus `ZAPO_NTBK_PROD_DSHP_EXCLUDE.txt` in workspace. Re-verify after transport.

---

## 1. Problem from your screenshots

| Input (`ZAPO_NTBK_REF`) | FM | PDH1 | PDH2 | Result in `ZAPO_NTBK_DETAIL` |
|-------------------------|-----|------|------|------------------------------|
| `TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)` | `ZAPO_NTBK_DMD_DSHP_EXCLUDE` | PPCOPOL. | ICP. | **No rows / NTBK_QTY blank** |
| `TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG` | `ZAPO_NTBK_DMD` | PPCOPOL. | ICP. | **Rows with NTBK_QTY** (e.g. 88.256, 105.907) |

Example input rows (index 1837–1839): Division 23, ZVTWEG 10,65 / 20,25 / 15, RM ETHYLENE, plant 3925, PDH1/PDH2 populated.

---

## 2. Root cause (confirmed)

### 2.1 Field name mismatch between PROD and DMD FM

`ZAPO_NTBK_DMD_DSHP_EXCLUDE` calls `ZAPO_NTBK_PROD_DSHP_EXCLUDE` and builds `lt_prod_filtered` with:

```abap
LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op
  WHERE pdh1 = <lfs_input>-pdh1
    AND pdh2 = <lfs_input>-pdh2.
  ...
  APPEND lw_prod_dshpex_op TO lt_prod_filtered.
ENDLOOP.
```

But `ZAPO_NTBK_PROD_DSHP_EXCLUDE` **never sets `pdh1` / `pdh2`** on `et_output`. It sets **`prodh1` / `prodh2`**:

```abap
" ZAPO_NTBK_PROD_DSHP_EXCLUDE (PROD FM — unchanged dummy-ship logic)
lw_output_1-prodh1 = <lfs_ref>-pdh1.
lw_output_1-prodh2 = <lfs_ref>-pdh2.
```

| Structure field | Set by PROD_DSHP_EXCLUDE? | Used in DMD_DSHP_EXCLUDE filter? |
|-----------------|--------------------------|----------------------------------|
| `pdh1` / `pdh2` | **No** (stays initial) | **Yes** — `WHERE pdh1 = ...` |
| `prodh1` / `prodh2` | **Yes** | **No** |

### 2.2 Why it fails only when PDH1/PDH2 are maintained

| Input PDH1/PDH2 | Filter condition | Prod row `pdh1` | Match? |
|-----------------|------------------|-----------------|--------|
| **Initial** | `pdh1 = initial AND pdh2 = initial` | initial | **All prod rows match** → some output (may be wrong qty) |
| **PPCOPOL. / ICP.** | `pdh1 = 'PPCOPOL.' AND pdh2 = 'ICP.'` | **initial** | **No rows match** → `lt_prod_filtered` empty → **no NTBK_QTY** |

This exactly matches your observation.

### 2.3 Why `ZAPO_NTBK_DMD` works

`ZAPO_NTBK_DMD` uses FM **`ZAPO_NTBK_PROD`** (not `PROD_DSHP_EXCLUDE`). That prod FM populates **`pdh1` / `pdh2`** on the production output (or the DMD loop matches the fields prod actually sets). The DMD loop then:

1. Filters prod by `pdh1` / `pdh2`
2. Applies **BOM ratio**: `ntbk_qty = prod_qty × (req_prnt_qty / pds_out_qty)` from `lt_log_bom_2`
3. Loops `lt_dmdvsup` with division / ZVTWEG filters

The CD8085121 refactor of `ZAPO_NTBK_DMD_DSHP_EXCLUDE` removed the `lt_dmdvsup` + BOM loop and only collects raw prod qty — a second gap after the PDH filter is fixed.

### 2.4 RM-only path also excludes PDH rows

```abap
IF ... AND <lfs_input>-pdh1 IS INITIAL AND <lfs_input>-pdh2 IS INITIAL ...
```

When PDH1/PDH2 are filled (your case), this block is **skipped**. Processing relies entirely on `lt_prod_filtered` — which is empty due to §2.1.

---

## 3. Comparison — ZAPO_NTBK_DMD vs ZAPO_NTBK_DMD_DSHP_EXCLUDE

| Step | ZAPO_NTBK_DMD (working) | ZAPO_NTBK_DMD_DSHP_EXCLUDE (current AD1) |
|------|-------------------------|-------------------------------------------|
| Prod FM | `ZAPO_NTBK_PROD` | `ZAPO_NTBK_PROD_DSHP_EXCLUDE` (dummy ship excluded) |
| PDH on prod output | `pdh1` / `pdh2` populated | Only `prodh1` / `prodh2` populated |
| Prod filter | `WHERE pdh1 = input-pdh1 AND pdh2 = input-pdh2` | Same — **breaks** |
| BOM multiply | `prod × req_prnt_qty / pds_out_qty` | **Removed** — `ntbk_qty = prod_qty` only |
| Demand loop | `LOOP lt_dmdvsup` + BOM from `lt_log_bom_2` | Replaced by hash + prod collect |
| Dummy ship exclusion | N/A | **Retained in PROD FM** — do not change |

---

## 4. Required correction (DMD_DSHP_EXCLUDE only)

**Scope:** Fix PDH1/PDH2 handling and restore BOM multiply like `ZAPO_NTBK_DMD`.  
**Out of scope:** Do **not** modify dummy-customer exclusion in `ZAPO_NTBK_PROD_DSHP_EXCLUDE`.

---

### 4.1 Fix A — PDH filter on `lt_prod_filtered` (mandatory)

**Replace** the current `LOOP AT lt_prod_dshpex_op ... WHERE pdh1 = ... AND pdh2 = ...` with explicit PDH matching on **`prodh1` / `prodh2`** (with fallback to `pdh1` / `pdh2`):

```abap
      CLEAR: lt_prod_filtered.

      LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op.

        " --- PDH1 filter (match prodh1 OR pdh1 from PROD FM output) ---
        IF <lfs_input>-pdh1 IS NOT INITIAL.
          IF lw_prod_dshpex_op-prodh1 IS NOT INITIAL.
            IF lw_prod_dshpex_op-prodh1 <> <lfs_input>-pdh1.
              CONTINUE.
            ENDIF.
          ELSEIF lw_prod_dshpex_op-pdh1 <> <lfs_input>-pdh1.
            CONTINUE.
          ENDIF.
        ENDIF.

        " --- PDH2 filter ---
        IF <lfs_input>-pdh2 IS NOT INITIAL.
          IF lw_prod_dshpex_op-prodh2 IS NOT INITIAL.
            IF lw_prod_dshpex_op-prodh2 <> <lfs_input>-pdh2.
              CONTINUE.
            ENDIF.
          ELSEIF lw_prod_dshpex_op-pdh2 <> <lfs_input>-pdh2.
            CONTINUE.
          ENDIF.
        ENDIF.

        " --- ZVTWEG filter (existing logic, with normalize) ---
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

> Optional hardening: `CONDENSE` / compare PDH values case-insensitively if trailing spaces differ (`'PPCOPOL.'` vs `'PPCOPOL'`).

---

### 4.2 Fix B — Restore BOM multiply (align with ZAPO_NTBK_DMD)

Add FORM `get_bom_rm_ratio` (uses existing `lt_log_bom_2` — already loaded in FM):

```abap
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

    DATA: ls_bom   TYPE lty_log_bom_1,
          lv_bkt   TYPE zapo_bucket.

    CLEAR cv_ratio.
    lv_bkt = iv_bucket.
    CONDENSE lv_bkt NO-GAPS.

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
```

Helper to resolve bucket from month (`ntbk_mth` → `lt_plandrv`):

```abap
  FORM get_bucket_from_month
    USING
      iv_sessionid TYPE /sapapo/snpsession
      iv_ntbk_mth  TYPE zapo_ntbk_prd
    CHANGING
      cv_bucket    TYPE zapo_bucket
      ct_plandrv   TYPE zapo_plndrv_tt.

    DATA ls_pln TYPE zapo_plndrv.
    CLEAR cv_bucket.
    READ TABLE ct_plandrv INTO ls_pln
      WITH KEY sessionid = iv_sessionid date_mthyr = iv_ntbk_mth.
    IF sy-subrc = 0.
      cv_bucket = ls_pln-bucke.
    ENDIF.
  ENDFORM.
```

**Replace collect assignment** in both RM-only and COMB loops:

```abap
            DATA: lv_bom_ratio TYPE zapo_req_inp_qty,
                  lv_bucket    TYPE zapo_bucket,
                  lv_rm_qty    TYPE zapo_ntbk_qty.

            PERFORM get_bucket_from_month USING
              is_session-sessionid
              lw_prod_dshpex_op-ntbk_mth
            CHANGING lv_bucket lt_plandrv.

            PERFORM get_bom_rm_ratio USING
              is_session-sessionname
              lw_prod_dshpex_op-ntbk_matnr
              lw_prod_dshpex_op-ntbk_locfr
              lv_bucket
              <lfs_input>-ntbk_rmmatnr
            CHANGING lv_bom_ratio lt_log_bom_2.

            IF lv_bom_ratio IS INITIAL.
              CONTINUE.
            ENDIF.

            lv_rm_qty = lw_prod_dshpex_op-ntbk_qty * lv_bom_ratio.

            lw_output_agg = lw_output.
            CLEAR: lw_output_agg-zvtweg, lw_output_agg-ntbk_locfr, lw_output_agg-ntbk_qty.
            lw_output_agg-ntbk_qty = lv_rm_qty.
            COLLECT lw_output_agg INTO lt_output_agg.
```

This restores the DMD formula:

```
NTBK_QTY = FG_Prod_Qty × (RM_req ÷ FG_out)   e.g. FG × 0.991
```

---

### 4.3 Fix C — Copy PDH to output header (optional but recommended)

Ensure output lines carry input PDH values:

```abap
            lw_output-pdh1 = <lfs_input>-pdh1.
            lw_output-pdh2 = <lfs_input>-pdh2.
```

Before `lw_output_agg = lw_output`.

---

### 4.4 Fix D — Division filter when DIV is maintained

Your rows have `DIV = 23`. Ensure division filter applies when building `lt_dmd_h_*`:

```abap
        IF lt_div IS NOT INITIAL.
          READ TABLE lt_div WITH KEY table_line = <lfs_dmdvsup>-planner_snp
            TRANSPORTING NO FIELDS.
          IF sy-subrc NE 0.
            CONTINUE.
          ENDIF.
        ENDIF.
```

(This exists in AD1 hash build — keep it.)

---

## 5. What NOT to change

| Component | Action |
|-----------|--------|
| `ZAPO_NTBK_PROD_DSHP_EXCLUDE` dummy ship exclusion | **No change** |
| `ZAPO_NTBK_PROD_DSHP_EXCLUDE` demand / promo / dummy qty CASE logic | **No change** |
| `ZAPO_NTBK_DMD` | **No change** |

All corrections are in **`ZAPO_NTBK_DMD_DSHP_EXCLUDE`** only.

---

## 6. Expected result after fix

For input index **1837** (`ZVTWEG=10,65`, `PDH1=PPCOPOL.`, `PDH2=ICP.`, RM ETHYLENE, plant 3925):

| Field | Expected |
|-------|----------|
| `NTBK_QTY` | Populated per month (like index 1691 for standard DMD rule) |
| `ZVTWEG` | `10,65` (combined from input) |
| `PDH1` / `PDH2` | `PPCOPOL.` / `ICP.` |
| Formula | `FG_prod × BOM_ratio` per month × DC |

Compare EXDSHP output qty (after dummy-ship exclusion in prod FM) with standard DMD qty — they will differ by design, but **both must produce rows**.

---

## 7. Test checklist (AD1)

| # | Test | Pass criteria |
|---|------|---------------|
| T1 | Index 1837–1839 (EXDSHP + PDH filled) | `ZAPO_NTBK_DETAIL` has rows with non-zero `NTBK_QTY` |
| T2 | Same session index 1691–1692 (standard DMD + PDH) | Still works (regression) |
| T3 | EXDSHP with PDH **initial** | Output still generated (no regression) |
| T4 | PDH2 = RCP. (index 1695 pattern on EXDSHP if configured) | Separate qty from ICP. rows |
| T5 | Dummy ship exclusion | Unchanged — verify prod FM not modified |

---

## 8. Implementation summary

| Priority | Change | Location |
|----------|--------|----------|
| **P1** | Filter `lt_prod_filtered` on **`prodh1`/`prodh2`** not only `pdh1`/`pdh2` | `ZAPO_NTBK_DMD_DSHP_EXCLUDE` — input loop |
| **P1** | Restore **`ntbk_qty = prod_qty × bom_ratio`** | Same FM — collect loops |
| P2 | Normalize ZVTWEG on prod filter | Same FM |
| P2 | Set `lw_output-pdh1/pdh2` from input on output | Same FM |

---

## 9. Related documents

- `ZAPO_NTBK_DMD_DSHP_EXCLUDE_AD1_Analysis_Corrections.md` — ZVTWEG, plant, BOM analysis  
- `ZAPO_NTBK_DMD_DSHP_EXCLUDE_ZVTWEG_Sum_Correction.md` — ZVTWEG summation  
- `ZAPO_NTBK_PROD_DSHP_EXCLUDE.txt` — prod FM reference (sets `prodh1`/`prodh2`)

---

**Author:** mahesh pathak  
**CD:** 8085121 — PDH1/PDH2 fix for EXDSHP rule
