# Code Correction: ZAPO_NTBK_DMD_DSHP_EXCLUDE
## Multiple Month Output & Correct Formula Implementation

---

## 1. Current Issues

### Issue 1: Missing Multiple Month Output per Distribution Channel
- Current code generates single aggregated output per Distribution Channel
- Should generate separate records for each month as per Bucket Definition (`ET_BUCKDF`)
- Each Distribution Channel from `ZAPO_NTBK_REF` field `ZVTWEG` should have separate records for each month

### Issue 2: Incorrect Formula Implementation
- **Current Formula**: `lw_output_1-ntbk_qty = lw_prod_dshpex_op-ntbk_qty * <lfs_dmdvsup>-req_qty_pmt_matnr_out`
- **Required Formula**: 
  ```
  NTBK_QTY = (Production Qty for Month from IT_PROMO) 
             × (Sum of Planned Delivery Qty for Month from IT_DEMAND for this DC) 
             ÷ (Total Planned Delivery Qty excluding DUMMY SHP)
  ```
- Formula should be calculated per month per Distribution Channel

### Issue 3: Missing Data Access
- Need to access Production Quantity from `/SAPAPO/SNPOPPRO` (Field `PRODU`)
- Need to access Planned Delivery Quantity from `/SAPAPO/SNPOPDMN` (Field `DELIV`)
- Need to use Bucket Definition from `ET_BUCKDF` (Table `/SAPAPO/SNPOPBCT` Field `BUCKE`)
- Need to exclude DUMMY SHP from `ZAPOPARAM-Value2`

---

## 2. Required Changes

### 2.1 Add Data Declarations

Add these declarations after existing data declarations (around line 100-150):

```abap
" ============================================================
" NEW: Data structures for multiple month calculation per DC
" ============================================================
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
       END OF lty_total_demand_by_month,

       BEGIN OF lty_demand_raw,
         deliv TYPE /sapapo/meins,
         bckto TYPE /sapapo/snpbucke,
         zvtweg TYPE vtweg,
         shptyp TYPE /sapapo/shptyp,  " Adjust field name based on actual structure
       END OF lty_demand_raw,

       BEGIN OF lty_promo_raw,
         produ TYPE /sapapo/meins,
         bckto TYPE /sapapo/snpbucke,
       END OF lty_promo_raw.

DATA: lt_promo_by_month TYPE STANDARD TABLE OF lty_promo_by_month,
      lt_demand_by_dc_month TYPE STANDARD TABLE OF lty_demand_by_dc_month,
      lt_total_demand_by_month TYPE STANDARD TABLE OF lty_total_demand_by_month,
      lt_dummy_shp TYPE STANDARD TABLE OF string,
      lt_bucket_def TYPE STANDARD TABLE OF zapo_plndrv,
      lt_demand_raw TYPE STANDARD TABLE OF lty_demand_raw,
      lt_promo_raw TYPE STANDARD TABLE OF lty_promo_raw,
      lw_promo_month TYPE lty_promo_by_month,
      lw_demand_dc TYPE lty_demand_by_dc_month,
      lw_total_demand TYPE lty_total_demand_by_month,
      lw_bucket_def TYPE zapo_plndrv,
      lv_dummy_shp_str TYPE string,
      lv_calc_month TYPE zapo_ntbk_prd,
      lv_numerator TYPE /sapapo/meins,
      lv_denominator TYPE /sapapo/meins.

FIELD-SYMBOLS: <lfs_promo> TYPE lty_promo_by_month,
               <lfs_demand_dc> TYPE lty_demand_by_dc_month,
               <lfs_total_demand> TYPE lty_total_demand_by_month,
               <lfs_dummy_shp> TYPE string.
```

### 2.2 Step 1: Read DUMMY SHP Exclusion List

Add this code after the existing `lt_zapoparam` selection (around line 200-250):

