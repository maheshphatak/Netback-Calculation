# Code Correction: Multiple Month Output for ZAPO_NTBK_DMD_DSHP_EXCLUDE

## 1. Context
- **Function Module**: `ZAPO_NTBK_DMD_DSHP_EXCLUDE`
- **Purpose**: Calculate Netback output for Business Rule "TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)"
- **Output Table**: `ZAPO_NTBK_DETAIL`
- **Input Table**: `ZAPO_NTBK_REF` (contains Distribution Channel information)
- **Reference Tables**:
  - `IT_PROMO` (from `/SAPAPO/SNPOPPRO`) - Production Quantity
  - `IT_DEMAND` (from `/SAPAPO/SNPOPDMN`) - Planned Delivery Quantity
  - `ET_BUCKDF` (from `/SAPAPO/SNPOPBCT`) - Bucket Definition for Netback Months
  - `ZAPOPARAM` - Contains DUMMY SHP exclusion list

---

## 2. Current Observation / Problem Statement

### 2.1 Issues Identified

1. **Missing Multiple Month Output**: 
   - Current implementation does not generate separate output records for each month as per Bucket Definition
   - Only single month or aggregated output is being saved to `ZAPO_NTBK_DETAIL`

2. **Incorrect Formula Implementation**:
   - For Business Rule "TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)", the calculation does not properly:
     - Consider Production Quantity from `IT_PROMO` for each Netback Month per bucket definition
     - Apply the correct ratio based on Planned Delivery Quantity from `IT_DEMAND`
     - Exclude DUMMY SHP shipments as per `ZAPOPARAM` table

3. **Distribution Channel Handling**:
   - Calculation is not performed separately for each Distribution Channel (DC) maintained in `ZAPO_NTBK_REF`
   - Multiple DCs may be aggregated incorrectly

---

## 3. Business Requirements

### 3.1 Requirement 1: Multiple Month Output per Distribution Channel

**Requirement**: Netback calculation output should provide **multiple month output** as per Bucket Definition (`ET_BUCKDF`) for **every Distribution Channel** maintained in Table `ZAPO_NTBK_REF`.

**Expected Behavior**:
- For each Distribution Channel in `ZAPO_NTBK_REF`, generate separate output records in `ZAPO_NTBK_DETAIL` for each month defined in the Bucket Definition
- Each output record should have:
  - Correct `NTBK_MTH` (month-year from bucket definition)
  - Correct `NTBK_QTY` (calculated quantity for that month)
  - All key fields from `ZAPO_NTBK_REF` (product, location, DC, etc.)

### 3.2 Requirement 2: Correct Formula for EXDSHP Business Rule

**Requirement**: For Business Rule "TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)" for each Distribution Channel (DC), the calculation should use:

**Formula**:
```
For each Netback Month (as per ET_BUCKDF):
  NTBK_QTY = (Production Qty for Month from IT_PROMO) 
             × (Sum of Planned Delivery Qty for Month from IT_DEMAND for this DC)
             ÷ (Total Planned Delivery Qty excluding DUMMY SHP from ZAPOPARAM)
```

**Detailed Logic**:
1. **Production Quantity**: Read 'Production Quantity' for the month from `IT_PROMO` (Table `/SAPAPO/SNPOPPRO`) for every Netback Month as per bucket Definition (`ET_BUCKDF`) in Table `/SAPAPO/SNPOPBCT`
2. **Planned Delivery Quantity (DC-specific)**: Sum 'Planned Delivery Quantity' for the month from `IT_DEMAND` (Table `/SAPAPO/SNPOPDMN`) filtered by:
   - The specific Distribution Channel (DC)
   - The month (as per bucket definition)
3. **Total Planned Delivery Quantity (Denominator)**: Calculate Total Planned Delivery Quantity from `IT_DEMAND` excluding:
   - DUMMY SHP shipments mentioned in Table `ZAPOPARAM` (parameter `ZAPO_NTBK_DMD_DSHP_EXCLUDE`)
   - For the same month
4. **Apply Formula**: Multiply Production Qty by DC-specific Planned Delivery Qty, then divide by Total Planned Delivery Qty (excluding DUMMY SHP)

---

## 4. Technical Solution

### 4.1 Data Structures and Tables

