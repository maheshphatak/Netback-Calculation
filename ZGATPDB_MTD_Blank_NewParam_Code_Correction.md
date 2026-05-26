# ZGATPDB — Month To Date popup blank for new param

**System checked:** MVP-SAP-AD1 / `DEVSCMAD1`  
**Program:** `ZAPO_GATP_ALLOCATION_REPORT`  
**Include:** `ZAPO_GATP_ALLOCATION_F005`  
**Method:** `pull_data_prime_buck`  
**Output affected:** Month To Date section in the MT/NT report / mail layout  
**Version:** 1.0 — 26/05/2026

---

## 1. Symptom

In the Month To Date section, all values are blank / zero for **new param** although `ZAPO_PRIME_BCKU` has valid rows for the month.

From your `ZAPO_PRIME_BCKU` screenshot, the month totals are:

| Field | Expected MTD total |
|------|--------------------|
| `INC_ORD_QUAN` | **372** |
| `MANUAL_TOUCH` | **99** |
| `NO_TOUCH` | **372** |

But the MTD section shows **0 / blank**.

Current Day is still populated correctly.

---

## 2. Root cause in AD1 code

### 2.1 `pull_data_prime_buck` fills base month records correctly

The method reads month rows from `ZAPO_PRIME_BCKU`:

```abap
SELECT *
  FROM zapo_prime_bcku
  INTO TABLE gt_prime_buk
  WHERE zdate IN so_date
    AND div   IN so_divi.
```

Then it builds `gt_prime` with daily values:

```abap
LOOP AT gt_prime_buk INTO gw_prime_buk.
  gw_prime-zdate        = gw_prime_buk-zdate.
  gw_prime-loc_type     = gw_prime_buk-location_type.
  gw_prime-div          = gw_prime_buk-div.
  gw_prime-manual_touch = gw_prime_buk-manual_touch.
  gw_prime-no_touch     = gw_prime_buk-no_touch.
  COLLECT gw_prime INTO gt_prime.
ENDLOOP.
```

So far, the table has the daily month rows.

### 2.2 MTD totals are only built in the **old-param** branch

Immediately after that, the code checks:

```abap
READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'.
IF sy-subrc <> 0.
  " old param logic only
  ...
  <lfs_prime>-total_mt = ...
  <lfs_prime>-total_nt = ...
  <lfs_prime>-total_div = ...
  MODIFY gt_prime ...
ENDIF.
```

This means:

- **Old param** (`sy-subrc <> 0`) -> totals are populated
- **New param** (`sy-subrc = 0`) -> this block is skipped completely

Therefore for new param:

- `gt_prime-manual_touch` and `gt_prime-no_touch` exist for daily rows
- but **`gt_prime-total_mt` / `total_nt` / `total_div` remain initial**

### 2.3 MTD renderer reads only `total_*` fields

In the HTML / mail builder, the Month To Date section does **not** read `manual_touch` / `no_touch`.
It reads:

```abap
gw_prime-total_mt
gw_prime-total_nt
gw_prime-total_div
```

Example:

```abap
READ TABLE gt_prime INTO gw_prime
  WITH KEY div = '23' loc_type = 'Plant'(258).

lw_allo-pp_qty = gw_prime-total_mt.   " MT MTD
...
lw_allo-pp_qty = gw_prime-total_nt.   " NT MTD
```

Since those `total_*` fields are blank for new param, the whole Month To Date section is blank.

---

## 3. Why this matches your screenshot exactly

Your `ZAPO_PRIME_BCKU` screenshot already has month totals in the table:

- Incoming Qty = **372**
- Manual Touch = **99**
- No Touch = **372**

But `pull_data_prime_buck` never transfers these month totals into `gt_prime-total_mt/total_nt/total_div` for the new-param path.

So the MTD renderer has nothing to display.

---

## 4. Correct fix

For **new param**, Month To Date should be built **directly from `ZAPO_PRIME_BCKU` monthly sums**, not from the old `gt_apo_so_list` add-on logic.

### Required totals

For each `DIV + LOCATION_TYPE` combination:

```text
total_div = SUM( inc_ord_quan )
total_mt  = SUM( manual_touch )
total_nt  = SUM( no_touch )
```

Using your screenshot example:

```text
Incoming Qty  = 372
Manual Touch  =  99
No Touch Qty  = 372
```

These values must be written into:

- `gt_prime-total_div`
- `gt_prime-total_mt`
- `gt_prime-total_nt`
- `gt_prime-dp_mt_total`
- `gt_prime-dp_nt_total`

so the MTD output can read them.

---

## 5. Paste-ready code change

### 5.1 Add local type/data in `pull_data_prime_buck`

Add after existing DATA declarations at start of method:

```abap
    TYPES: BEGIN OF lty_prime_month,
             div       TYPE /bic/oip_div,
             loc_type  TYPE zloc_type,
             total_div TYPE zno_touch,
             total_mt  TYPE zno_touch,
             total_nt  TYPE zno_touch,
           END OF lty_prime_month.

    DATA: lt_prime_month TYPE TABLE OF lty_prime_month,
          lw_prime_month TYPE lty_prime_month.
```

---

### 5.2 Replace the `READ TABLE gt_zapoparam ...` block with old+new handling

**Current code:**

```abap
    READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'. " less records
    IF sy-subrc <> 0.

      lt_apo_so_list_mt   =  gt_apo_so_list .
      lt_apo_so_list_nt   = gt_apo_so_list.

      DELETE : lt_apo_so_list_mt WHERE req_mt NE 'KL' ,
               lt_apo_so_list_nt WHERE req_nt NE 'KL'.

      SORT gt_prime_buk BY zdate div .
      SORT lt_apo_so_list_mt BY spart edatu .
      SORT lt_apo_so_list_nt BY spart edatu .

      LOOP AT gt_prime ASSIGNING <lfs_prime> .
        ...
        <lfs_prime>-total_mt = <lfs_prime>-dp_mt_total = lw_div_mt .
        <lfs_prime>-total_nt = <lfs_prime>-dp_nt_total = lw_div_nt.
        <lfs_prime>-total_div =  lw_div_mt + lw_div_nt.
        MODIFY gt_prime FROM <lfs_prime> TRANSPORTING total_div total_mt total_nt dp_mt_total dp_nt_total
          WHERE div = <lfs_prime>-div AND loc_type = <lfs_prime>-loc_type .
        ...
      ENDLOOP.

    ENDIF.
```

**Corrected code:**

```abap
    READ TABLE gt_zapoparam TRANSPORTING NO FIELDS WITH KEY param1 = 'GATP'. " less records
    IF sy-subrc <> 0.

      " Old param logic - keep as is
      lt_apo_so_list_mt   =  gt_apo_so_list .
      lt_apo_so_list_nt   = gt_apo_so_list.

      DELETE : lt_apo_so_list_mt WHERE req_mt NE 'KL' ,
               lt_apo_so_list_nt WHERE req_nt NE 'KL'.

      SORT gt_prime_buk BY zdate div .
      SORT lt_apo_so_list_mt BY spart edatu .
      SORT lt_apo_so_list_nt BY spart edatu .

      LOOP AT gt_prime ASSIGNING <lfs_prime> .
        AT END OF loc_type.
          lw_loc = 'X'.
        ENDAT.

        LOOP AT lt_apo_so_list_mt INTO lw_apo_so_list_nt WHERE spart = <lfs_prime>-div
                                                           AND edatu = <lfs_prime>-zdate.
          READ TABLE gt_loc_type_sto INTO gw_loc_type_sto WITH KEY loc = lw_apo_so_list_nt-werks BINARY SEARCH.
          IF sy-subrc EQ 0.
            IF gw_loc_type_sto-loc_type = '1001'.
              lw_mt_qty_plant =  lw_mt_qty_plant +  lw_apo_so_list_nt-bmeng.
            ELSEIF gw_loc_type_sto-loc_type = '1002'.
              lw_mt_qty_depot =  lw_mt_qty_depot +  lw_apo_so_list_nt-bmeng.
            ENDIF.
          ENDIF.
        ENDLOOP.

        CLEAR lw_apo_so_list_nt.
        LOOP AT lt_apo_so_list_nt INTO lw_apo_so_list_nt WHERE spart = <lfs_prime>-div
                                                           AND edatu = <lfs_prime>-zdate.
          READ TABLE gt_loc_type_sto INTO gw_loc_type_sto WITH KEY loc = lw_apo_so_list_nt-werks BINARY SEARCH.
          IF sy-subrc EQ 0.
            IF gw_loc_type_sto-loc_type = '1001'.
              lw_nt_qty_plant =  lw_nt_qty_plant +  lw_apo_so_list_nt-bmeng.
            ELSEIF gw_loc_type_sto-loc_type = '1002'.
              lw_nt_qty_depot =  lw_nt_qty_depot +  lw_apo_so_list_nt-bmeng.
            ENDIF.
          ENDIF.
        ENDLOOP.

        IF <lfs_prime>-loc_type = 'Plant'(258).
          <lfs_prime>-no_touch     =  <lfs_prime>-no_touch     + lw_nt_qty_plant.
          <lfs_prime>-manual_touch =  <lfs_prime>-manual_touch + lw_mt_qty_plant.
        ELSEIF <lfs_prime>-loc_type = 'Depot'(259).
          <lfs_prime>-no_touch     =  <lfs_prime>-no_touch     + lw_nt_qty_depot.
          <lfs_prime>-manual_touch =  <lfs_prime>-manual_touch + lw_mt_qty_depot.
        ENDIF.

        <lfs_prime>-total =   <lfs_prime>-no_touch  +  <lfs_prime>-manual_touch.
        lw_div_qty =  lw_div_qty +  <lfs_prime>-total .
        lw_div_nt =  lw_div_nt +  <lfs_prime>-no_touch .
        lw_div_mt =  lw_div_mt +  <lfs_prime>-manual_touch .
        CLEAR: lw_nt_qty_plant, lw_nt_qty_depot , lw_mt_qty_plant ,  lw_mt_qty_depot.

        IF lw_loc IS NOT INITIAL.
          CLEAR lw_loc.
          <lfs_prime>-total_mt = <lfs_prime>-dp_mt_total = lw_div_mt .
          <lfs_prime>-total_nt = <lfs_prime>-dp_nt_total = lw_div_nt.
          <lfs_prime>-total_div =  lw_div_mt + lw_div_nt.
          MODIFY gt_prime FROM <lfs_prime> TRANSPORTING total_div total_mt total_nt dp_mt_total dp_nt_total
            WHERE div = <lfs_prime>-div AND loc_type = <lfs_prime>-loc_type .
          CLEAR : lw_div_qty,  lw_div_mt , lw_div_nt .
        ENDIF.
      ENDLOOP.

    ELSE.

      " New param logic - build Month To Date directly from ZAPO_PRIME_BCKU
      CLEAR lt_prime_month.

      LOOP AT gt_prime_buk INTO gw_prime_buk.
        CLEAR lw_prime_month.
        lw_prime_month-div       = gw_prime_buk-div.
        lw_prime_month-loc_type  = gw_prime_buk-location_type.
        lw_prime_month-total_div = gw_prime_buk-inc_ord_quan.
        lw_prime_month-total_mt  = gw_prime_buk-manual_touch.
        lw_prime_month-total_nt  = gw_prime_buk-no_touch.
        COLLECT lw_prime_month INTO lt_prime_month.
      ENDLOOP.

      LOOP AT lt_prime_month INTO lw_prime_month.
        CLEAR gw_prime.
        gw_prime-div        = lw_prime_month-div.
        gw_prime-loc_type   = lw_prime_month-loc_type.
        gw_prime-total_div  = lw_prime_month-total_div.
        gw_prime-total_mt   = lw_prime_month-total_mt.
        gw_prime-total_nt   = lw_prime_month-total_nt.
        gw_prime-dp_mt_total = lw_prime_month-total_mt.
        gw_prime-dp_nt_total = lw_prime_month-total_nt.

        MODIFY gt_prime FROM gw_prime
          TRANSPORTING total_div total_mt total_nt dp_mt_total dp_nt_total
          WHERE div = lw_prime_month-div
            AND loc_type = lw_prime_month-loc_type.
      ENDLOOP.

    ENDIF.
```

