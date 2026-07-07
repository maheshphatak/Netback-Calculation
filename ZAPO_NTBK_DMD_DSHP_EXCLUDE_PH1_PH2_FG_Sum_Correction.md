# ZAPO_NTBK_DMD_DSHP_EXCLUDE — PH1/PH2 FG Summation Correction

**System:** AD1 (`SAPLZAPO_NTBK` / function group `ZAPO_NTBK`)  
**Function module:** `ZAPO_NTBK_DMD_DSHP_EXCLUDE`  
**Business rule:** `TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)`  
**Date:** 07-Jul-2026  
**Author:** mahesh pathak — CD 8085121 follow-up (PH hierarchy aggregation)

> **AD1 note:** MCP was unavailable at document creation. Corrections are based on AD1 source verified 29-Jun-2026, workspace `ZAPO_NTBK_PROD_DSHP_EXCLUDE.txt`, and your screenshots (`ZAPO_NTBK_REF` index 1837–1839, `ZAPO_NTBK_DETAIL` output). Re-verify after transport.

---

## 1. Problem (from your screenshots)

### Input (`ZAPO_NTBK_REF` — screenshot 2)

| Index | Rule (EXDSHP) | DIV | DC | RM | FG Material | From Loc | PH1 | PH2 |
|-------|---------------|-----|-----|-----|-------------|----------|-----|-----|
| 1837 | TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP) | 23 | 10,65 | ETHYLENE | **blank** | 3925 | PPCOPOL. | ICP. |
| 1838 | same | 23 | 20,25 | ETHYLENE | **blank** | 3925 | PPCOPOL. | ICP. |
| 1839 | same | 23 | 15 | ETHYLENE | **blank** | 3925 | PPCOPOL. | ICP. |

FG Material is **not** maintained. PH1 and PH2 define which finished goods belong to the rule.

### Current output (`ZAPO_NTBK_DETAIL` — screenshot 1)

For index 1837 / month **202608** / DC **10,65** / ETHYLENE:

| FG Material | NTBK_QTY (current) |
|-------------|-------------------|
| B120MA | 72.716 |
| C080MA | 130.030 |

**Two separate rows** — one per FG at plant 3925.

### Expected output

When PH1 and PH2 are maintained and FG Material is blank:

1. **Sum RM qty** across all FG materials that fall under PH1 + PH2 (+ division + plant + RM BOM).
2. **One row** per planning month × input index × combined DC string (same ZVTWEG summation pattern as `10,65`).
3. Output row must still show **which FG codes contributed** (comma-separated in `NTBK_MATNR`, e.g. `B120MA,C080MA`).

| Planning month | DC (input) | Expected NTBK_QTY | Contributing FGs |
|----------------|------------|-------------------|------------------|
| 202608 | 10,65 | **202.746** | B120MA + C080MA (72.716 + 130.030) |
| 202609 | 20,25 | **192.720** *(verify after BOM)* | Sum of all PPCOPOL./ICP. FGs at 3925 |

---

## 2. Root cause

### 2.1 Per-FG aggregation key in DMD FM

`ZAPO_NTBK_DMD_DSHP_EXCLUDE` calls `ZAPO_NTBK_PROD_DSHP_EXCLUDE`, which returns **one prod row per FG** when input FG is blank (plant-only path — `ntbk_matnr IS INITIAL`, `ntbk_locfr` set).

The DMD FM then aggregates via `sum_prod_dshp_to_agg` / `COLLECT` on `lt_output_agg` where **`ntbk_matnr` is part of the COLLECT key** on structure `ZAPO_NTBK_DETAIL`:

```abap
" Current pattern — separate row per FG
LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup> ...
  PERFORM sum_prod_dshp_to_agg USING
    <lfs_dmdvsup>-matnr    " ← one FG per iteration
    ...
  CHANGING lt_output_agg lt_prod_dshpex_op.
ENDLOOP.

" Inside sum_prod_dshp_to_agg:
LOOP AT it_prod_dshpex INTO lw_prod
  WHERE ntbk_matnr = iv_matnr ...   " ← filters single FG
COLLECT lw_agg INTO ct_output_agg.  " ← ntbk_matnr stays in key → no cross-FG sum
```

