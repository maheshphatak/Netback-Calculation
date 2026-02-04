# Specific Code Corrections for ZAPO_NTBK_PROD_DSHP_EXCLUDE

## Problem Statement
1. Netback calculation should provide multiple month output as per Bucket Definition for every Distribution Channel maintained in Table ZAPO_NTBK_REF in Field 'ZVTWEG'
2. Formula: Production Quantity for the month from IT_PROMO (Table /SAPAPO/SNPOPPRO in Field 'PRODU') for every Netback Month as per bucket Definition (ET_BUCKDF) in Table /SAPAPO/SNPOPBCT in Field 'BUCKE' multiplied by summation of 'Planned Delivery Quantity' for the month from IT_DEMAND (Table /SAPAPO/SNPOPDMN in Field DELIV) Divided by Total Planned Delivery Quantity excluding DUMMY SHP mentioned in Table ZAPOPARAM-Value2

---

## CRITICAL ISSUE: Current Code Problem

The current code has the following issues:

1. **Missing Month-wise Loop**: The code calculates quantities but doesn't properly loop through each month from bucket definitions for each DC
2. **Incorrect Aggregation**: The final output uses `DELETE ADJACENT DUPLICATES FROM et_output COMPARING zvtweg ntbk_qty` which removes monthly records
3. **Formula Not Applied Per Month**: The formula calculation happens but not properly per month per DC

---

## SPECIFIC CODE CORRECTIONS

### Correction 1: Remove Duplicate Removal (Line ~850)

**FIND THIS CODE:**
```abap
IF et_output IS NOT INITIAL.
  SORT et_output BY zvtweg ntbk_qty.
  DELETE ADJACENT DUPLICATES FROM et_output COMPARING zvtweg ntbk_qty.
ENDIF.
```

**REPLACE WITH:**
```abap
IF et_output IS NOT INITIAL.
  SORT et_output BY ntbk_index zvtweg ntbk_mth.
  " DO NOT remove duplicates - we need all monthly records per DC
ENDIF.
```

---

### Correction 2: Restructure Output Generation Logic

**FIND THE SECTION** starting around line 400 where it says:
```abap
LOOP AT it_input ASSIGNING <lfs_input>.
  CLEAR: lw_output, lw_locid, lw_matid, lt_ref, lt_intout.
  ...
```

**REPLACE THE ENTIRE OUTPUT GENERATION LOGIC** (from the `LOOP AT it_input` through the end of that loop, approximately lines 400-850) with the following:

```abap
" ============================================================
" MAIN LOOP: Process each input record
" ============================================================
LOOP AT it_input ASSIGNING <lfs_input>.
  CLEAR: lw_output, lw_locid, lw_matid, lt_ref, lt_intout, lt_output_tmp, lt_output_tmp1.

  " Prepare reference combinations
  CALL FUNCTION 'ZAPO_NTBK_COMB_PREPARE'
    EXPORTING
      is_input     = <lfs_input>
      it_drv_matnr = it_drv_matnr
    IMPORTING
      et_out       = lt_ref.

  MOVE-CORRESPONDING <lfs_input> TO lw_output.
  CLEAR: lw_output-ntbk_index.

  " ============================================================
  " Get Distribution Channels from input
  " ============================================================
  DATA: lt_vtweg_list TYPE STANDARD TABLE OF vtweg,
        lv_vtweg TYPE vtweg.
  
  CLEAR: lt_vtweg_list.
  IF <lfs_input>-zvtweg IS NOT INITIAL.
    " Split comma-separated DCs if any
    SPLIT <lfs_input>-zvtweg AT ',' INTO TABLE lt_vtweg_list.
  ELSE.
    " If no DC specified, get all DCs from demand data
    CLEAR lt_vtweg_list.
    LOOP AT lt_dcdrv INTO lw_dcdrv.
      READ TABLE lt_vtweg_list WITH KEY table_line = lw_dcdrv-zvtweg TRANSPORTING NO FIELDS.
      IF sy-subrc <> 0.
        APPEND lw_dcdrv-zvtweg TO lt_vtweg_list.
      ENDIF.
    ENDLOOP.
  ENDIF.

  " ============================================================
  " Loop through each reference combination
  " ============================================================
  LOOP AT lt_ref ASSIGNING <lfs_ref>.
    CLEAR: lw_locid, lw_matid, lw_output_1.

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
    " Calculate production quantities by month (from existing logic)
    " ============================================================
    IF <lfs_ref>-ntbk_matnr IS NOT INITIAL AND <lfs_ref>-ntbk_locfr IS NOT INITIAL.
      " Case 1: Both material and location specified
      LOOP AT lt_snpoppma ASSIGNING <lfs_snpoppma> 
        WHERE sessionid = is_session-sessionid
          AND matid = lw_matid
          AND locid = lw_locid.
        
        LOOP AT lt_snpopbct_con ASSIGNING <lfs_snpopbct_con> 
          WHERE sessionid = <lfs_snpoppma>-sessionid
            AND proid = <lfs_snpoppma>-proid.

          " ============================================================
          " CRITICAL: Loop through each month from bucket definition
          " ============================================================
          LOOP AT lt_plandrv ASSIGNING <lfs_plandrv> 
            WHERE sessionid = <lfs_snpoppma>-sessionid
              AND bucke = <lfs_snpopbct_con>-bucke.

            " Calculate production quantity for this month
            lw_output-ntbk_mth = <lfs_plandrv>-date_mthyr.
            lw_output-ntbk_qty = <lfs_snpoppma>-outin * <lfs_snpopbct_con>-produ.

            " Prepare intermediate output
            lw_output_1-ntbk_matnr = <lfs_ref>-ntbk_matnr.
            lw_output_1-ntbk_locfr = <lfs_ref>-ntbk_locfr.
            lw_output_1-spart = <lfs_ref>-div.
            lw_output_1-prodh1 = <lfs_ref>-pdh1.
            lw_output_1-prodh2 = <lfs_ref>-pdh2.
            lw_output_1-bucke = <lfs_snpopbct_con>-bucke.
            lw_output_1-prodn = lw_output-ntbk_qty.
            CONDENSE lw_output_1-bucke.
            COLLECT lw_output_1 INTO lt_intout.

            " Store production quantity by month for later use
            COLLECT lw_output INTO lt_output_tmp.

          ENDLOOP. " End of bucket/month loop
        ENDLOOP. " End of production bucket loop
      ENDLOOP. " End of production master loop

    ELSEIF <lfs_ref>-ntbk_matnr IS NOT INITIAL AND <lfs_ref>-ntbk_locfr IS INITIAL.
      " Case 2: Only material specified
      LOOP AT lt_snpoppma ASSIGNING <lfs_snpoppma> 
        WHERE sessionid = is_session-sessionid
          AND matid = lw_matid.
        
        LOOP AT lt_snpopbct_con ASSIGNING <lfs_snpopbct_con> 
          WHERE sessionid = <lfs_snpoppma>-sessionid
            AND proid = <lfs_snpoppma>-proid.

          LOOP AT lt_plandrv ASSIGNING <lfs_plandrv> 
            WHERE sessionid = <lfs_snpoppma>-sessionid
              AND bucke = <lfs_snpopbct_con>-bucke.

            lw_output-ntbk_mth = <lfs_plandrv>-date_mthyr.
            lw_output-ntbk_qty = <lfs_snpoppma>-outin * <lfs_snpopbct_con>-produ.

            lw_output_1-ntbk_matnr = <lfs_ref>-ntbk_matnr.
            lw_output_1-ntbk_locfr = <lfs_ref>-ntbk_locfr.
            lw_output_1-spart = <lfs_ref>-div.
            lw_output_1-prodh1 = <lfs_ref>-pdh1.
            lw_output_1-prodh2 = <lfs_ref>-pdh2.
            lw_output_1-prodn = lw_output-ntbk_qty.
            lw_output_1-bucke = <lfs_snpopbct_con>-bucke.
            CONDENSE lw_output_1-bucke.
            COLLECT lw_output_1 INTO lt_intout.
            COLLECT lw_output INTO lt_output_tmp.

          ENDLOOP.
        ENDLOOP.
      ENDLOOP.

    ELSEIF <lfs_ref>-ntbk_locfr IS NOT INITIAL AND <lfs_ref>-ntbk_matnr IS INITIAL.
      " Case 3: Only location specified
      LOOP AT lt_snpoppma ASSIGNING <lfs_snpoppma> 
        WHERE sessionid = is_session-sessionid
          AND locid = lw_locid.
        
        LOOP AT lt_snpopbct_con ASSIGNING <lfs_snpopbct_con> 
          WHERE sessionid = <lfs_snpoppma>-sessionid
            AND proid = <lfs_snpoppma>-proid.

          LOOP AT lt_plandrv ASSIGNING <lfs_plandrv> 
            WHERE sessionid = <lfs_snpoppma>-sessionid
              AND bucke = <lfs_snpopbct_con>-bucke.

            lw_output-ntbk_mth = <lfs_plandrv>-date_mthyr.
            lw_output-ntbk_qty = <lfs_snpoppma>-outin * <lfs_snpopbct_con>-produ.

            lw_output_1-ntbk_matnr = <lfs_ref>-ntbk_matnr.
            lw_output_1-ntbk_locfr = <lfs_ref>-ntbk_locfr.
            lw_output_1-spart = <lfs_ref>-div.
            lw_output_1-prodh1 = <lfs_ref>-pdh1.
            lw_output_1-prodh2 = <lfs_ref>-pdh2.
            lw_output_1-prodn = lw_output-ntbk_qty.
            lw_output_1-bucke = <lfs_snpopbct_con>-bucke.
            CONDENSE lw_output_1-bucke.
            COLLECT lw_output_1 INTO lt_intout.
            COLLECT lw_output INTO lt_output_tmp.

          ENDLOOP.
        ENDLOOP.
      ENDLOOP.
    ENDIF.

    " ============================================================
    " CRITICAL: Now apply formula per month per Distribution Channel
    " Formula: (Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY)
    " ============================================================
    
    " Calculate demand quantities by DC (existing logic)
    CLEAR: lv_domestic_qty, lv_export_qty, lv_non_domes_export_qty, lv_dummy_qty, lv_demand_qty.
    
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
        
        lv_demand_qty = lv_demand_qty + lw_/sapapo/snpopdmn-deliv.
      ENDLOOP.
    ENDIF.

    " Calculate total demand excluding DUMMY SHP
    lv_demand_plus_dummy_qty = lv_demand_qty - lv_dummy_qty.

    " ============================================================
    " CRITICAL: Loop through each Distribution Channel
    " ============================================================
    LOOP AT lt_vtweg_list INTO lv_vtweg.
      
      " ============================================================
      " CRITICAL: Loop through each month from production data
      " ============================================================
      LOOP AT lt_output_tmp INTO lw_output.
        
        CLEAR: lw_output1, lv_dc_demand_qty, lv_numerator, lv_denominator.
        
        " Get DC-specific demand quantity
        CASE lv_vtweg.
          WHEN '10'. " Domestic
            READ TABLE lt_/sapapo/loc_1 TRANSPORTING NO FIELDS
              WITH KEY locid = lw_locid BINARY SEARCH.
            IF sy-subrc = 0.
              " Location is DUMMY SHP, exclude from calculation
              lv_dc_demand_qty = lv_domestic_qty - lv_dummy_qty.
            ELSE.
              lv_dc_demand_qty = lv_domestic_qty.
            ENDIF.
          WHEN '15'. " Export
            READ TABLE lt_/sapapo/loc_1 TRANSPORTING NO FIELDS
              WITH KEY locid = lw_locid BINARY SEARCH.
            IF sy-subrc = 0.
              lv_dc_demand_qty = lv_export_qty - lv_dummy_qty.
            ELSE.
              lv_dc_demand_qty = lv_export_qty.
            ENDIF.
          WHEN OTHERS.
            READ TABLE lt_/sapapo/loc_1 TRANSPORTING NO FIELDS
              WITH KEY locid = lw_locid BINARY SEARCH.
            IF sy-subrc = 0.
              lv_dc_demand_qty = lv_non_domes_export_qty - lv_dummy_qty.
            ELSE.
              lv_dc_demand_qty = lv_non_domes_export_qty.
            ENDIF.
        ENDCASE.

        " ============================================================
        " APPLY FORMULA: (Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY)
        " ============================================================
        lv_numerator = lw_output-ntbk_qty * lv_dc_demand_qty.
        lv_denominator = lv_demand_plus_dummy_qty.

        " Initialize output structure
        CLEAR: lw_output1.
        MOVE-CORRESPONDING <lfs_input> TO lw_output1.
        CLEAR: lw_output1-ntbk_index.

        " Set key fields
        lw_output1-ntbk_matnr = <lfs_ref>-ntbk_matnr.
        lw_output1-ntbk_locfr = <lfs_ref>-ntbk_locfr.
        lw_output1-spart = <lfs_ref>-div.
        lw_output1-prodh1 = <lfs_ref>-pdh1.
        lw_output1-prodh2 = <lfs_ref>-pdh2.
        lw_output1-ntbk_cat = <lfs_input>-ntbk_cat.
        lw_output1-ntbk_logic = <lfs_input>-ntbk_logic.
        lw_output1-ntbk_site = <lfs_input>-ntbk_site.

        " Set Distribution Channel and Month
        lw_output1-zvtweg = lv_vtweg.
        lw_output1-ntbk_mth = lw_output-ntbk_mth.
        lw_output1-sessionname = is_session-sessionname.

        " Apply formula
        IF lv_denominator <> 0.
          lw_output1-ntbk_qty = lv_numerator / lv_denominator.
        ELSE.
          lw_output1-ntbk_qty = 0.
        ENDIF.

        " Append to output (use APPEND, not COLLECT, to preserve all monthly records)
        APPEND lw_output1 TO lt_output_tmp1.

      ENDLOOP. " End of month loop
      
    ENDLOOP. " End of Distribution Channel loop

  ENDLOOP. " End of reference combination loop

  " ============================================================
  " Assign ntbk_index and move to export parameter
  " ============================================================
  CLEAR lt_output_tmp[].
  lt_output_tmp = lt_output_tmp1.
  
  IF lt_output_tmp IS NOT INITIAL.
    lv_ntbk_index = <lfs_input>-ntbk_index.
    
    LOOP AT lt_output_tmp ASSIGNING <lfs_output>.
      <lfs_output>-ntbk_index = lv_ntbk_index.
      lv_ntbk_index = lv_ntbk_index + 1.
      APPEND <lfs_output> TO et_output.
    ENDLOOP.
    
    IF lt_intout IS NOT INITIAL.
      APPEND LINES OF lt_intout TO et_intout.
    ENDIF.
    
    CLEAR: lt_output_tmp, lt_intout, lt_output_tmp1.
  ENDIF.

ENDLOOP. " End of input loop
```

