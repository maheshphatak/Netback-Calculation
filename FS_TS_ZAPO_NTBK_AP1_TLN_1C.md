## Functional Specification (FS) – T-Lane Transportation Cost (1C) for Netback

### 1. Background
- **Object**: Function module `ZAPO_NTBK_AP1_TLN_1C`.
- **Purpose**: Calculate and provide Transportation Lane (T-Lane) cost details for Netback, based on APO SNP T-Lane information and publish it via output table `ZAPO_NTBK_DETAIL` (type `ZAPO_NTBK_DETAIL_T`).
- **Reference Table**: `/SAPAPO/SNPOPIAM` (source for T-Lane / means of transport related cost determination, as per business design).
- **High-Level Flow**:
  - `ZAPO_NTBK_AP1_TLN_1C` calls `ZAPO_NTBK_AP1_TLN_C` to build an internal table `LT_TLN`.
  - For each row in input `IT_INPUT` (`ZAPO_NTBK_REF_T`), the FM maps rows from `LT_TLN` to `ET_OUTPUT` (`ZAPO_NTBK_DETAIL_T`).
  - Monthly values are published using month bucket `DATE_MYTHR` into `NTBK_MTH`.

### 2. Problem Statement
- **Issue**: Output table `ZAPO_NTBK_DETAIL` shows **inconsistent / incorrect aggregation** for **different products** maintained in input table `ZAPO_NTBK_REF` in the **same month**.
- **Incorrect Behaviour**:
  - Different products (and/or different transport lane attributes such as means of transport, source, destination) maintained in `ZAPO_NTBK_REF` for the same month are **wrongly summed together** in `ZAPO_NTBK_DETAIL`.
- **Expected Behaviour**:
  - `ZAPO_NTBK_DETAIL` must contain **separate records** per unique combination (at least: index/product/source/destination/means of transport/month, as per business granularity), and must **not** sum costs across different products/MOT for the same month.

### 3. Root Cause (Functional View)
- The FM uses `COLLECT` to aggregate rows into an intermediate table (`LT_OUTPUT`).
- During collection, the field `NTBK_INDEX` is **cleared in the work area** before aggregation, and therefore the aggregation key becomes incomplete.
- As a result, records belonging to different `NTBK_INDEX` (and therefore different products / lanes) can be treated as the same key during aggregation, causing **wrong summation** across products/MOT for the same month.

### 4. Business Requirement
- For each row in `IT_INPUT` (`ZAPO_NTBK_REF`):
  - Derive transportation lane cost/quantity for each month bucket present in the APO result (`LT_TLN`).
  - Publish **one output row per month per unique lane/product combination**, without mixing different products or transport lane attributes.
- If aggregation is required, it must be done **only within the correct key granularity**, aligned with `/SAPAPO/SNPOPIAM` and the intended Netback reporting key.

### 5. Scope
- **In Scope**:
  - Changes within `ZAPO_NTBK_AP1_TLN_1C` to ensure output aggregation does not mix different products / lanes in the same month.
  - Any necessary change to internal aggregation approach (`COLLECT` vs `APPEND`) in this FM only.
- **Out of Scope**:
  - Structural changes to table `ZAPO_NTBK_DETAIL`.
  - Any change in `ZAPO_NTBK_AP1_TLN_C` logic or APO extraction logic unless required for key completeness.

### 6. Functional Solution Overview
- Ensure the aggregation key always contains the correct `NTBK_INDEX` (and other relevant identifying fields).
- Prefer generating output lines using `APPEND` (no implicit summation), or if aggregation is required, ensure the internal table key includes the correct discriminating fields so that only intended records are summed.

---

## Technical Specification (TS) – T-Lane Transportation Cost (1C) for Netback

### 1. Objects Impacted
- **Function Module**: `ZAPO_NTBK_AP1_TLN_1C`
  - Area: Construction of intermediate output table and `ET_OUTPUT`.