```abap
" Input Tables
DATA: it_input TYPE ZAPO_NTBK_REF_T,           " Input from ZAPO_NTBK_REF
      it_promo TYPE STANDARD TABLE OF /SAPAPO/SNPOPPRO,  " Production data
      it_demand TYPE STANDARD TABLE OF /SAPAPO/SNPOPDMN, " Demand data
      et_buckdf TYPE STANDARD TABLE OF /SAPAPO/SNPOPBCT, " Bucket definition
      it_param TYPE STANDARD TABLE OF ZAPOPARAM.         " Parameter table

" Internal working tables
DATA: lt_dummy_shp TYPE STANDARD TABLE OF string,       " DUMMY SHP list
      lt_output TYPE ZAPO_NTBK_DETAIL_T,                 " Output table
      lt_promo_by_month TYPE STANDARD TABLE OF ty_promo_month,
      lt_demand_by_dc_month TYPE STANDARD TABLE OF ty_demand_dc_month,
      lt_total_demand_by_month TYPE STANDARD TABLE OF ty_total_demand_month.

" Work areas
DATA: lw_input TYPE ZAPO_NTBK_REF,
      lw_output TYPE ZAPO_NTBK_DETAIL,
      lw_bucket TYPE /SAPAPO/SNPOPBCT,
      lw_promo TYPE /SAPAPO/SNPOPPRO,
      lw_demand TYPE /SAPAPO/SNPOPDMN.

" Local types
TYPES: BEGIN OF ty_promo_month,
         month TYPE dats,              " Month-year
         prod_qty TYPE menge_d,        " Production quantity
       END OF ty_promo_month.

TYPES: BEGIN OF ty_demand_dc_month,
         dist_channel TYPE vtweg,      " Distribution Channel
         month TYPE dats,              " Month-year
         plan_del_qty TYPE menge_d,    " Planned delivery quantity
       END OF ty_demand_dc_month.

TYPES: BEGIN OF ty_total_demand_month,
         month TYPE dats,              " Month-year
         total_del_qty TYPE menge_d,   " Total delivery quantity (excl DUMMY SHP)
       END OF ty_total_demand_month.
```

### 4.2 Step 1: Read DUMMY SHP Exclusion List

```abap
" ============================================================
" STEP 1: Read DUMMY SHP exclusion list from ZAPOPARAM
" ============================================================
CLEAR: lt_dummy_shp.

SELECT * FROM ZAPOPARAM
  INTO TABLE it_param
  WHERE param_name = 'ZAPO_NTBK_DMD_DSHP_EXCLUDE'.

IF sy-subrc = 0.
  LOOP AT it_param INTO DATA(lw_param).
    " Assuming param_value contains comma-separated list of DUMMY SHP values
    " Split and add to exclusion list
    SPLIT lw_param-param_value AT ',' INTO TABLE DATA(lt_values).
    LOOP AT lt_values INTO DATA(lv_value).
      CONDENSE lv_value.
      IF lv_value IS NOT INITIAL.
        APPEND lv_value TO lt_dummy_shp.
      ENDIF.
    ENDLOOP.
  ENDLOOP.
ENDIF.

" Sort and remove duplicates
SORT lt_dummy_shp.
DELETE ADJACENT DUPLICATES FROM lt_dummy_shp.
```

### 4.3 Step 2: Prepare Production Quantity by Month

```abap
" ============================================================
" STEP 2: Aggregate Production Quantity by Month from IT_PROMO
" ============================================================
CLEAR: lt_promo_by_month.

LOOP AT it_promo INTO lw_promo.
  " Map bucket to month-year using ET_BUCKDF
  READ TABLE et_buckdf INTO lw_bucket
    WITH KEY bckto = lw_promo-bckto.
  
  IF sy-subrc = 0.
    " Get month-year from bucket definition
    DATA(lv_month) = lw_bucket-date_mythr.  " Assuming date_mythr contains month-year
    
    " Aggregate production quantity by month
    READ TABLE lt_promo_by_month ASSIGNING FIELD-SYMBOL(<lfs_promo>)
      WITH KEY month = lv_month.
    
    IF sy-subrc <> 0.
      " Create new entry
      APPEND INITIAL LINE TO lt_promo_by_month ASSIGNING <lfs_promo>.
      <lfs_promo>-month = lv_month.
      <lfs_promo>-prod_qty = 0.
    ENDIF.
    
    " Add production quantity (assuming field name is prod_qty or similar)
    <lfs_promo>-prod_qty = <lfs_promo>-prod_qty + lw_promo-prod_qty.
  ENDIF.
ENDLOOP.

SORT lt_promo_by_month BY month.
```