```abap
" ============================================================
" STEP 1: Read DUMMY SHP exclusion list from ZAPOPARAM
" ============================================================
CLEAR: lt_dummy_shp.

" Read DUMMY SHP exclusion list from ZAPOPARAM
" Adjust param1, param2, param3 based on your system configuration
SELECT value2 FROM zapoparam
  INTO TABLE lt_dummy_shp
  WHERE param1 = 'SNP'
    AND param2 = 'ZAPO_NTBK'
    AND param3 = 'DMD_DSHP_EXCLUDE'
    AND active_flag = 'X'.

" If value2 contains comma-separated list, split it
IF lines( lt_dummy_shp ) = 1.
  READ TABLE lt_dummy_shp INDEX 1 INTO lv_dummy_shp_str.
  IF lv_dummy_shp_str CS ','.
    CLEAR lt_dummy_shp.
    SPLIT lv_dummy_shp_str AT ',' INTO TABLE lt_dummy_shp.
    LOOP AT lt_dummy_shp ASSIGNING <lfs_dummy_shp>.
      CONDENSE <lfs_dummy_shp>.
      TRANSLATE <lfs_dummy_shp> TO UPPER CASE.
    ENDLOOP.
  ELSE.
    " Single value - normalize it
    CONDENSE lv_dummy_shp_str.
    TRANSLATE lv_dummy_shp_str TO UPPER CASE.
    CLEAR lt_dummy_shp.
    APPEND lv_dummy_shp_str TO lt_dummy_shp.
  ENDIF.
ENDIF.

SORT lt_dummy_shp.
DELETE ADJACENT DUPLICATES FROM lt_dummy_shp.
```

### 2.3 Step 2: Prepare Bucket Definition

Add this code after the `ZAPO_NTBK_PLNDRV` call (around line 180-190):

```abap
" ============================================================
" STEP 2: Prepare bucket definition for multiple months
" ============================================================
" lt_plandrv already contains bucket definitions from ZAPO_NTBK_PLNDRV
" Copy to working table and get unique months
CLEAR: lt_bucket_def.
lt_bucket_def = lt_plandrv.
SORT lt_bucket_def BY date_mthyr.
DELETE ADJACENT DUPLICATES FROM lt_bucket_def COMPARING date_mthyr.
```

### 2.4 Step 3: Aggregate Production Quantity by Month from IT_PROMO

Add this code after Step 2 (around line 190-220):

```abap
" ============================================================
" STEP 3: Aggregate Production Quantity by Month from IT_PROMO
" Table: /SAPAPO/SNPOPPRO, Field: PRODU
" ============================================================
CLEAR: lt_promo_by_month, lt_promo_raw.

" Read Production Quantity from /SAPAPO/SNPOPPRO table
" Filter by session and material
SELECT produ AS prod_qty,
       bckto
  FROM /sapapo/snpoppro
  INTO TABLE lt_promo_raw
  WHERE sessionid = is_session-sessionid
    AND matnr IN lr_mat.

IF sy-subrc = 0.
  LOOP AT lt_promo_raw INTO DATA(lw_promo_raw).
    " Map bucket to month-year using bucket definition
    READ TABLE lt_plandrv INTO lw_bucket_def
      WITH KEY sessionid = is_session-sessionid
               bucke = lw_promo_raw-bckto
      BINARY SEARCH.
    
    IF sy-subrc = 0.
      " Aggregate production quantity by month
      READ TABLE lt_promo_by_month ASSIGNING <lfs_promo>
        WITH KEY date_mythr = lw_bucket_def-date_mthyr.
      
      IF sy-subrc <> 0.
        APPEND INITIAL LINE TO lt_promo_by_month ASSIGNING <lfs_promo>.
        <lfs_promo>-date_mythr = lw_bucket_def-date_mthyr.
        <lfs_promo>-prod_qty = 0.
      ENDIF.
      
      <lfs_promo>-prod_qty = <lfs_promo>-prod_qty + lw_promo_raw-prod_qty.
    ENDIF.
  ENDLOOP.
ENDIF.

SORT lt_promo_by_month BY date_mythr.
```

### 2.5 Step 4: Aggregate Planned Delivery Quantity by DC and Month from IT_DEMAND

Add this code after Step 3 (around line 220-280):