- **Called FM**: `ZAPO_NTBK_AP1_TLN_C` (no changes assumed; provides `ET_TLN` / `LT_TLN`).
- **Tables / Structures (Read / Write)**:
  - Input: `IT_INPUT` (type `ZAPO_NTBK_REF_T` / `ZAPO_NTBK_REF`)
  - Intermediate: `LT_TLN` (type `ZAPO_NTBK_TLN_T` from `ZAPO_NTBK_AP1_TLN_C`)
  - Output: `ET_OUTPUT` (type `ZAPO_NTBK_DETAIL_T` / `ZAPO_NTBK_DETAIL`)
  - Reference: `/SAPAPO/SNPOPIAM`

### 2. Detailed Technical Changes

#### 2.1 Current Implementation (Issue Area)
- Current pattern (simplified from FM logic):

```abap
MOVE-CORRESPONDING <lfs_input> TO lw_output.
CLEAR: lw_output-ntbk_index, lw_index.
CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
  EXPORTING input  = <lfs_input>-ntbk_index
  IMPORTING output = lw_index.

LOOP AT lt_tln ASSIGNING <lfs_tln> WHERE index = lw_index.
  lw_output-ntbk_mth  = <lfs_tln>-date_mythr.
  lw_output-ntbk_qty  = <lfs_tln>-trans.
  lw_output-ntbk_cost = <lfs_tln>-eff_cost.
  COLLECT lw_output INTO lt_output.
ENDLOOP.
```

- **Technical Issue**:
  - `LW_OUTPUT-NTBK_INDEX` is cleared before `COLLECT`, so aggregation happens with an incomplete/incorrect key.
  - `COLLECT` implicitly sums numeric fields for entries with the same key of the line type (`ZAPO_NTBK_DETAIL`), which can lead to summation across different products/lane attributes if the key is not aligned to required granularity.

#### 2.2 Proposed Implementation (Correction)
- **Option A (Recommended / Safest): Remove implicit aggregation**
  - Set `LW_OUTPUT-NTBK_INDEX` to the converted index and use `APPEND` instead of `COLLECT`.
  - This prevents summing different products/MOT together and preserves record-level detail.

```abap
MOVE-CORRESPONDING <lfs_input> TO lw_output.

CLEAR lw_index.
CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
  EXPORTING input  = <lfs_input>-ntbk_index
  IMPORTING output = lw_index.

lw_output-ntbk_index  = lw_index.
lw_output-sessionname = is_session-sessionname.

LOOP AT lt_tln ASSIGNING <lfs_tln> WHERE index = lw_index.
  lw_output-ntbk_mth  = <lfs_tln>-date_mythr.
  lw_output-ntbk_qty  = <lfs_tln>-trans.
  lw_output-ntbk_cost = <lfs_tln>-eff_cost.
  APPEND lw_output TO lt_output.
ENDLOOP.
```

- **Option B (Only if aggregation is required by design): Keep `COLLECT` but fix key completeness**
  - Ensure `LW_OUTPUT-NTBK_INDEX` (and any other key-discriminating fields required by reporting granularity) are populated before `COLLECT`.
  - Additionally ensure the internal table line type key aligns to required granularity (DDIC key of `ZAPO_NTBK_DETAIL`), otherwise `COLLECT` will still over-aggregate.

### 3. Data and Performance Considerations
- **Data Volume**:
  - Option A may increase number of output rows (one per month per lane/product combination), which is functionally correct and prevents unintended summation.
- **Performance**:
  - No additional DB reads introduced; change is limited to internal table processing.

### 4. Error Handling and Logging
- No changes proposed to `ET_RETURN` behaviour.

### 5. Testing Approach

#### 5.1 Test Scenarios
- **TS1 – Two different products, same month**
  - Maintain `ZAPO_NTBK_REF` with two products mapped to the same month and same source/destination but different indices.
  - Expected:
    - `ZAPO_NTBK_DETAIL` shows two separate rows, each with correct cost and no cross-summation.

- **TS2 – Same product, different means of transport (MOT), same month**
  - Maintain input such that MOT differs (as per `/SAPAPO/SNPOPIAM` lane definition).
  - Expected:
    - Output rows are separated per MOT and not summed together.

- **TS3 – Multiple months for same product**
  - Maintain data across multiple `DATE_MYTHR`.
  - Expected:
    - One row per month with correct monthly values.

#### 5.2 Regression Testing
- Validate `ZAPO_NTBK_DETAIL` consumers (reports/interfaces/O2C platform) to confirm they correctly handle the intended row-level granularity.


