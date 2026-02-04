# Code Correction: ZAPO_NTBK_PROD_DSHP_EXCLUDE
## Multiple Month Output & Correct Formula Implementation

---

## 1. Current Issues Identified

### Issue 1: Missing Multiple Month Output per Distribution Channel
- Current code generates aggregated output per Distribution Channel without proper month-wise breakdown
- Should generate separate records for each month as per Bucket Definition (`ET_BUCKDF` from `/SAPAPO/SNPOPBCT`)
- Each Distribution Channel from `ZAPO_NTBK_REF` field `ZVTWEG` should have separate records for each month

### Issue 2: Incorrect Formula Implementation
- **Current Formula**: The code calculates `lw_output-ntbk_qty` but doesn't properly apply the required formula per month per DC
- **Required Formula**: 
  ```
  NTBK_QTY = (Production Qty for Month from IT_PROMO) 
             × (Sum of Planned Delivery Qty for Month from IT_DEMAND for this DC) 
             ÷ (Total Planned Delivery Qty excluding DUMMY SHP)
  ```
- Formula should be calculated per month per Distribution Channel as per bucket definition

### Issue 3: Missing Month-wise Breakdown
- The code reads production and demand data but aggregates across months
- Need to break down calculations by month using bucket definitions from `lt_plandrv`
- Need to ensure each DC gets separate records for each month

---

## 2. Required Changes

### 2.1 Data Structure Additions

Add these data declarations after existing declarations (around line 100-150):

```abap
" ============================================================
" NEW: Data structures for multiple month calculation per DC
" ============================================================
TYPES: BEGIN OF lty_prod_by_month_dc,
         date_mthyr TYPE zapo_ntbk_prd,
         zvtweg     TYPE vtweg,
         prod_qty   TYPE /sapapo/snpprodu,
         bucke      TYPE /sapapo/snpbucke,
       END OF lty_prod_by_month_dc,

       BEGIN OF lty_demand_by_dc_month,
         zvtweg     TYPE vtweg,
         date_mthyr TYPE zapo_ntbk_prd,
         plan_del_qty TYPE /sapapo/snpdeliv,
         bucke      TYPE /sapapo/snpbucke,
       END OF lty_demand_by_dc_month,

       BEGIN OF lty_total_demand_by_month,
         date_mthyr   TYPE zapo_ntbk_prd,
         total_del_qty TYPE /sapapo/snpdeliv,
         bucke        TYPE /sapapo/snpbucke,
       END OF lty_total_demand_by_month.

DATA: lt_prod_by_month_dc TYPE STANDARD TABLE OF lty_prod_by_month_dc,
      lt_demand_by_dc_month TYPE STANDARD TABLE OF lty_demand_by_dc_month,
      lt_total_demand_by_month TYPE STANDARD TABLE OF lty_total_demand_by_month,
      lt_vtweg_list TYPE STANDARD TABLE OF vtweg,
      lw_prod_month_dc TYPE lty_prod_by_month_dc,
      lw_demand_dc TYPE lty_demand_by_dc_month,
      lw_total_demand TYPE lty_total_demand_by_month,
      lv_calc_month TYPE zapo_ntbk_prd,
      lv_numerator TYPE /sapapo/snpdeliv,
      lv_denominator TYPE /sapapo/snpdeliv,
      lv_vtweg TYPE vtweg.

FIELD-SYMBOLS: <lfs_prod_month> TYPE lty_prod_by_month_dc,
               <lfs_demand_dc> TYPE lty_demand_by_dc_month,
               <lfs_total_demand> TYPE lty_total_demand_by_month,
               <lfs_vtweg> TYPE vtweg.
```

### 2.2 Step 1: Aggregate Production Quantity by Month and DC

Replace the existing production aggregation logic (around line 300-400) with:

```abap
" ============================================================
" STEP 1: Aggregate Production Quantity by Month and DC
" Source: /SAPAPO/SNPOPPRO (Field PRODU)
" Map buckets to months using lt_plandrv
" ============================================================
CLEAR: lt_prod_by_month_dc.

" Loop through production data (lt_snpopbct_con) which has production quantities
LOOP AT lt_snpopbct_con ASSIGNING <lfs_snpopbct_con>.
  " Find corresponding bucket definition to get month
  READ TABLE lt_plandrv INTO lw_snpoplpr
    WITH KEY sessionid = <lfs_snpopbct_con>-sessionid
             bucke = <lfs_snpopbct_con>-bucke
    BINARY SEARCH.
  
  IF sy-subrc = 0.
    " Get Distribution Channel from input reference
    " Note: We need to get DC from ZAPO_NTBK_REF for this material/location
    " For now, we'll aggregate by month and bucket, then split by DC in output generation
    
    " Aggregate production quantity by month and bucket
    READ TABLE lt_prod_by_month_dc ASSIGNING <lfs_prod_month>
      WITH KEY date_mthyr = lw_snpoplpr-date_mthyr
               bucke = <lfs_snpopbct_con>-bucke.
    
    IF sy-subrc <> 0.
      APPEND INITIAL LINE TO lt_prod_by_month_dc ASSIGNING <lfs_prod_month>.
      <lfs_prod_month>-date_mthyr = lw_snpoplpr-date_mthyr.
      <lfs_prod_month>-bucke = <lfs_snpopbct_con>-bucke.
      <lfs_prod_month>-prod_qty = 0.
    ENDIF.
    
    " Calculate production quantity: outin * produ
    " Note: outin comes from lt_snpoppma, produ from lt_snpoppro
    " This calculation should match the existing logic
    READ TABLE lt_snpoppma ASSIGNING <lfs_snpoppma>
      WITH KEY sessionid = <lfs_snpopbct_con>-sessionid
               proid = <lfs_snpopbct_con>-proid
      BINARY SEARCH.
    
    IF sy-subrc = 0.
      <lfs_prod_month>-prod_qty = <lfs_prod_month>-prod_qty + 
                                   ( <lfs_snpoppma>-outin * <lfs_snpopbct_con>-produ ).
    ENDIF.
  ENDIF.
ENDLOOP.

SORT lt_prod_by_month_dc BY date_mthyr bucke.
```

### 2.3 Step 2: Aggregate Demand Quantity by DC and Month

Add this code after demand data is read (around line 250-300):

```abap
" ============================================================
" STEP 2: Aggregate Planned Delivery Quantity by DC and Month
" Source: /SAPAPO/SNPOPDMN (Field DELIV)
" Exclude DUMMY SHP locations from lt_/sapapo/loc
" ============================================================
CLEAR: lt_demand_by_dc_month, lt_total_demand_by_month.

" Get unique Distribution Channels from demand data
CLEAR: lt_vtweg_list.
LOOP AT lt_dcdrv INTO lw_dcdrv.
  READ TABLE lt_vtweg_list WITH KEY table_line = lw_dcdrv-zvtweg TRANSPORTING NO FIELDS.
  IF sy-subrc <> 0.
    APPEND lw_dcdrv-zvtweg TO lt_vtweg_list.
  ENDIF.
ENDLOOP.

" Aggregate demand by DC and month
" Note: Demand data (lt_/sapapo/snpopdmn) doesn't have bucket/month directly
" We need to map location to bucket using planning driver or other logic
" For now, assuming we can aggregate by DC and then split by month in output

LOOP AT lt_/sapapo/snpopdmn INTO lw_/sapapo/snpopdmn.
  " Check if this location is DUMMY SHP (exclude from total)
  READ TABLE lt_/sapapo/loc TRANSPORTING NO FIELDS
    WITH KEY locid = lw_/sapapo/snpopdmn-locid
    BINARY SEARCH.
  
  lv_is_dummy = sy-subrc = 0. " If found in lt_/sapapo/loc, it's DUMMY SHP
  
  " Get Distribution Channel for this location
  CLEAR lw_dcdrv.
  READ TABLE lt_dcdrv INTO lw_dcdrv
    WITH KEY locid = lw_/sapapo/snpopdmn-locid
    BINARY SEARCH.
  
  IF sy-subrc = 0.
    " For each bucket/month, we need to aggregate
    " Since demand doesn't have bucket directly, we'll aggregate by DC first
    " Then in output generation, we'll distribute by month based on production buckets
    
    " Aggregate by DC (we'll split by month later using production buckets)
    READ TABLE lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>
      WITH KEY zvtweg = lw_dcdrv-zvtweg.
    
    IF sy-subrc <> 0.
      APPEND INITIAL LINE TO lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>.
      <lfs_demand_dc>-zvtweg = lw_dcdrv-zvtweg.
      <lfs_demand_dc>-plan_del_qty = 0.
    ENDIF.
    
    <lfs_demand_dc>-plan_del_qty = <lfs_demand_dc>-plan_del_qty + 
                                   lw_/sapapo/snpopdmn-deliv.
    
    " Aggregate total demand (excluding DUMMY SHP) by month
    " Note: We need to map demand to months using bucket definitions
    " For now, aggregate total excluding DUMMY
    IF lv_is_dummy = abap_false.
      " Add to total demand (excluding DUMMY)
      " We'll need to split by month in output generation
      READ TABLE lt_total_demand_by_month ASSIGNING <lfs_total_demand>
        WITH KEY date_mthyr = space. " Temporary, will update in output loop
      
      IF sy-subrc <> 0.
        APPEND INITIAL LINE TO lt_total_demand_by_month ASSIGNING <lfs_total_demand>.
        <lfs_total_demand>-date_mthyr = space.
        <lfs_total_demand>-total_del_qty = 0.
      ENDIF.
      
      <lfs_total_demand>-total_del_qty = <lfs_total_demand>-total_del_qty + 
                                          lw_/sapapo/snpopdmn-deliv.
    ENDIF.
  ENDIF.
ENDLOOP.

SORT lt_demand_by_dc_month BY zvtweg.
SORT lt_total_demand_by_month BY date_mthyr.
```