Result: **one `ZAPO_NTBK_DETAIL` line per FG** even when input has only PH1/PH2.

### 2.2 No PH hierarchy filter on FG master

`ZAPO_NTBK_PROD_DSHP_EXCLUDE` loops **all** SNP production at plant 3925 when FG is blank. It copies PH1/PH2 from the **rule ref** onto every prod row (`prodh1` / `prodh2`), but does **not** restrict to FGs whose master PDH matches `ZPRODHMST`.

DMD FM must either:

- filter prod rows using **`ZPRODHMST`** (key `SPART` + `MATNR`), or  
- rely on prod rows only after explicit PH-based FG list is built.

### 2.3 Related gaps (apply together)

| Gap | Effect when PH1/PH2 filled |
|-----|---------------------------|
| PDH filter uses `pdh1`/`pdh2` instead of `prodh1`/`prodh2` | No output at all (see `ZAPO_NTBK_DMD_DSHP_EXCLUDE_PDH1_PDH2_Correction.md`) |
| BOM ratio not applied | FG prod qty instead of RM req qty |
| `zvtweg` in COLLECT key | Separate rows per DC instead of combined `10,65` |

Your screenshot shows qty **with** separate FG rows — so prod + partial DMD path is running; the **missing behaviour is cross-FG summation** for PH-driven rules.

---

## 3. Business rule (target behaviour)

```
IF ZAPO_NTBK_REF-NTBK_MATNR is INITIAL
AND (PDH1 is NOT INITIAL OR PDH2 is NOT INITIAL)
AND Rule = TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)

THEN for each planning month:
  NTBK_QTY = SUM over all FG where:
    ZPRODHMST-SPART = input DIV
    ZPRODHMST-MATNR = FG code
    ZPRODHMST-PDH1  = input PDH1   (if input PDH1 maintained)
    ZPRODHMST-PDH2  = input PDH2   (if input PDH2 maintained)
    FG prod at input NTBK_LOCFR
    FG prod DC in input ZVTWEG list
    RM BOM ratio applied per FG

  Output: ONE row per month × index
  NTBK_MATNR = comma-separated list of contributing FG codes
  ZVTWEG     = combined input value (e.g. 10,65)
  PDH1/PDH2  = from input
```

When **FG Material is maintained** on input → keep **existing per-FG** behaviour (no cross-FG sum).

---

## 4. Scope

| In scope | Out of scope |
|----------|--------------|
| `ZAPO_NTBK_DMD_DSHP_EXCLUDE` only | `ZAPO_NTBK_PROD_DSHP_EXCLUDE` dummy-ship exclusion logic |
| PH1/PH2 + blank FG → summed output | Changes to `ZAPO_NTBK_REF` table structure |
| ZPRODHMST read for FG validation | `ZAPO_NTBK_DMD` (non-EXDSHP rule) unless same pattern requested |

---

## 5. Code correction — `ZAPO_NTBK_DMD_DSHP_EXCLUDE`

### 5.1 Local types and data (top of FM)

```abap
TYPES: BEGIN OF lty_zprodhmst,
         spart TYPE spart,
         matnr TYPE matnr,
         pdh1  TYPE zapo_pdh1,
         pdh2  TYPE zapo_pdh2,
       END OF lty_zprodhmst.

DATA: lt_zprodhmst     TYPE STANDARD TABLE OF lty_zprodhmst,
      lw_zprodhmst     TYPE lty_zprodhmst,
      lt_fg_under_pdh  TYPE SORTED TABLE OF matnr WITH UNIQUE KEY table_line,
      lv_fg_list       TYPE string,
      lv_sum_rm_qty    TYPE zapo_ntbk_qty,
      lv_bom_ratio     TYPE zapo_req_inp_qty,
      lv_rm_line_qty   TYPE zapo_ntbk_qty,
      lv_bucket        TYPE zapo_bucket,
      lv_pdh_sum_mode  TYPE abap_bool.   " cross-FG sum when PH maintained, FG blank
```