### 4.4 Step 3: Prepare Planned Delivery Quantity by DC and Month

```abap
" ============================================================
" STEP 3: Aggregate Planned Delivery Quantity by DC and Month from IT_DEMAND
" ============================================================
CLEAR: lt_demand_by_dc_month, lt_total_demand_by_month.

LOOP AT it_demand INTO lw_demand.
  " Check if this is a DUMMY SHP shipment (exclude it)
  READ TABLE lt_dummy_shp TRANSPORTING NO FIELDS
    WITH KEY table_line = lw_demand-shipment_type.  " Adjust field name as needed
  
  IF sy-subrc = 0.
    " This is a DUMMY SHP - skip it
    CONTINUE.
  ENDIF.
  
  " Map bucket to month-year using ET_BUCKDF
  READ TABLE et_buckdf INTO lw_bucket
    WITH KEY bckto = lw_demand-bckto.
  
  IF sy-subrc = 0.
    DATA(lv_demand_month) = lw_bucket-date_mythr.
    DATA(lv_dist_channel) = lw_demand-vtweg.  " Distribution Channel
    
    " Aggregate by DC and Month (for numerator)
    READ TABLE lt_demand_by_dc_month ASSIGNING FIELD-SYMBOL(<lfs_demand_dc>)
      WITH KEY dist_channel = lv_dist_channel
               month = lv_demand_month.
    
    IF sy-subrc <> 0.
      APPEND INITIAL LINE TO lt_demand_by_dc_month ASSIGNING <lfs_demand_dc>.
      <lfs_demand_dc>-dist_channel = lv_dist_channel.
      <lfs_demand_dc>-month = lv_demand_month.
      <lfs_demand_dc>-plan_del_qty = 0.
    ENDIF.
    
    " Add planned delivery quantity (adjust field name as needed)
    <lfs_demand_dc>-plan_del_qty = <lfs_demand_dc>-plan_del_qty + lw_demand-plan_del_qty.
    
    " Aggregate total by Month (for denominator - excludes DUMMY SHP already)
    READ TABLE lt_total_demand_by_month ASSIGNING FIELD-SYMBOL(<lfs_total_demand>)
      WITH KEY month = lv_demand_month.
    
    IF sy-subrc <> 0.
      APPEND INITIAL LINE TO lt_total_demand_by_month ASSIGNING <lfs_total_demand>.
      <lfs_total_demand>-month = lv_demand_month.
      <lfs_total_demand>-total_del_qty = 0.
    ENDIF.
    
    <lfs_total_demand>-total_del_qty = <lfs_total_demand>-total_del_qty + lw_demand-plan_del_qty.
  ENDIF.
ENDLOOP.

SORT lt_demand_by_dc_month BY dist_channel month.
SORT lt_total_demand_by_month BY month.
```

### 4.5 Step 4: Calculate Netback for Each DC and Month