---

### Correction 3: Add Missing Variable Declarations

**ADD THESE VARIABLES** in the data declaration section (around line 100):

```abap
DATA: lt_vtweg_list TYPE STANDARD TABLE OF vtweg,
      lv_vtweg TYPE vtweg,
      lv_dc_demand_qty TYPE /sapapo/snpdeliv,
      lv_numerator TYPE /sapapo/snpdeliv,
      lv_denominator TYPE /sapapo/snpdeliv,
      lv_is_dummy TYPE abap_bool.
```

---

## Summary of Changes

1. ✅ **Added Distribution Channel Loop**: Loop through each DC from `ZAPO_NTBK_REF.ZVTWEG`
2. ✅ **Added Month Loop**: Loop through each month from production data (`lt_output_tmp`)
3. ✅ **Applied Formula Per Month Per DC**: Calculate `(Prod Qty × DC Del Qty) ÷ Total Del Qty (excl DUMMY)` for each month/DC combination
4. ✅ **Removed Duplicate Removal**: Changed final sorting to preserve all monthly records
5. ✅ **Used APPEND Instead of COLLECT**: Preserve all monthly records in output

---

## Testing Verification

After implementing these changes, verify:
- [ ] Each Distribution Channel has separate records for each month
- [ ] Formula is calculated correctly: `(Prod × DC Del) ÷ Total Del (excl DUMMY)`
- [ ] DUMMY SHP locations are excluded from denominator
- [ ] All months from bucket definitions are present in output
- [ ] No records are incorrectly removed as duplicates

---

**End of Specific Corrections**

