# Implementation Summary: ZAPO_NTBK_DMD_DSHP_EXCLUDE Corrections

## Quick Reference: Key Code Changes

### 1. Add Data Declarations (After existing declarations, ~line 100-150)

```abap
TYPES: BEGIN OF lty_promo_by_month,
         date_mythr TYPE zapo_ntbk_prd,
         prod_qty   TYPE /sapapo/meins,
       END OF lty_promo_by_month,
       BEGIN OF lty_demand_by_dc_month,
         zvtweg     TYPE vtweg,
         date_mythr TYPE zapo_ntbk_prd,
         plan_del_qty TYPE /sapapo/meins,
       END OF lty_demand_by_dc_month,
       BEGIN OF lty_total_demand_by_month,
         date_mythr   TYPE zapo_ntbk_prd,
         total_del_qty TYPE /sapapo/meins,
       END OF lty_total_demand_by_month.

DATA: lt_promo_by_month TYPE STANDARD TABLE OF lty_promo_by_month,
      lt_demand_by_dc_month TYPE STANDARD TABLE OF lty_demand_by_dc_month,
      lt_total_demand_by_month TYPE STANDARD TABLE OF lty_total_demand_by_month,
      lt_dummy_shp TYPE STANDARD TABLE OF string,
      lt_bucket_def TYPE STANDARD TABLE OF zapo_plndrv.
```

### 2. Read DUMMY SHP Exclusion List (After lt_zapoparam selection, ~line 200-250)

```abap
SELECT value2 FROM zapoparam
  INTO TABLE lt_dummy_shp
  WHERE param1 = 'SNP'
    AND param2 = 'ZAPO_NTBK'
    AND param3 = 'DMD_DSHP_EXCLUDE'
    AND active_flag = 'X'.
" Split comma-separated values if needed
```

### 3. Prepare Bucket Definition (After ZAPO_NTBK_PLNDRV call, ~line 180-190)

```abap
lt_bucket_def = lt_plandrv.
SORT lt_bucket_def BY date_mthyr.
DELETE ADJACENT DUPLICATES FROM lt_bucket_def COMPARING date_mthyr.
```

### 4. Aggregate Production Quantity by Month (~line 190-220)

```abap
SELECT produ AS prod_qty, bckto
  FROM /sapapo/snpoppro
  INTO TABLE lt_promo_raw
  WHERE sessionid = is_session-sessionid
    AND matnr IN lr_mat.

" Map buckets to months and aggregate by month
" Use lt_plandrv for bucket-to-month mapping
```

### 5. Aggregate Demand Quantity by DC and Month (~line 220-280)

**Option A: Direct from /SAPAPO/SNPOPDMN** (if zvtweg is available):
```abap
SELECT deliv AS plan_del_qty, bckto, zvtweg
  FROM /sapapo/snpopdmn
  INTO TABLE lt_demand_raw
  WHERE sessionid = is_session-sessionid
    AND matnr IN lr_mat.
```

**Option B: Use existing lt_dmdvsup** (recommended if zvtweg not in /SAPAPO/SNPOPDMN):
```abap
" Use lt_dmdvsup which already has zvtweg and date_mythr
" Aggregate sup_cust_1 by zvtweg and date_mythr
" Exclude DUMMY SHP from total calculation
```

### 6. Replace Output Generation Logic (Replace entire section from ~line 400)

**Key Changes:**
- Loop through each Distribution Channel from `lt_vtweg`
- Loop through each month from `lt_bucket_def`
- Calculate: `NTBK_QTY = (Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY SHP)`
- Use `APPEND` instead of `COLLECT` to preserve all monthly records

```abap
LOOP AT it_input ASSIGNING <lfs_input>.
  " Get Distribution Channels
  IF <lfs_input>-zvtweg IS NOT INITIAL.
    SPLIT <lfs_input>-zvtweg AT ',' INTO TABLE lt_vtweg.
  ENDIF.

  " Loop through each DC
  LOOP AT lt_vtweg INTO lv_vtweg.
    " Loop through each month
    LOOP AT lt_bucket_def INTO lw_bucket_def.
      " Get Production Qty for month
      " Get DC Demand Qty for month
      " Get Total Demand Qty (excl DUMMY SHP) for month
      " Calculate: NTBK_QTY = (Prod × DC Del) ÷ Total Del
      " Append to output
    ENDLOOP.
  ENDLOOP.
ENDLOOP.
```

### 7. Remove Old Logic

**Remove:**
- Old loops using `lt_prod_dshpex_op` and `req_qty_pmt_matnr_out`
- Duplicate removal: `DELETE ADJACENT DUPLICATES FROM et_output COMPARING zvtweg ntbk_qty`

**Replace with:**
```abap
SORT et_output BY ntbk_index zvtweg ntbk_mth.
```

---

## Formula Implementation

**Required Formula:**
```
NTBK_QTY = (Production Qty for Month from IT_PROMO) 
           × (Sum of Planned Delivery Qty for Month from IT_DEMAND for this DC) 
           ÷ (Total Planned Delivery Qty excluding DUMMY SHP)
```

**Implementation:**
```abap
lv_numerator = lw_promo_month-prod_qty * lw_demand_dc-plan_del_qty.
lv_denominator = lw_total_demand-total_del_qty.

IF lv_denominator <> 0.
  lw_output-ntbk_qty = lv_numerator / lv_denominator.
ELSE.
  lw_output-ntbk_qty = 0.
ENDIF.
```

---

## Critical Points to Verify

1. **Field Names**: Adjust based on actual table structures
   - `/SAPAPO/SNPOPPRO`: `PRODU` field
   - `/SAPAPO/SNPOPDMN`: `DELIV` field, `zvtweg` availability
   
2. **DUMMY SHP Identification**: Confirm how to identify DUMMY SHP shipments
   - Check against `ZAPOPARAM-Value2`
   - May need to check shipment type field or join with other table

3. **Distribution Channel Source**: 
   - If `zvtweg` not in `/SAPAPO/SNPOPDMN`, use `lt_dmdvsup` which already has it

4. **BOM Requirement Ratio**: 
   - If still needed, apply `req_qty_pmt_matnr_out` after formula calculation

---

## Testing Priorities

1. ✅ Multiple months generated per Distribution Channel
2. ✅ Formula calculation matches requirement
3. ✅ DUMMY SHP excluded from denominator
4. ✅ All Distribution Channels from input processed
5. ✅ No duplicate removal of monthly records

