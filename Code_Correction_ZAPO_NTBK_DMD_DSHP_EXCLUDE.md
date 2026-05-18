## Code Correction: Arithmetic Overflow in `ZAPO_NTBK_PROD_DSHP_EXCLUDE`

### 1. Context
- **Program**: `SAPLZAPO_NTBK`  
- **Include**: `LZAPO_NTBKU24`  
- **Function Module**: `ZAPO_NTBK_PROD_DSHP_EXCLUDE`  
- **Dump**: `COMPUTE_BCD_OVERFLOW` / `CX_SY_ARITHMETIC_OVERFLOW`  
- **Termination Line (ST22)**: Line 527 in include `LZAPO_NTBKU24`

The short dump shows the failure in the `WHEN OTHERS` branch of a `CASE` statement, in the "non domestic and non exports scenario":

- Safe branch (no dump):
  - `lv_demand_plus_promo_qty = ( lv_export_qty - lv_dummy_qty ...`
- Failing branch:
  - `lv_demand_plus_promo_qty = ( lv_non_domes_export_qty  * lw_output-...`

The multiplication with large non‑domestic export quantities causes an **overflow on type P**.

---

### 2. Root Cause

In the `WHEN OTHERS` case, the code multiplies `lv_non_domes_export_qty` by a factor taken from `lw_output-*`.  
For certain data combinations this product no longer fits into the packed number (`TYPE p`) used for `lv_demand_plus_promo_qty`, leading to:

- Runtime error: **`COMPUTE_BCD_OVERFLOW`**
- Exception: **`CX_SY_ARITHMETIC_OVERFLOW`**

---

### 3. Recommended Code Change (Logic-Level Fix)

#### 3.1. Problematic Code Pattern (ST22 extract)

```abap
* CASE ... " previous code and other WHEN branches
  WHEN OTHERS. " non domestic and non exports scenario / added

    READ TABLE lt_/sapapo/loc_1 TRANSPORTING NO FIELDS
         WITH KEY ... " (location key)
         .

    IF sy-subrc = 0.
      lv_demand_plus_promo_qty = ( lv_non_domes_export_qty - lv_dummy_qty ).
    ELSE.
      " >>> Overflow originates here
      lv_demand_plus_promo_qty = ( lv_non_domes_export_qty * lw_output-... ).
    ENDIF.

    lv_demand_plus_dummy_qty = ( lv_demand_qty - lv_dummy_qty ).
    IF lv_demand_plus_dummy_qty NE 0.
      lw_output1-ntbk_qty = lv_demand_plus_promo_qty / lv_demand_plus_dummy_qty.
    ENDIF.
* ENDCASE.
```

> Note: The field names in the `WITH KEY` and after `lw_output-` are truncated in the dump; keep them as in your existing code.

#### 3.2. Corrected Code (Remove Risky Multiplication)

If business logic allows the same behavior as the earlier (domestic/export) branch, you can safely **avoid the multiplication** and thereby remove the overflow risk:

```abap
  WHEN OTHERS. " non domestic and non exports scenario / corrected

    READ TABLE lt_/sapapo/loc_1 TRANSPORTING NO FIELDS
         WITH KEY ... " keep existing key
         .

    IF sy-subrc = 0.
      " Location exists – subtract dummy quantity as in other branch
      lv_demand_plus_promo_qty = lv_non_domes_export_qty - lv_dummy_qty.
    ELSE.
      " Fallback: use non‑domestic export quantity as-is, without multiplication
      lv_demand_plus_promo_qty = lv_non_domes_export_qty.
    ENDIF.

    lv_demand_plus_dummy_qty = lv_demand_qty - lv_dummy_qty.
    IF lv_demand_plus_dummy_qty NE 0.
      lw_output1-ntbk_qty = lv_demand_plus_promo_qty / lv_demand_plus_dummy_qty.
    ENDIF.
```

**Effect:**
- Eliminates the `TYPE p` multiplication that was overflowing.
- Keeps the same ratio‑based calculation pattern used in the earlier (working) branch.
- Business behavior remains predictable: when no location entry is found, the non‑domestic export quantity is taken as‑is instead of being up‑scaled.

---

### 4. Alternative: Data Type / Numeric-Safety Fix (If Multiplication Is Mandatory)

If the multiplication is required by business logic and cannot be removed, you should:

1. **Perform the multiplication in `DECFLOAT34`**, and  
2. **Only convert back to packed number at the latest possible step** (or change the DDIC type to `DECFLOAT34`).

Example pattern:

```abap
DATA: lv_demand_plus_promo_qty_df TYPE decfloat34,
      lv_non_domes_export_qty_df  TYPE decfloat34,
      lv_factor_df                TYPE decfloat34.

lv_non_domes_export_qty_df = CONV decfloat34( lv_non_domes_export_qty ).
lv_factor_df                = CONV decfloat34( lw_output-<your_factor_field> ).

lv_demand_plus_promo_qty_df =
  lv_non_domes_export_qty_df * lv_factor_df.

" Optional: convert back into packed number if target field is TYPE p
lv_demand_plus_promo_qty = CONV #( lv_demand_plus_promo_qty_df ).
```

> Use this alternative only if changing the logic in section **3.2** is not acceptable for the business; otherwise the logic‑level fix is simpler and safer.

---

### 5. Testing Checklist

1. **Unit test / developer test**
   - Run `ZAPO_O2C_NTBK_REPORT` with the same selection parameters that previously caused the dump.
   - Confirm:
     - No short dump `COMPUTE_BCD_OVERFLOW`.
     - `ZAPO_NTBK_PROD_DSHP_EXCLUDE` completes successfully.
2. **Boundary tests**
   - Test with **very large** `lv_non_domes_export_qty` values.
   - Verify `lw_output1-ntbk_qty` is populated and no overflow occurs.
3. **Regression**
   - Compare netback values for domestic/export scenarios before vs after change to ensure there is **no unintended impact** on existing correct cases.