### 5.2 Load `ZPRODHMST` once per FM call (after prod FM returns)

Build FG list from all prod output matnrs + input divisions:

```abap
REFRESH lt_zprodhmst.
IF lt_prod_dshpex_op IS NOT INITIAL.
  DATA(lt_fg_distinct) = VALUE matnr_t( ).
  LOOP AT lt_prod_dshpex_op INTO DATA(ls_prod_fg).
    IF ls_prod_fg-ntbk_matnr IS NOT INITIAL.
      APPEND ls_prod_fg-ntbk_matnr TO lt_fg_distinct.
    ENDIF.
  ENDLOOP.
  SORT lt_fg_distinct.
  DELETE ADJACENT DUPLICATES FROM lt_fg_distinct.

  IF lt_fg_distinct IS NOT INITIAL.
    SELECT spart matnr pdh1 pdh2
      FROM zprodhmst
      INTO TABLE lt_zprodhmst
      FOR ALL ENTRIES IN lt_fg_distinct
      WHERE matnr = lt_fg_distinct-table_line.
    IF sy-subrc = 0.
      SORT lt_zprodhmst BY spart matnr.
    ENDIF.
  ENDIF.
ENDIF.
```

> If `ZPRODHMST` is on ECC only, use existing RFC pattern from `ZAPO_MATERIAL_BALANCE_CLS` (`fetch_ecc_prod_hrrchy_master`) instead of direct `SELECT`.

### 5.3 FORM — check FG belongs to input PH

```abap
FORM fg_matches_input_pdh
  USING
    iv_fg_matnr    TYPE /sapapo/matnr
    iv_div         TYPE spart
    iv_pdh1        TYPE zapo_pdh1
    iv_pdh2        TYPE zapo_pdh2
  CHANGING
    cv_ok          TYPE abap_bool.

  DATA lv_div TYPE spart.
  CLEAR cv_ok.

  READ TABLE lt_zprodhmst INTO lw_zprodhmst
    WITH KEY spart = iv_div matnr = iv_fg_matnr BINARY SEARCH.
  IF sy-subrc NE 0.
    " No master row — skip FG when PH filter is active
    IF iv_pdh1 IS NOT INITIAL OR iv_pdh2 IS NOT INITIAL.
      RETURN.
    ENDIF.
    cv_ok = abap_true.
    RETURN.
  ENDIF.

  IF iv_pdh1 IS NOT INITIAL AND lw_zprodhmst-pdh1 NE iv_pdh1.
    RETURN.
  ENDIF.
  IF iv_pdh2 IS NOT INITIAL AND lw_zprodhmst-pdh2 NE iv_pdh2.
    RETURN.
  ENDIF.

  cv_ok = abap_true.
ENDFORM.
```

### 5.4 FORM — sum prod qty across PH-matching FGs (new)

Replace per-FG `sum_prod_dshp_to_agg` calls when `lv_pdh_sum_mode = abap_true`.