**Note**: The above logic needs refinement based on how demand data maps to buckets/months. The key is to ensure we can calculate:
- Production Qty per month (from production buckets)
- DC Demand Qty per month (from demand data mapped to months)
- Total Demand Qty per month excluding DUMMY SHP

### 2.4 Step 3: Replace Output Generation Logic

**CRITICAL CHANGE**: Replace the entire output generation section (starting around line 400 where `LOOP AT it_input` begins) with the following:

```abap
" ============================================================
" STEP 3: Generate Output with Multiple Months per Distribution Channel
" ============================================================
CLEAR: lt_output_tmp, lt_output_tmp1, et_output, et_intout.

LOOP AT it_input ASSIGNING <lfs_input>.
  CLEAR: lw_output, lw_output1, lt_ref, lt_intout, lt_output_tmp, lt_output_tmp1.
  
  " Prepare reference combinations
  CALL FUNCTION 'ZAPO_NTBK_COMB_PREPARE'
    EXPORTING
      is_input     = <lfs_input>
      it_drv_matnr = it_drv_matnr
    IMPORTING
      et_out       = lt_ref.
  
  " Get Distribution Channels from input
  CLEAR: lt_vtweg_list.
  IF <lfs_input>-zvtweg IS NOT INITIAL.
    " Split comma-separated DCs if any
    SPLIT <lfs_input>-zvtweg AT ',' INTO TABLE lt_vtweg_list.
  ELSE.
    " If no DC specified in input, use all DCs from demand data
    CLEAR lt_vtweg_list.
    LOOP AT lt_dcdrv INTO lw_dcdrv.
      READ TABLE lt_vtweg_list WITH KEY table_line = lw_dcdrv-zvtweg TRANSPORTING NO FIELDS.
      IF sy-subrc <> 0.
        APPEND lw_dcdrv-zvtweg TO lt_vtweg_list.
      ENDIF.
    ENDLOOP.
  ENDIF.
  
  " Loop through each reference combination
  LOOP AT lt_ref ASSIGNING <lfs_ref>.
    CLEAR: lw_locid, lw_matid.
    
    " Get location and material IDs
    IF <lfs_ref>-ntbk_locfr IS NOT INITIAL.
      READ TABLE it_loc ASSIGNING <lfs_loc> WITH KEY locno = <lfs_ref>-ntbk_locfr.
      IF sy-subrc = 0 AND <lfs_loc> IS ASSIGNED.
        lw_locid = <lfs_loc>-locid.
      ENDIF.
    ENDIF.
    
    IF <lfs_ref>-ntbk_matnr IS NOT INITIAL.
      READ TABLE it_matkey ASSIGNING <lfs_matkey> WITH KEY matnr = <lfs_ref>-ntbk_matnr.
      IF sy-subrc = 0 AND <lfs_matkey> IS ASSIGNED.
        lw_matid = <lfs_matkey>-matid.
      ENDIF.
    ENDIF.
    
    " ============================================================
    " Loop through each Distribution Channel
    " ============================================================
    LOOP AT lt_vtweg_list INTO lv_vtweg.
      
      " ============================================================
      " Loop through each month from bucket definitions
      " ============================================================
      LOOP AT lt_plandrv ASSIGNING <lfs_plandrv>.
        CLEAR: lw_output, lw_output1, lv_numerator, lv_denominator.
        
        lv_calc_month = <lfs_plandrv>-date_mthyr.
        
        " ============================================================
        " Get Production Quantity for this month and bucket
        " ============================================================
        CLEAR: lw_prod_month_dc.
        READ TABLE lt_prod_by_month_dc INTO lw_prod_month_dc
          WITH KEY date_mthyr = lv_calc_month
                   bucke = <lfs_plandrv>-bucke
          BINARY SEARCH.
        
        IF sy-subrc <> 0.
          " No production data for this month - skip
          CONTINUE.
        ENDIF.
        
        " ============================================================
        " Get Planned Delivery Quantity for this DC
        " Note: Since demand may not be directly mapped to buckets,
        " we use the aggregated DC demand and distribute proportionally
        " OR use the existing lv_domestic_qty, lv_export_qty logic
        " ============================================================
        CLEAR: lv_domestic_qty, lv_export_qty, lv_non_domes_export_qty, lv_dummy_qty.
        
        " Re-calculate DC-specific demand quantities for this material
        READ TABLE lt_/sapapo/snpopdmn TRANSPORTING NO FIELDS 
          WITH KEY matid = lw_matid BINARY SEARCH.
        
        IF sy-subrc = 0.
          lv_tabix = sy-tabix.
          
          LOOP AT lt_/sapapo/snpopdmn INTO lw_/sapapo/snpopdmn FROM lv_tabix.
            IF lw_/sapapo/snpopdmn-matid NE lw_matid.
              EXIT.
            ENDIF.
            
            CLEAR lw_dcdrv.
            READ TABLE lt_dcdrv INTO lw_dcdrv 
              WITH KEY locid = lw_/sapapo/snpopdmn-locid BINARY SEARCH.
            
            IF sy-subrc = 0.
              " Check if DUMMY SHP
              READ TABLE lt_/sapapo/loc_1 TRANSPORTING NO FIELDS
                WITH KEY locid = lw_dcdrv-locid
                         locno = lw_dcdrv-locno BINARY SEARCH.
              
              lv_is_dummy = sy-subrc = 0.
              
              " Aggregate by DC
              CASE lw_dcdrv-zvtweg.
                WHEN '10'. " Domestic
                  lv_domestic_qty = lv_domestic_qty + lw_/sapapo/snpopdmn-deliv.
                  IF lv_is_dummy = abap_true.
                    lv_dummy_qty = lv_dummy_qty + lw_/sapapo/snpopdmn-deliv.
                  ENDIF.
                WHEN '15'. " Export
                  lv_export_qty = lv_export_qty + lw_/sapapo/snpopdmn-deliv.
                  IF lv_is_dummy = abap_true.
                    lv_dummy_qty = lv_dummy_qty + lw_/sapapo/snpopdmn-deliv.
                  ENDIF.
                WHEN OTHERS.
                  lv_non_domes_export_qty = lv_non_domes_export_qty + lw_/sapapo/snpopdmn-deliv.
                  IF lv_is_dummy = abap_true.
                    lv_dummy_qty = lv_dummy_qty + lw_/sapapo/snpopdmn-deliv.
                  ENDIF.
              ENDCASE.
            ENDIF.
          ENDLOOP.
        ENDIF.
        
        " Calculate total demand excluding DUMMY SHP
        lv_demand_plus_dummy_qty = lv_domestic_qty + lv_export_qty + lv_non_domes_export_qty - lv_dummy_qty.
        
        " Get DC-specific demand quantity
        CASE lv_vtweg.
          WHEN '10'. " Domestic
            IF lv_is_dummy = abap_true.
              lv_dc_demand_qty = lv_domestic_qty - lv_dummy_qty.
            ELSE.
              lv_dc_demand_qty = lv_domestic_qty.
            ENDIF.
          WHEN '15'. " Export
            IF lv_is_dummy = abap_true.
              lv_dc_demand_qty = lv_export_qty - lv_dummy_qty.
            ELSE.
              lv_dc_demand_qty = lv_export_qty.
            ENDIF.
          WHEN OTHERS.
            IF lv_is_dummy = abap_true.
              lv_dc_demand_qty = lv_non_domes_export_qty - lv_dummy_qty.
            ELSE.
              lv_dc_demand_qty = lv_non_domes_export_qty.
            ENDIF.
        ENDCASE.
        
        " ============================================================
        " CALCULATE NETBACK QUANTITY using the formula:
        " NTBK_QTY = (Prod Qty for Month) × (DC Planned Del Qty) 
        "            ÷ (Total Planned Del Qty excl DUMMY SHP)
        " ============================================================
        
        " Initialize output structure
        CLEAR: lw_output.
        MOVE-CORRESPONDING <lfs_input> TO lw_output.
        CLEAR: lw_output-ntbk_index.
        
        " Set key fields from reference
        lw_output-ntbk_matnr = <lfs_ref>-ntbk_matnr.
        lw_output-ntbk_locfr = <lfs_ref>-ntbk_locfr.
        lw_output-spart = <lfs_ref>-div.
        lw_output-prodh1 = <lfs_ref>-pdh1.
        lw_output-prodh2 = <lfs_ref>-pdh2.
        
        " Set Distribution Channel and Month
        lw_output-zvtweg = lv_vtweg.
        lw_output-ntbk_mth = lv_calc_month.
        lw_output-sessionname = is_session-sessionname.
        
        " Calculate numerator: Production Qty × DC Demand Qty
        lv_numerator = lw_prod_month_dc-prod_qty * lv_dc_demand_qty.
        
        " Denominator: Total Demand Qty excluding DUMMY SHP
        lv_denominator = lv_demand_plus_dummy_qty.
        
        " Apply formula
        IF lv_denominator <> 0.
          lw_output-ntbk_qty = lv_numerator / lv_denominator.
        ELSE.
          lw_output-ntbk_qty = 0.
        ENDIF.
        
        " Set other required fields
        lw_output-ntbk_cat = <lfs_input>-ntbk_cat.
        lw_output-ntbk_logic = <lfs_input>-ntbk_logic.
        lw_output-ntbk_site = <lfs_input>-ntbk_site.
        
        " Append to output (use APPEND, not COLLECT, to preserve all monthly records)
        APPEND lw_output TO lt_output_tmp1.
        
        " Also prepare intermediate output for ET_INTOUT if needed
        CLEAR: lw_output_1.
        lw_output_1-ntbk_matnr = <lfs_ref>-ntbk_matnr.
        lw_output_1-ntbk_locfr = <lfs_ref>-ntbk_locfr.
        lw_output_1-spart = <lfs_ref>-div.
        lw_output_1-prodh1 = <lfs_ref>-pdh1.
        lw_output_1-prodh2 = <lfs_ref>-pdh2.
        lw_output_1-bucke = <lfs_plandrv>-bucke.
        lw_output_1-prodn = lw_prod_month_dc-prod_qty.
        CONDENSE lw_output_1-bucke.
        COLLECT lw_output_1 INTO lt_intout.
        
      ENDLOOP. " End of bucket/month loop
      
    ENDLOOP. " End of Distribution Channel loop
    
  ENDLOOP. " End of reference combination loop
  
  " ============================================================
  " Assign ntbk_index and move to export parameter
  " ============================================================
  IF lt_output_tmp1 IS NOT INITIAL.
    lv_ntbk_index = <lfs_input>-ntbk_index.
    
    LOOP AT lt_output_tmp1 ASSIGNING <lfs_output>.
      <lfs_output>-ntbk_index = lv_ntbk_index.
      lv_ntbk_index = lv_ntbk_index + 1.
      APPEND <lfs_output> TO et_output.
    ENDLOOP.
    
    IF lt_intout IS NOT INITIAL.
      APPEND LINES OF lt_intout TO et_intout.
    ENDIF.
  ENDIF.
  
ENDLOOP. " End of input loop

" ============================================================
" Final sorting (DO NOT remove duplicates - we need all monthly records)
" ============================================================
IF et_output IS NOT INITIAL.
  SORT et_output BY ntbk_index zvtweg ntbk_mth.
  " REMOVED: DELETE ADJACENT DUPLICATES - we need all monthly records
ENDIF.

IF et_intout IS NOT INITIAL.
  SORT et_intout BY ntbk_matnr ntbk_locfr bucke spart prodh1 prodh2.
  DELETE ADJACENT DUPLICATES FROM et_intout 
    COMPARING ntbk_matnr ntbk_locfr bucke spart prodh1 prodh2.
ENDIF.
```

