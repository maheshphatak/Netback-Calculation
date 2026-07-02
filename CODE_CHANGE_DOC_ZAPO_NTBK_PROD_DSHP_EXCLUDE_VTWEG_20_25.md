## Code Change Document

### Title
Fix double qty for vtweg 20/25 in `ZAPO_NTBK_PROD_DSHP_EXCLUDE`

### Object
- **FM**: `ZAPO_NTBK_PROD_DSHP_EXCLUDE` (FG `ZAPO_NTBK`)
- **Reference**: `Netback/ZAPO_NTBK_PROD_DSHP_EXCLUDE.txt`
- **Environment**: UAT AT2

---

### Issue
After bucket-wise demand summary changes in `ZAPO_NTBK_DMD_DSHP_EXCLUDE` → `ZAPO_NTBK_PROD_DSHP_EXCLUDE`, input with **vtweg 20** (or **25**) shows **double quantity**.

**Root cause**: vtweg 20/25 fell into `WHEN OTHERS` and used `lv_non_domes_export_qty`, while qty was also included in `lv_demand_qty` total → inflated result. Only vtweg **10** and **15** had dedicated domestic/export logic.

**Business rule**: Only **10** is domestic; **15, 20, 25** are export.

---

### Fix
Treat **20 and 25 like 15** (Export) in two places:

#### 1. Pre-build bucket summary (`lt_demand_by_bucke`)
```abap
IF sy-subrc = 0 AND lw_dcdrv-zvtweg EQ '10'. " Domestic
  lw_demand_by_bucke-domestic_qty = lw_/sapapo/snpopdmn-deliv.
ELSEIF lw_dcdrv-zvtweg EQ '15' OR lw_dcdrv-zvtweg EQ '20' OR lw_dcdrv-zvtweg EQ '25'. " Export
  lw_demand_by_bucke-export_qty = lw_/sapapo/snpopdmn-deliv.
ELSE.
  lw_demand_by_bucke-non_domes_export_qty = lw_/sapapo/snpopdmn-deliv.
ENDIF.
```

#### 2. All 3 `CASE lw_dcdrv-zvtweg` blocks (mat+loc / mat only / loc only)
```abap
WHEN 10.
  " ... existing domestic logic (lv_domestic_qty) ...

WHEN 15.
WHEN 20.
WHEN 25.
  " ... existing export logic (lv_export_qty) ...

WHEN OTHERS.
  " ... unchanged — only true non-dom/non-export vtweg ...
```

`ZAPO_NTBK_DMD_DSHP_EXCLUDE` — **no change** (calls PROD FM only).

---

### Test plan
1. Input **only vtweg 20** → qty matches single production, not 2×.
2. Input **only vtweg 25** → same as 15 behaviour.
3. Input **vtweg 10** → domestic logic unchanged.
4. Input **vtweg 15** → export logic unchanged.
5. Input other vtweg → still uses `WHEN OTHERS`.
