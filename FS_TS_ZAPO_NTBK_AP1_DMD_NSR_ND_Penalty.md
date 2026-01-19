## Functional Specification (FS) – ND Penalty per Month for Demand Netback

### 1. Background
- **Object**: Function module `ZAPO_NTBK_AP1_DMD_NSR`.
- **Purpose**: Calculate and provide Netback / ND Penalty information for demand planning, based on SNP data (`/SAPAPO/SNPOPDCT`) and APO planning driver (`ZAPO_NTBK_PLNDRV`), and publish it via table `ZAPO_NTBK_DETAIL`.
- **Existing Logic**:
  - Reads ND Penalty (`ndpen`) per bucket (`bckto`) from `/SAPAPO/SNPOPDCT` filtered by session, material and location.
  - Maps buckets to month–year (`date_mythr`) using planning driver data.
  - Builds an internal table `lt_nsr` with one record per material / location / bucket / month, with `ndpen` and derived fields.
  - Builds `et_output` from `lt_nsr` and `it_input`, which is then used to update `ZAPO_NTBK_DETAIL`.

### 2. Problem Statement
- **Issue**: Only the **first month’s** ND Penalty is effectively published to `ZAPO_NTBK_DETAIL` for a given input combination (index), even though multiple months/buckets exist in the SNP data.
- **Root Cause**:
  - In the final loop where `et_output` is prepared, the work area `lw_out` is:
    - Initially filled with `MOVE-CORRESPONDING <lfs_input> TO lw_out` **once** per input row.
    - Then updated for each matching `lt_nsr` row and **cleared** (`CLEAR lw_out`) inside the inner loop.
  - After the first append, subsequent rows in the inner loop do not re-initialize `lw_out` from `<lfs_input>`, so key fields from the input are not consistently present for all months.
  - Downstream processing (or selection logic) then effectively recognizes only the first month’s ND Penalty as valid.

### 3. Business Requirement
- **Goal**: For each eligible input record in `IT_INPUT`, **every month’s ND Penalty** (per bucket / period present in `/SAPAPO/SNPOPDCT`) must be calculated and written to `ZAPO_NTBK_DETAIL` as **separate records**, all with:
  - Correct APO Session information.
  - Correct product / location / division / hierarchy attributes.
  - Correct month–year (`NTBK_MTH`) and quantity (`NTBK_QTY`) per period.
  - Correct ND Penalty (`NTBK_COST`) per period.
- **Expected Behaviour**:
  - For a given `NTBK_INDEX` where `lt_nsr` contains multiple periods (e.g. Jan, Feb, Mar), the output should have three distinct lines in `ZAPO_NTBK_DETAIL`, each with its own month and ND Penalty, rather than effectively only the first month.

### 4. Scope
- **In Scope**:
  - Changes only within FM `ZAPO_NTBK_AP1_DMD_NSR`.
  - Logic that constructs `et_output` from `it_input` and `lt_nsr`.
- **Out of Scope**:
  - Any change to:
    - Data model of `ZAPO_NTBK_DETAIL`.
    - SNP data extraction logic (`/SAPAPO/SNPOPDCT`).
    - Other netback FMs (e.g. `ZAPO_NTBK_AP1_NSR`) except for aligning behaviour conceptually.

### 5. Functional Solution Overview
- For each input row (`<lfs_input>` in `IT_INPUT`) and for each corresponding NSR/ND record (`<lfs_nsr>` in `LT_NSR` with matching `INDEX`):
  - Build a complete output row (`LW_OUT`) by:
    - Re-initializing and re-populating it with the input fields for **every** month.
    - Overwriting only the month-specific and ND-specific fields (`SESSIONNAME`, `NTBK_MTH`, `NTBK_QTY`, `NTBK_COST`).
  - Ensure `NTBK_INDEX` is set correctly for every row.
  - Append each row to `ET_OUTPUT` so that there is **one row per month per index**.

---

## Technical Specification (TS) – ND Penalty per Month for Demand Netback

### 1. Objects Impacted
- **Function Module**: `ZAPO_NTBK_AP1_DMD_NSR`
  - Section: Final construction of `ET_OUTPUT` from `IT_INPUT` and `LT_NSR`.
- **Tables / Structures (Read / Write)**:
  - Input:
    - `IT_INPUT` (type `ZAPO_NTBK_REF_T` / `ZAPO_NTBK_REF`).
    - `LT_NSR` (local type `LTY_NSR`).
  - Output:
    - `ET_OUTPUT` (type `ZAPO_NTBK_DETAIL_T` / `ZAPO_NTBK_DETAIL`).
    - Downstream: `ZAPO_NTBK_DETAIL` (persisted table; no structural change).

### 2. Detailed Technical Changes

#### 2.1 Current Implementation (Issue Area)
- Existing code pattern (simplified):

```abap
CLEAR lt_output.
LOOP AT it_input ASSIGNING <lfs_input>.
  MOVE-CORRESPONDING <lfs_input> TO lw_out.
  CLEAR lw_out-ntbk_index.

  LOOP AT lt_nsr ASSIGNING <lfs_nsr> WHERE index = <lfs_input>-ntbk_index.
    lw_out-sessionname = <lfs_nsr>-sessionname.
    lw_out-ntbk_mth   = <lfs_nsr>-date_mythr.
    lw_out-ntbk_qty   = <lfs_nsr>-trans.
    lw_out-ntbk_cost  = <lfs_nsr>-eff_nd.
    APPEND lw_out TO lt_output.
    CLEAR lw_out.
  ENDLOOP.

  IF lt_output IS NOT INITIAL.
    LOOP AT lt_output ASSIGNING <lfs_out>.
      <lfs_out>-ntbk_index = <lfs_input>-ntbk_index.
      APPEND <lfs_out> TO et_output.
    ENDLOOP.
    CLEAR lt_output.
  ENDIF.
ENDLOOP.
```