```abap
FORM sum_prod_dshp_pdh_hier_to_agg
  USING
    iv_locfr         TYPE /sapapo/locno
    iv_mth           TYPE zapo_ntbk_prd
    iv_pdh1          TYPE zapo_pdh1
    iv_pdh2          TYPE zapo_pdh2
    iv_div           TYPE spart
    iv_rm_matnr      TYPE zapo_matnr_out
    iv_sessionname   TYPE zapo_sessionname
    iw_output        TYPE zapo_ntbk_detail
  CHANGING
    ct_output_agg    TYPE zapo_ntbk_detail_t
    ct_prod_dshpex   TYPE zapo_ntbk_detail_t
    ct_log_bom       TYPE STANDARD TABLE.

  DATA: lw_prod     TYPE zapo_ntbk_detail,
        lw_agg      TYPE zapo_ntbk_detail,
        lv_fg_ok    TYPE abap_bool,
        lv_fg_list  TYPE string,
        lv_sum_qty  TYPE zapo_ntbk_qty,
        lv_bom_rat  TYPE zapo_req_inp_qty,
        lv_rm_qty   TYPE zapo_ntbk_qty,
        lv_bucket   TYPE zapo_bucket,
        lt_fg_seen  TYPE SORTED TABLE OF matnr WITH UNIQUE KEY table_line.

  CLEAR: lv_fg_list, lv_sum_qty.

  LOOP AT ct_prod_dshpex INTO lw_prod.

    " Month
    IF iv_mth IS NOT INITIAL AND lw_prod-ntbk_mth <> iv_mth.
      CONTINUE.
    ENDIF.

    " Plant — use plant_matches FORM if internal loc ID (see AD1_Analysis_Corrections.md §3)
    IF iv_locfr IS NOT INITIAL AND lw_prod-ntbk_locfr <> iv_locfr.
      " CALL plant_matches ... IF sy-subrc ne match: CONTINUE.
      CONTINUE.
    ENDIF.

    " PDH on prod row (prodh1/prodh2 from PROD FM)
    IF iv_pdh1 IS NOT INITIAL.
      IF lw_prod-prodh1 IS NOT INITIAL AND lw_prod-prodh1 <> iv_pdh1.
        CONTINUE.
      ENDIF.
      IF lw_prod-prodh1 IS INITIAL AND lw_prod-pdh1 IS NOT INITIAL
         AND lw_prod-pdh1 <> iv_pdh1.
        CONTINUE.
      ENDIF.
    ENDIF.
    IF iv_pdh2 IS NOT INITIAL.
      IF lw_prod-prodh2 IS NOT INITIAL AND lw_prod-prodh2 <> iv_pdh2.
        CONTINUE.
      ENDIF.
      IF lw_prod-prodh2 IS INITIAL AND lw_prod-pdh2 IS NOT INITIAL
         AND lw_prod-pdh2 <> iv_pdh2.
        CONTINUE.
      ENDIF.
    ENDIF.

    " ZVTWEG — must be in input split list (lt_vtweg_h)
    IF lt_vtweg_h IS NOT INITIAL.
      DATA(lv_vtweg) = lw_prod-zvtweg.
      CONDENSE lv_vtweg NO-GAPS.
      READ TABLE lt_vtweg_h WITH TABLE KEY vtweg = lv_vtweg TRANSPORTING NO FIELDS.
      IF sy-subrc NE 0. CONTINUE. ENDIF.
    ENDIF.

    IF lw_prod-ntbk_matnr IS INITIAL.
      CONTINUE.
    ENDIF.

    " ZPRODHMST hierarchy check
    PERFORM fg_matches_input_pdh USING
      lw_prod-ntbk_matnr iv_div iv_pdh1 iv_pdh2
    CHANGING lv_fg_ok.
    IF lv_fg_ok NE abap_true.
      CONTINUE.
    ENDIF.

    " BOM ratio per FG
    CLEAR lv_bom_rat.
    PERFORM get_bucket_from_month USING
      is_session-sessionid lw_prod-ntbk_mth
    CHANGING lv_bucket lt_plandrv.

    PERFORM get_bom_rm_ratio USING
      iv_sessionname lw_prod-ntbk_matnr lw_prod-ntbk_locfr
      lv_bucket iv_rm_matnr
    CHANGING lv_bom_rat ct_log_bom.

    IF lv_bom_rat IS INITIAL.
      CONTINUE.
    ENDIF.

    lv_rm_qty = lw_prod-ntbk_qty * lv_bom_rat.
    lv_sum_qty = lv_sum_qty + lv_rm_qty.

    READ TABLE lt_fg_seen WITH TABLE KEY table_line = lw_prod-ntbk_matnr TRANSPORTING NO FIELDS.
    IF sy-subrc NE 0.
      INSERT lw_prod-ntbk_matnr INTO TABLE lt_fg_seen.
      IF lv_fg_list IS INITIAL.
        lv_fg_list = lw_prod-ntbk_matnr.
      ELSE.
        lv_fg_list = |{ lv_fg_list },{ lw_prod-ntbk_matnr }|.
      ENDIF.
    ENDIF.

  ENDLOOP.

  IF lv_sum_qty IS INITIAL.
    RETURN.
  ENDIF.

  lw_agg = iw_output.
  CLEAR: lw_agg-zvtweg, lw_agg-ntbk_qty.
  lw_agg-ntbk_qty  = lv_sum_qty.
  lw_agg-ntbk_matnr = lv_fg_list.    " B120MA,C080MA — satisfies "output with FG Code"
  lw_agg-pdh1 = iv_pdh1.
  lw_agg-pdh2 = iv_pdh2.
  COLLECT lw_agg INTO ct_output_agg.  " ntbk_matnr same for all FGs in one month → one row

ENDFORM.
```