```abap
" ============================================================
" STEP 4: Calculate Netback for each Distribution Channel and Month
" ============================================================
CLEAR: lt_output.

" Loop through each input record (each Distribution Channel)
LOOP AT it_input INTO lw_input.
  
  " Get Distribution Channel from input
  DATA(lv_input_dc) = lw_input-vtweg.  " Adjust field name as needed
  
  " Loop through each month in bucket definition
  LOOP AT et_buckdf INTO lw_bucket.
    
    DATA(lv_calc_month) = lw_bucket-date_mythr.
    
    " Get Production Quantity for this month
    READ TABLE lt_promo_by_month INTO DATA(lw_promo_month)
      WITH KEY month = lv_calc_month.
    
    IF sy-subrc <> 0.
      " No production data for this month - skip or set to zero
      CONTINUE.
    ENDIF.
    
    " Get Planned Delivery Quantity for this DC and month
    READ TABLE lt_demand_by_dc_month INTO DATA(lw_demand_dc)
      WITH KEY dist_channel = lv_input_dc
               month = lv_calc_month.
    
    IF sy-subrc <> 0.
      " No demand data for this DC and month - set to zero
      CLEAR lw_demand_dc.
      lw_demand_dc-plan_del_qty = 0.
    ENDIF.
    
    " Get Total Planned Delivery Quantity (excluding DUMMY SHP) for this month
    READ TABLE lt_total_demand_by_month INTO DATA(lw_total_demand)
      WITH KEY month = lv_calc_month.
    
    IF sy-subrc <> 0 OR lw_total_demand-total_del_qty = 0.
      " No total demand or zero - cannot calculate, skip
      CONTINUE.
    ENDIF.
    
    " ============================================================
    " CALCULATE NETBACK QUANTITY using the formula:
    " NTBK_QTY = (Prod Qty) × (DC Planned Del Qty) ÷ (Total Planned Del Qty excl DUMMY SHP)
    " ============================================================
    CLEAR: lw_output.
    
    " Copy all fields from input
    MOVE-CORRESPONDING lw_input TO lw_output.
    
    " Set month from bucket definition
    lw_output-ntbk_mth = lv_calc_month.
    
    " Calculate netback quantity
    DATA(lv_numerator) = lw_promo_month-prod_qty * lw_demand_dc-plan_del_qty.
    DATA(lv_denominator) = lw_total_demand-total_del_qty.
    
    IF lv_denominator <> 0.
      lw_output-ntbk_qty = lv_numerator / lv_denominator.
    ELSE.
      lw_output-ntbk_qty = 0.
    ENDIF.
    
    " Set business rule identifier
    lw_output-business_rule = 'TOTAL INPUT RM QTY REQUIRED TO PRODUCE FG(EXDSHP)'.
    
    " Set other required fields
    lw_output-ntbk_index = lw_input-ntbk_index.
    " ... set other fields as per ZAPO_NTBK_DETAIL structure ...
    
    " Append to output table
    APPEND lw_output TO lt_output.
    
  ENDLOOP. " End of bucket definition loop
  
ENDLOOP. " End of input loop

" Move to export parameter
et_output = lt_output.
```

### 4.6 Complete Corrected Code Structure

```abap
FUNCTION ZAPO_NTBK_DMD_DSHP_EXCLUDE.
*"----------------------------------------------------------------------
*"*"Local Interface:
*"  IMPORTING
*"     VALUE(IT_INPUT) TYPE ZAPO_NTBK_REF_T
*"     VALUE(IT_PROMO) TYPE STANDARD TABLE OF /SAPAPO/SNPOPPRO
*"     VALUE(IT_DEMAND) TYPE STANDARD TABLE OF /SAPAPO/SNPOPDMN
*"     VALUE(ET_BUCKDF) TYPE STANDARD TABLE OF /SAPAPO/SNPOPBCT
*"  EXPORTING
*"     VALUE(ET_OUTPUT) TYPE ZAPO_NTBK_DETAIL_T
*"     VALUE(ET_RETURN) TYPE BAPIRET2_T
*"----------------------------------------------------------------------

  " Data declarations (as shown in section 4.1)
  " ... 

  " Step 1: Read DUMMY SHP exclusion list
  " ... (code from section 4.2)

  " Step 2: Prepare Production Quantity by Month
  " ... (code from section 4.3)

  " Step 3: Prepare Planned Delivery Quantity by DC and Month
  " ... (code from section 4.4)

  " Step 4: Calculate Netback for Each DC and Month
  " ... (code from section 4.5)

  " Error handling and return messages
  IF et_output IS INITIAL.
    " Add warning message
    APPEND INITIAL LINE TO et_return ASSIGNING FIELD-SYMBOL(<lfs_return>).
    <lfs_return>-type = 'W'.
    <lfs_return>-id = 'ZAPO_NTBK'.
    <lfs_return>-number = '001'.
    <lfs_return>-message_v1 = 'No output generated'.
  ENDIF.

ENDFUNCTION.
```

---

## 5. Key Changes Summary

### 5.1 Changes Required

1. **Multiple Month Processing**:
   - Add loop through `ET_BUCKDF` (bucket definition) to process each month separately
   - Generate separate output record for each month per Distribution Channel

2. **Formula Implementation**:
   - Read Production Quantity from `IT_PROMO` aggregated by month
   - Read Planned Delivery Quantity from `IT_DEMAND` aggregated by DC and month
   - Calculate Total Planned Delivery Quantity excluding DUMMY SHP
   - Apply formula: `(Prod Qty × DC Planned Del Qty) ÷ Total Planned Del Qty (excl DUMMY SHP)`

3. **Distribution Channel Separation**:
   - Process each Distribution Channel from `ZAPO_NTBK_REF` separately
   - Calculate DC-specific Planned Delivery Quantity for numerator

4. **DUMMY SHP Exclusion**:
   - Read exclusion list from `ZAPOPARAM` table
   - Filter out DUMMY SHP shipments when calculating Total Planned Delivery Quantity