---

## 6. Why this fix works

For **new param**:

- month rows already exist in `gt_prime_buk`
- now we explicitly aggregate:
  - `inc_ord_quan` -> `total_div`
  - `manual_touch` -> `total_mt`
  - `no_touch` -> `total_nt`
- these values are copied into `gt_prime`
- the MTD renderer now reads valid totals and displays them

So the Month To Date section will no longer be blank.

---

## 7. Expected result after fix

From the screenshot totals in `ZAPO_PRIME_BCKU`:

| MTD field | Expected |
|----------|----------|
| Incoming Quantity | **372** |
| Manual Touch Order Quantity | **99** |
| No Touch Order Quantity | **372** |

These values will now be available to the MTD report logic through:

- `gw_prime-total_div = 372`
- `gw_prime-total_mt = 99`
- `gw_prime-total_nt = 372`

---

## 8. Test plan

| Test | Expected result |
|------|-----------------|
| Open MTD output with old params | No change |
| Open MTD output with new param | Month To Date section populated |
| Sum `ZAPO_PRIME_BCKU-INC_ORD_QUAN` for month | Matches `gt_prime-total_div` |
| Sum `ZAPO_PRIME_BCKU-MANUAL_TOUCH` for month | Matches `gt_prime-total_mt` |
| Sum `ZAPO_PRIME_BCKU-NO_TOUCH` for month | Matches `gt_prime-total_nt` |

---

## 9. Notes

1. This is a **reporting-population** issue, not a database-read issue.
2. `ZAPO_PRIME_BCKU` already contains the month data.
3. The blank MTD happens because totals are not assigned in the **new-param** branch.
4. If desired later, the same new-param branch can also calculate MTD percentages more precisely from `total_div`.

---

*Reference checked from AD1 include `ZAPO_GATP_ALLOCATION_F005`: `pull_data_prime_buck` (~3679 onward) and MTD renderer in `send_mail` / HTML builder (~4168 onward).*