### 5.5 Input loop — branch on PH sum mode

At start of each `LOOP AT it_input ASSIGNING <lfs_input>`:

```abap
CLEAR lv_pdh_sum_mode.
IF <lfs_input>-ntbk_matnr IS INITIAL
AND ( <lfs_input>-pdh1 IS NOT INITIAL OR <lfs_input>-pdh2 IS NOT INITIAL ).
  lv_pdh_sum_mode = abap_true.
ENDIF.
```

**When `lv_pdh_sum_mode = abap_true`:**

1. Build `lt_prod_filtered` using **`prodh1`/`prodh2`** filter (not `pdh1`/`pdh2`) — see `PDH1_PDH2_Correction.md` §4.1.
2. Do **not** loop `lt_dmdvsup` per FG with `sum_prod_dshp_to_agg`.
3. Instead, loop distinct **months** from `lt_prod_filtered` (or `lt_plandrv`):

```abap
IF lv_pdh_sum_mode = abap_true.

  CLEAR lt_output_agg.
  LOOP AT lt_plandrv INTO DATA(ls_pln)
    WHERE sessionid = is_session-sessionid.

    CLEAR lw_output.
    MOVE-CORRESPONDING <lfs_input> TO lw_output.
    CLEAR lw_output-ntbk_index.
    lw_output-sessionname = is_session-sessionname.
    lw_output-ntbk_mth    = ls_pln-date_mthyr.

    PERFORM sum_prod_dshp_pdh_hier_to_agg USING
      <lfs_input>-ntbk_locfr
      ls_pln-date_mthyr
      <lfs_input>-pdh1
      <lfs_input>-pdh2
      <lfs_input>-div
      <lfs_input>-ntbk_rmmatnr
      is_session-sessionname
      lw_output
    CHANGING lt_output_agg lt_prod_filtered lt_log_bom_2.

  ENDLOOP.

  PERFORM flush_output_agg USING <lfs_input>-zvtweg
    CHANGING lt_output_agg lt_output.

ENDIF.
```

**When `lv_pdh_sum_mode = abap_false`:** keep existing per-FG `sum_prod_dshp_to_agg` / COMB_PREPARE logic unchanged.

### 5.6 Fix `lt_prod_filtered` PDH filter (mandatory prerequisite)

```abap
CLEAR lt_prod_filtered.
LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op.

  IF <lfs_input>-pdh1 IS NOT INITIAL.
    IF lw_prod_dshpex_op-prodh1 IS NOT INITIAL.
      IF lw_prod_dshpex_op-prodh1 <> <lfs_input>-pdh1. CONTINUE. ENDIF.
    ELSEIF lw_prod_dshpex_op-pdh1 <> <lfs_input>-pdh1.
      CONTINUE.
    ENDIF.
  ENDIF.

  IF <lfs_input>-pdh2 IS NOT INITIAL.
    IF lw_prod_dshpex_op-prodh2 IS NOT INITIAL.
      IF lw_prod_dshpex_op-prodh2 <> <lfs_input>-pdh2. CONTINUE. ENDIF.
    ELSEIF lw_prod_dshpex_op-pdh2 <> <lfs_input>-pdh2.
      CONTINUE.
    ENDIF.
  ENDIF.

  " ZVTWEG normalize + filter (existing ZVTWEG correction)
  ...
  APPEND lw_prod_dshpex_op TO lt_prod_filtered.
ENDLOOP.
```

### 5.7 Prevent double processing

Wrap RM-only block and COMB_PREPARE block:

```abap
IF lv_pdh_sum_mode = abap_true.
  " PH hierarchy sum path only (§5.5)
ELSEIF <lfs_input>-ntbk_matnr IS INITIAL AND ... RM-only conditions ...
  " existing RM-only path
ELSE.
  CALL FUNCTION 'ZAPO_NTBK_COMB_PREPARE' ...
  " existing COMB loops with per-FG sum_prod_dshp_to_agg
ENDIF.
```

---

## 6. Expected result after transport

### Index 1837 — ETHYLENE / 3925 / PPCOPOL. / ICP. / DC 10,65

| NTBK_MTH | ZVTWEG | NTBK_RMMATNR | NTBK_MATNR (FG list) | NTBK_QTY |
|----------|--------|--------------|----------------------|----------|
| 202608 | 10,65 | ETHYLENE | B120MA,C080MA | **202.746** |
| 202609 | 10,65 | ETHYLENE | *(FGs under PH)* | *(sum)* |
| 202610 | 10,65 | ETHYLENE | *(FGs under PH)* | *(sum)* |

**One row per month** — not separate rows for B120MA and C080MA.

### Index 1838 — DC 20,25

| NTBK_MTH | ZVTWEG | NTBK_QTY (expected) |
|----------|--------|---------------------|
| 202609 | 20,25 | **192.720** (sum of PPCOPOL./ICP. FGs; verify after BOM) |

---

## 7. Test plan (AD1)

| # | Test | Expected |
|---|------|----------|
| 1 | Index 1837, month 202608 | Single row, `NTBK_QTY = 202.746`, `NTBK_MATNR` contains `B120MA` and `C080MA` |
| 2 | Index 1838, month 202609 | Single row, `ZVTWEG = 20,25`, summed qty (not per-FG rows) |
| 3 | Same rule with **FG Material maintained** (e.g. B120MA only) | **One row** for that FG only — no cross-FG sum |
| 4 | PH1/PH2 **blank**, FG blank | Existing RM-only path — unchanged |
| 5 | PH1/PH2 filled, FG **outside** hierarchy in `ZPRODHMST` | FG excluded from sum |
| 6 | `ZVTWEG = 20` only (not 20,25) | Qty includes DC 20 only |
| 7 | Compare BOM | `NTBK_QTY = FG_prod × (req_prnt_qty / pds_out_qty)` per FG before sum |

---

## 8. Transport checklist

| Priority | Change | FM |
|----------|--------|-----|
| **P1** | `lv_pdh_sum_mode` branch + `sum_prod_dshp_pdh_hier_to_agg` | `ZAPO_NTBK_DMD_DSHP_EXCLUDE` |
| **P1** | `lt_prod_filtered` on `prodh1`/`prodh2` | same |
| **P1** | `ZPRODHMST` read + `fg_matches_input_pdh` | same |
| **P1** | BOM multiply in sum FORM | same |
| P2 | `IF/ELSE` — no RM-only + COMB double append | same |
| P2 | ZVTWEG normalize + combined DC flush | same (see ZVTWEG_Sum_Correction.md) |

**Do not change:** dummy-customer / ship-exclude logic in `ZAPO_NTBK_PROD_DSHP_EXCLUDE`.

---

## 9. Related documents

| File | Topic |
|------|-------|
| `ZAPO_NTBK_DMD_DSHP_EXCLUDE_PDH1_PDH2_Correction.md` | PDH field mismatch (`pdh1` vs `prodh1`) |
| `ZAPO_NTBK_DMD_DSHP_EXCLUDE_ZVTWEG_Sum_Correction.md` | Combined DC summation (`10,65`) |
| `ZAPO_NTBK_DMD_DSHP_EXCLUDE_AD1_Analysis_Corrections.md` | BOM ratio, multi-plant, DC double-count |

---

**CD:** 8085121 — PH1/PH2 FG hierarchy summation for EXDSHP rule  
**Path:** `Netback Calculation\ZAPO_NTBK_DMD_DSHP_EXCLUDE_PH1_PH2_FG_Sum_Correction.md`