5. **Output Structure**:
   - Use `APPEND` instead of `COLLECT` to preserve all monthly records
   - Ensure `NTBK_MTH` is set correctly for each output record
   - Ensure all key fields from input are copied to each output record

---

## 6. Testing Approach

### 6.1 Test Scenarios

1. **TS1 - Multiple Months, Single DC**:
   - Setup: One Distribution Channel in `ZAPO_NTBK_REF`, 3 months in bucket definition
   - Expected: 3 separate records in `ZAPO_NTBK_DETAIL`, one per month

2. **TS2 - Multiple Months, Multiple DCs**:
   - Setup: 2 Distribution Channels, 3 months in bucket definition
   - Expected: 6 separate records (2 DCs × 3 months)

3. **TS3 - DUMMY SHP Exclusion**:
   - Setup: Demand data with DUMMY SHP shipments
   - Expected: DUMMY SHP excluded from denominator calculation

4. **TS4 - Formula Verification**:
   - Setup: Known Production Qty, DC Planned Del Qty, Total Planned Del Qty
   - Expected: Calculated `NTBK_QTY` matches expected formula result

5. **TS5 - Zero/Empty Data Handling**:
   - Setup: Missing production or demand data for some months
   - Expected: Appropriate handling (skip or set to zero) without errors

### 6.2 Validation Checklist

- [ ] Multiple month output generated for each Distribution Channel
- [ ] Formula calculation matches business requirement
- [ ] DUMMY SHP shipments excluded from denominator
- [ ] Each output record has correct `NTBK_MTH`
- [ ] Each output record has correct `NTBK_QTY` based on formula
- [ ] All key fields from `ZAPO_NTBK_REF` preserved in output
- [ ] No data loss or incorrect aggregation
- [ ] Performance acceptable for large datasets

---

## 7. Field Name Mapping Notes

**Important**: The actual field names in the ABAP code may differ from the examples above. Please verify and adjust:

- Production Quantity field in `/SAPAPO/SNPOPPRO`: May be `prod_qty`, `quantity`, `qty`, etc.
- Planned Delivery Quantity field in `/SAPAPO/SNPOPDMN`: May be `plan_del_qty`, `delivery_qty`, `qty`, etc.
- Distribution Channel field: May be `vtweg`, `dist_channel`, `dc`, etc.
- Shipment Type field for DUMMY SHP: May be `shipment_type`, `shp_type`, `type`, etc.
- Month field in bucket definition: May be `date_mythr`, `month`, `period`, etc.
- Bucket field: May be `bckto`, `bucket`, `period`, etc.

**Action Required**: Review the actual DDIC structures and adjust field names accordingly.

---

## 8. Implementation Priority

| Priority | Task | Impact |
|----------|------|--------|
| **P0** | Implement multiple month loop per DC | Critical - Core requirement |
| **P0** | Implement correct formula calculation | Critical - Business logic |
| **P0** | Implement DUMMY SHP exclusion | Critical - Data accuracy |
| **P1** | Add error handling and validation | High - Stability |
| **P1** | Optimize performance for large datasets | High - Usability |

---

## 9. Expected Results

### 9.1 Before Correction
- Single output record per Distribution Channel (or aggregated)
- Incorrect or missing formula implementation
- DUMMY SHP may not be excluded

### 9.2 After Correction
- Multiple output records per Distribution Channel (one per month)
- Correct formula: `(Prod Qty × DC Planned Del Qty) ÷ Total Planned Del Qty (excl DUMMY SHP)`
- DUMMY SHP properly excluded from denominator
- All months from bucket definition represented in output

---

## 10. Notes and Considerations

1. **Performance**: For large datasets, consider:
   - Indexing on key fields for faster lookups
   - Processing in chunks if memory is a concern
   - Using sorted tables for binary search

2. **Data Consistency**: Ensure:
   - Bucket definitions are consistent across all input data
   - Month formats are consistent (YYYYMM, date, etc.)
   - Distribution Channel values match between `ZAPO_NTBK_REF` and `IT_DEMAND`

3. **Error Handling**: Add appropriate:
   - Division by zero checks
   - Missing data handling
   - Logging for debugging

4. **Maintenance**: Document:
   - Formula logic clearly
   - Field mappings
   - Business rule assumptions

---

**Document Version**: 1.0  
**Created**: Based on User Requirements  
**Status**: Ready for Code Review and Implementation