- **Technical Issue**:
  - `LW_OUT` is cleared within the inner loop and is **not** re-populated from `<LFS_INPUT>` for subsequent months.
  - This can lead to incomplete/incorrect records for months after the first, which then may be filtered or ignored downstream.

#### 2.2 Proposed Implementation

- **Change Approach**:
  - Move `MOVE-CORRESPONDING <lfs_input> TO lw_out` **inside** the inner loop.
  - For each `<lfs_nsr>` row:
    - `CLEAR` then `MOVE-CORRESPONDING` from input.
    - Set month and ND-specific fields.
    - Append to `lt_output`.

- **Revised Code Block**:

```abap
CLEAR lt_output.
LOOP AT it_input ASSIGNING <lfs_input>.

  LOOP AT lt_nsr ASSIGNING <lfs_nsr> WHERE index = <lfs_input>-ntbk_index.
    CLEAR lw_out.
    MOVE-CORRESPONDING <lfs_input> TO lw_out.
    CLEAR lw_out-ntbk_index.

    lw_out-sessionname = <lfs_nsr>-sessionname.
    lw_out-ntbk_mth    = <lfs_nsr>-date_mythr.
    lw_out-ntbk_qty    = <lfs_nsr>-trans.
    lw_out-ntbk_cost   = <lfs_nsr>-eff_nd.

    APPEND lw_out TO lt_output.
  ENDLOOP.

  IF lt_output IS NOT INITIAL.
    LOOP AT lt_output ASSIGNING <lfs_out>.
      <lfs_out>-ntbk_index = <lfs_input>-ntbk_index.
      APPEND <lfs_out> TO et_output.
    ENDLOOP.
    CLEAR lt_output.
  ENDIF.

ENDLOOP.
```

- **Notes**:
  - No changes to type definitions, tables, or interfaces.
  - Only the placement of `MOVE-CORRESPONDING` and `CLEAR` is modified to ensure each period row is fully populated.

### 3. Data and Performance Considerations
- **Data Volume**:
  - Number of output rows will increase in proportion to the number of ND Penalty periods per input record.
  - This is the correct functional behaviour (1 record per month / period).
- **Performance**:
  - Complexity remains \(O(n \times m)\), where:
    - \(n\) = number of input records in `IT_INPUT`.
    - \(m\) = average number of periods per input in `LT_NSR`.
  - No additional database reads are introduced; change is purely in internal processing.

### 4. Error Handling and Logging
- Existing error logging via `ET_RETURN` and messages for missing combinations in `/SAPAPO/SNPOPDCT` remains unchanged.
- No new error messages or exceptions are introduced.

### 5. Testing Approach

#### 5.1 Test Scenarios
- **TS1 – Multiple Months ND Penalty Present**
  - Setup:
    - For a given `SESSIONID`, `MATNR`, `LOCID`, maintain `/SAPAPO/SNPOPDCT` with `NDPEN > 0` for 3 consecutive buckets/months.
  - Input:
    - One entry in `IT_INPUT` with matching material / location / index.
  - Expected:
    - `LT_NSR` contains 3 records for the same index with different `DATE_Mythr`.
    - `ET_OUTPUT` and `ZAPO_NTBK_DETAIL` contain 3 rows with:
      - Same product/location/index attributes.
      - Different `NTBK_MTH`.
      - Correct `NTBK_COST` per month (matching `EFF_ND`).

- **TS2 – Single Month ND Penalty**
  - Setup:
    - `/SAPAPO/SNPOPDCT` has ND Penalty for only 1 bucket.
  - Expected:
    - Exactly 1 row in `ET_OUTPUT` / `ZAPO_NTBK_DETAIL` for that index and month.

- **TS3 – No ND Penalty Records**
  - Setup:
    - No matching `NDPEN > 0` entries in `/SAPAPO/SNPOPDCT`.
  - Expected:
    - No rows appended to `ET_OUTPUT` for that input index.
    - Appropriate error / warning entry in `ET_RETURN` (as per existing logic).

- **TS4 – Multiple Inputs, Multiple Months**
  - Setup:
    - Several entries in `IT_INPUT` for different combinations of product / location.
    - Each combination has multiple ND Penalty periods in `/SAPAPO/SNPOPDCT`.
  - Expected:
    - For each `NTBK_INDEX`, number of rows in `ET_OUTPUT` equals number of periods in `LT_NSR`.
    - All rows have correct indexing and attributes.

#### 5.2 Regression Testing
- Re-run any existing batch job or interface that consumes `ZAPO_NTBK_DETAIL` to:
  - Confirm that downstream consumers correctly handle multiple rows per index.
  - Validate that where previously only the first month was visible, now all months’ ND Penalties appear with correct values.

### 6. Transport and Deployment
- Include FM change for `ZAPO_NTBK_AP1_DMD_NSR` in a single transport request.
- Move transport through standard landscape (DEV → QA → PROD) following existing change management procedures.