### 2.5 Additional Variable Declarations Needed

Add these variable declarations in the data declaration section:

```abap
DATA: lv_dc_demand_qty TYPE /sapapo/snpdeliv,
      lv_is_dummy TYPE abap_bool,
      lv_tabix TYPE sy-tabix.
```

---

## 3. Key Changes Summary

### 3.1 Removed Logic
- ❌ Remove the duplicate removal: `DELETE ADJACENT DUPLICATES FROM et_output COMPARING zvtweg ntbk_qty`
- ❌ Remove aggregation logic that combines months
- ❌ Remove the old calculation that doesn't properly split by month

### 3.2 Added Logic
- ✅ Loop through each Distribution Channel from `ZAPO_NTBK_REF.ZVTWEG`
- ✅ Loop through each month from bucket definitions (`lt_plandrv`)
- ✅ Calculate formula per month per DC: `(Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY)`
- ✅ Generate separate output records for each month per DC
- ✅ Use `APPEND` instead of `COLLECT` to preserve all monthly records

### 3.3 Formula Implementation
```abap
lv_numerator = lw_prod_month_dc-prod_qty * lv_dc_demand_qty.
lv_denominator = lv_demand_plus_dummy_qty.
IF lv_denominator <> 0.
  lw_output-ntbk_qty = lv_numerator / lv_denominator.
ELSE.
  lw_output-ntbk_qty = 0.
ENDIF.
```