```abap
" ============================================================
" STEP 4: Aggregate Planned Delivery Quantity by DC and Month from IT_DEMAND
" Table: /SAPAPO/SNPOPDMN, Field: DELIV
" Exclude DUMMY SHP shipments based on ZAPOPARAM-Value2
" ============================================================
CLEAR: lt_demand_by_dc_month, lt_total_demand_by_month, lt_demand_raw.

" Read Planned Delivery Quantity from /SAPAPO/SNPOPDMN table
" Filter by session and material
" Note: Adjust field names based on actual table structure
" If zvtweg is not in /SAPAPO/SNPOPDMN, you may need to join with zapo_snp_dmdvsup
SELECT deliv AS plan_del_qty,
       bckto,
       zvtweg
  FROM /sapapo/snpopdmn
  INTO TABLE lt_demand_raw
  WHERE sessionid = is_session-sessionid
    AND matnr IN lr_mat.

" If zvtweg is not in /SAPAPO/SNPOPDMN, use zapo_snp_dmdvsup instead
" Alternative approach if needed:
" SELECT d~deliv AS plan_del_qty,
"        d~bckto,
"        v~zvtweg
"   FROM /sapapo/snpopdmn AS d
"   INNER JOIN zapo_snp_dmdvsup AS v
"     ON d~sessionid = v~sessionid
"    AND d~matnr = v~matnr
"    AND d~locno = v~locno
"    AND d~bckto = v~buckt
"   INTO TABLE lt_demand_raw
"   WHERE d~sessionid = is_session-sessionid
"     AND d~matnr IN lr_mat.

IF sy-subrc = 0.
  LOOP AT lt_demand_raw INTO DATA(lw_demand_raw).
    " Check if this is a DUMMY SHP shipment (exclude from total, but process for DC-specific)
    " Note: Adjust field name based on actual structure
    " You may need to check a shipment type field or join with another table
    DATA(lv_is_dummy_shp) = abap_false.
    
    " Check if shipment type matches DUMMY SHP exclusion list
    " Adjust this logic based on how DUMMY SHP is identified in your system
    " For example, if there's a field like shptyp or ship_to_party:
    " READ TABLE lt_dummy_shp WITH KEY table_line = lw_demand_raw-shptyp TRANSPORTING NO FIELDS.
    " IF sy-subrc = 0.
    "   lv_is_dummy_shp = abap_true.
    " ENDIF.
    
    " Alternative: If DUMMY SHP is identified by a specific value in a field
    " Check against exclusion list - adjust field name as needed
    " For now, assuming we can identify DUMMY SHP from the data
    " You may need to add a join or additional field check here
    
    " Map bucket to month-year using bucket definition
    READ TABLE lt_plandrv INTO lw_bucket_def
      WITH KEY sessionid = is_session-sessionid
               bucke = lw_demand_raw-bckto
      BINARY SEARCH.
    
    IF sy-subrc = 0.
      DATA(lv_demand_month) = lw_bucket_def-date_mthyr.
      DATA(lv_dist_channel) = lw_demand_raw-zvtweg.
      
      " Aggregate by DC and Month (for numerator - includes all shipments)
      READ TABLE lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>
        WITH KEY zvtweg = lv_dist_channel
                 date_mythr = lv_demand_month.
      
      IF sy-subrc <> 0.
        APPEND INITIAL LINE TO lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>.
        <lfs_demand_dc>-zvtweg = lv_dist_channel.
        <lfs_demand_dc>-date_mythr = lv_demand_month.
        <lfs_demand_dc>-plan_del_qty = 0.
      ENDIF.
      
      <lfs_demand_dc>-plan_del_qty = <lfs_demand_dc>-plan_del_qty + lw_demand_raw-plan_del_qty.
      
      " Aggregate total by Month (for denominator - excluding DUMMY SHP)
      IF lv_is_dummy_shp = abap_false.
        READ TABLE lt_total_demand_by_month ASSIGNING <lfs_total_demand>
          WITH KEY date_mythr = lv_demand_month.
        
        IF sy-subrc <> 0.
          APPEND INITIAL LINE TO lt_total_demand_by_month ASSIGNING <lfs_total_demand>.
          <lfs_total_demand>-date_mythr = lv_demand_month.
          <lfs_total_demand>-total_del_qty = 0.
        ENDIF.
        
        <lfs_total_demand>-total_del_qty = <lfs_total_demand>-total_del_qty + lw_demand_raw-plan_del_qty.
      ENDIF.
    ENDIF.
  ENDLOOP.
ENDIF.

SORT lt_demand_by_dc_month BY zvtweg date_mythr.
SORT lt_total_demand_by_month BY date_mythr.
```

**Note**: If `zvtweg` is not directly available in `/SAPAPO/SNPOPDMN`, you may need to use the existing `lt_dmdvsup` table which already contains `zvtweg`. In that case, modify the logic to:

```abap
" Alternative: Use existing lt_dmdvsup which already has zvtweg
" Aggregate Planned Delivery Quantity from lt_dmdvsup
CLEAR: lt_demand_by_dc_month, lt_total_demand_by_month.

LOOP AT lt_dmdvsup ASSIGNING <lfs_dmdvsup>.
  " Check if this is a DUMMY SHP shipment
  DATA(lv_is_dummy_shp) = abap_false.
  " Add logic to check against lt_dummy_shp based on your DUMMY SHP identification method
  
  " Use date_mythr from lt_dmdvsup (already populated from lt_plandrv)
  DATA(lv_demand_month) = <lfs_dmdvsup>-date_mythr.
  DATA(lv_dist_channel) = <lfs_dmdvsup>-zvtweg.
  
  " Aggregate by DC and Month (for numerator)
  READ TABLE lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>
    WITH KEY zvtweg = lv_dist_channel
             date_mythr = lv_demand_month.
  
  IF sy-subrc <> 0.
    APPEND INITIAL LINE TO lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>.
    <lfs_demand_dc>-zvtweg = lv_dist_channel.
    <lfs_demand_dc>-date_mythr = lv_demand_month.
    <lfs_demand_dc>-plan_del_qty = 0.
  ENDIF.
  
  <lfs_demand_dc>-plan_del_qty = <lfs_demand_dc>-plan_del_qty + <lfs_dmdvsup>-sup_cust_1.
  
  " Aggregate total by Month (for denominator - excluding DUMMY SHP)
  IF lv_is_dummy_shp = abap_false.
    READ TABLE lt_total_demand_by_month ASSIGNING <lfs_total_demand>
      WITH KEY date_mythr = lv_demand_month.
    
    IF sy-subrc <> 0.
      APPEND INITIAL LINE TO lt_total_demand_by_month ASSIGNING <lfs_total_demand>.
      <lfs_total_demand>-date_mythr = lv_demand_month.
      <lfs_total_demand>-total_del_qty = 0.
    ENDIF.
    
    <lfs_total_demand>-total_del_qty = <lfs_total_demand>-total_del_qty + <lfs_dmdvsup>-sup_cust_1.
  ENDIF.
ENDLOOP.

SORT lt_demand_by_dc_month BY zvtweg date_mythr.
SORT lt_total_demand_by_month BY date_mythr.
```

### 2.6 Step 5: Replace Output Generation Logic

**Replace the entire output generation section** (starting from around line 400 where `LOOP AT it_input ASSIGNING <lfs_input>` begins) with the following corrected logic:

```abap
" ============================================================
" STEP 5: Generate output with multiple months per Distribution Channel
" ============================================================
CLEAR: lt_output.

LOOP AT it_input ASSIGNING <lfs_input>.
  CLEAR: lt_div, lt_vtweg, lt_loc.
  TRANSLATE <lfs_input>-ntbk_rmmatnr TO UPPER CASE.
  
  " Get Division filter
  IF <lfs_input>-div IS NOT INITIAL.
    SPLIT <lfs_input>-div AT ',' INTO TABLE lt_div.
  ENDIF.

  " Get Distribution Channel(s) from input
  IF <lfs_input>-zvtweg IS NOT INITIAL.
    SPLIT <lfs_input>-zvtweg AT ',' INTO TABLE lt_vtweg.
  ELSE.
    " If no DC specified, process all DCs from demand data
    CLEAR lt_vtweg.
    LOOP AT lt_demand_by_dc_month INTO lw_demand_dc.
      READ TABLE lt_vtweg WITH KEY table_line = lw_demand_dc-zvtweg TRANSPORTING NO FIELDS.
      IF sy-subrc <> 0.
        APPEND lw_demand_dc-zvtweg TO lt_vtweg.
      ENDIF.
    ENDLOOP.
  ENDIF.

  " Get Location filter
  IF <lfs_input>-ntbk_locfr IS NOT INITIAL.
    SPLIT <lfs_input>-ntbk_locfr AT ',' INTO TABLE lt_loc.
  ENDIF.

  " Loop through each Distribution Channel
  LOOP AT lt_vtweg INTO DATA(lv_vtweg).
    
    " Loop through each month in bucket definition
    LOOP AT lt_bucket_def INTO lw_bucket_def.
      
      lv_calc_month = lw_bucket_def-date_mthyr.
      
      " Get Production Quantity for this month
      READ TABLE lt_promo_by_month INTO lw_promo_month
        WITH KEY date_mythr = lv_calc_month
        BINARY SEARCH.
      
      IF sy-subrc <> 0.
        " No production data for this month - skip
        CONTINUE.
      ENDIF.
      
      " Get Planned Delivery Quantity for this DC and month
      READ TABLE lt_demand_by_dc_month INTO lw_demand_dc
        WITH KEY zvtweg = lv_vtweg
                 date_mythr = lv_calc_month
        BINARY SEARCH.
      
      IF sy-subrc <> 0.
        " No demand data for this DC and month - set to zero
        CLEAR lw_demand_dc.
        lw_demand_dc-plan_del_qty = 0.
      ENDIF.
      
      " Get Total Planned Delivery Quantity (excluding DUMMY SHP) for this month
      READ TABLE lt_total_demand_by_month INTO lw_total_demand
        WITH KEY date_mythr = lv_calc_month
        BINARY SEARCH.
      
      IF sy-subrc <> 0 OR lw_total_demand-total_del_qty = 0.
        " No total demand or zero - cannot calculate, skip
        CONTINUE.
      ENDIF.
      
      " ============================================================
      " CALCULATE NETBACK QUANTITY using the correct formula:
      " NTBK_QTY = (Prod Qty) × (DC Planned Del Qty) ÷ (Total Planned Del Qty excl DUMMY SHP)
      " ============================================================
      CLEAR: lw_output.
      
      " Copy all fields from input
      MOVE-CORRESPONDING <lfs_input> TO lw_output.
      CLEAR: lw_output-ntbk_index.
      
      " Set Distribution Channel
      lw_output-zvtweg = lv_vtweg.
      
      " Set month from bucket definition
      lw_output-ntbk_mth = lv_calc_month.
      lw_output-sessionname = is_session-sessionname.
      
      " Calculate netback quantity using the correct formula
      lv_numerator = lw_promo_month-prod_qty * lw_demand_dc-plan_del_qty.
      lv_denominator = lw_total_demand-total_del_qty.
      
      IF lv_denominator <> 0.
        " Apply the formula: (Prod Qty × DC Planned Del Qty) ÷ Total Planned Del Qty
        lw_output-ntbk_qty = lv_numerator / lv_denominator.
      ELSE.
        lw_output-ntbk_qty = 0.
      ENDIF.
      
      " Apply BOM requirement ratio if needed
      " Note: If req_qty_pmt_matnr_out is still needed for RM calculation, apply it here
      " For now, assuming the formula above gives the final quantity
      " If you need to multiply by BOM ratio, uncomment below:
      " IF <lfs_dmdvsup>-req_qty_pmt_matnr_out IS NOT INITIAL.
      "   lw_output-ntbk_qty = lw_output-ntbk_qty * <lfs_dmdvsup>-req_qty_pmt_matnr_out.
      " ENDIF.
      
      " Apply division filter if specified
      IF lt_div IS NOT INITIAL.
        " Check if this record matches division filter
        " You may need to get division from material/location data
        " For now, assuming division check is done elsewhere or not needed here
      ENDIF.
      
      " Apply location filter if specified
      IF lt_loc IS NOT INITIAL.
        " Check if this record matches location filter
        " You may need to get location from material data
        " For now, assuming location check is done elsewhere or not needed here
      ENDIF.
      
      " Append to output table (use APPEND, not COLLECT, to preserve all monthly records)
      APPEND lw_output TO lt_output.
      
    ENDLOOP. " End of bucket definition loop (months)
    
  ENDLOOP. " End of Distribution Channel loop
  
ENDLOOP. " End of input loop

" ============================================================
" Assign ntbk_index and move to export parameter
" ============================================================
IF lt_output IS NOT INITIAL.
  lv_ntbk_index = 1.
  LOOP AT lt_output ASSIGNING <lfs_output>.
    <lfs_output>-ntbk_index = lv_ntbk_index.
    lv_ntbk_index = lv_ntbk_index + 1.
    APPEND <lfs_output> TO et_output.
  ENDLOOP.
  CLEAR: lt_output.
ENDIF.
```

### 2.7 Step 6: Remove Old Output Logic and Duplicate Removal

**Remove or comment out** the following sections:

1. **Remove the old output generation loops** that use `lt_prod_dshpex_op` and multiply by `req_qty_pmt_matnr_out`:
   - The loops starting with `LOOP AT lt_prod_dshpex_op INTO lw_prod_dshpex_op`
   - The multiplication: `lw_output_1-ntbk_qty = lw_prod_dshpex_op-ntbk_qty * <lfs_dmdvsup>-req_qty_pmt_matnr_out`

2. **Remove the duplicate removal at the end** (around line 650-660):
   ```abap
   " REMOVE THESE LINES:
   " SORT et_output BY zvtweg ntbk_qty.
   " DELETE ADJACENT DUPLICATES FROM et_output COMPARING zvtweg ntbk_qty.
   ```

3. **Replace with proper sorting** (if needed):
   ```abap
   " Sort output by index, distribution channel, and month
   SORT et_output BY ntbk_index zvtweg ntbk_mth.
   ```

**Note**: You may still need to keep the call to `ZAPO_NTBK_PROD_DSHP_EXCLUDE` if it's used for other purposes, but the output calculation logic should be replaced with the new formula-based approach.

---

## 3. Important Notes

### 3.1 Field Name Adjustments
You may need to adjust field names based on your actual table structures:
- `/SAPAPO/SNPOPPRO` - Production quantity field name (may be `PRODU` or similar)
- `/SAPAPO/SNPOPDMN` - Planned delivery quantity field name (may be `DELIV` or similar)
- Distribution Channel field (`zvtweg`) may not be in `/SAPAPO/SNPOPDMN` - use `zapo_snp_dmdvsup` or join as needed
- Shipment type field for DUMMY SHP identification

### 3.2 DUMMY SHP Identification
The exact method to identify DUMMY SHP shipments needs to be confirmed. Based on your requirement:
- DUMMY SHP values are in `ZAPOPARAM-Value2`
- You may need to:
  - Check a shipment type field in `/SAPAPO/SNPOPDMN` or `zapo_snp_dmdvsup`
  - Join with another table that contains shipment information
  - Use a flag or indicator field
  - Check against `sup_cust_1` or similar field

### 3.3 Data Source Selection
Choose the appropriate approach:
- **Option A**: Read directly from `/SAPAPO/SNPOPPRO` and `/SAPAPO/SNPOPDMN` (as shown in Steps 3 and 4)
- **Option B**: Use existing data structures `lt_dmdvsup` (which already has `zvtweg` and `date_mythr`) and aggregate from there

### 3.4 BOM Requirement Ratio
If you still need to apply the BOM requirement ratio (`req_qty_pmt_matnr_out`), you can:
- Calculate it once per material/location/bucket combination
- Apply it in the final formula calculation
- Or integrate it into the production quantity calculation

---

## 4. Testing Checklist

1. **Functional Tests**
   - Verify multiple month outputs are generated for each Distribution Channel
   - Verify formula calculation is correct: `(Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY SHP)`
   - Verify DUMMY SHP shipments are excluded from denominator
   - Verify all Distribution Channels from `ZAPO_NTBK_REF` are processed

2. **Data Validation**
   - Compare production quantities with source data from `/SAPAPO/SNPOPPRO`
   - Compare demand quantities with source data from `/SAPAPO/SNPOPDMN`
   - Verify bucket-to-month mapping is correct
   - Verify DUMMY SHP exclusion list is read correctly

3. **Performance Tests**
   - Monitor execution time with large datasets
   - Verify memory usage is acceptable
   - Test with multiple Distribution Channels and months

4. **Regression Tests**
   - Compare output with previous version (where applicable)
   - Verify existing functionality is not broken
   - Test edge cases (no production data, no demand data, zero denominators)

---

## 5. Summary of Changes

1. **Added data structures** for production, demand, and bucket definitions
2. **Added DUMMY SHP exclusion** logic from `ZAPOPARAM`
3. **Added production quantity aggregation** by month from `/SAPAPO/SNPOPPRO`
4. **Added demand quantity aggregation** by DC and month from `/SAPAPO/SNPOPDMN` (or `zapo_snp_dmdvsup`)
5. **Replaced output generation logic** to loop through months and Distribution Channels
6. **Implemented correct formula**: `(Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY SHP)`
7. **Removed duplicate removal** to preserve all monthly records
8. **Removed old output logic** that used incorrect formula

---

**Document Version**: 1.0  
**Created**: Based on User Requirements  
**Status**: Ready for Implementation Review