---

## 4. Testing Checklist

- [ ] Multiple months generated per Distribution Channel
- [ ] Formula calculation matches requirement: `(Prod × DC Del) ÷ Total Del (excl DUMMY)`
- [ ] DUMMY SHP locations excluded from denominator
- [ ] All Distribution Channels from `ZAPO_NTBK_REF.ZVTWEG` processed
- [ ] No duplicate removal of monthly records
- [ ] Production quantities correctly mapped to months via bucket definitions
- [ ] Demand quantities correctly aggregated by DC and month

---

## 5. Important Notes

1. **Bucket to Month Mapping**: Ensure `lt_plandrv` correctly maps buckets (`BUCKE`) to months (`date_mthyr`)

2. **Demand to Month Mapping**: The current code assumes demand can be mapped to months. If demand data doesn't have direct bucket/month mapping, you may need to:
   - Use proportional distribution based on production buckets
   - Or use a separate mapping table/function

3. **DUMMY SHP Exclusion**: The code already has logic to identify DUMMY SHP locations from `lt_/sapapo/loc`. Ensure this is correctly applied in the denominator calculation.

4. **Distribution Channel Source**: If `ZVTWEG` is not directly available in demand data, the existing `lt_dcdrv` table (from `ZAPO_NTBK_DCDRV`) should provide the mapping from location to DC.

5. **Performance**: Consider adding indexes or optimizing the nested loops if dealing with large datasets.

---

## 6. Code Location Reference

- **Data Declarations**: Add around line 100-150
- **Production Aggregation**: Replace around line 300-400
- **Demand Aggregation**: Add around line 250-300
- **Output Generation**: Replace entire section starting around line 400 (main `LOOP AT it_input`)

---

**End of Correction Document**

