# Seed files — Quality KPI v2

Two kinds of file here. **Do not confuse them.**

| Folder | Contents | Import to production? |
|---|---|---|
| `seed/` | **Real reference data.** Tests, taxonomy, config. | ✅ Yes |
| `seed/sample/` | **Generated sample data** for development and testing. | ❌ No |

---

## `seed/` — real reference data

| File | Rows | List |
|---|---|---|
| `qk_test.csv` | 3 | `qk_test` |
| `qk_test_type.csv` | 9 | `qk_test_type` |
| `qk_defect_type.csv` | 66 | `qk_defect_type` |
| `qk_config.csv` | 6 | `qk_config` |

### Import order — parents first

```
1. qk_test           (no dependencies)
2. qk_test_type      (Test_ID → qk_test)
3. qk_defect_type    (Type_ID → qk_test_type)
4. qk_config         (no dependencies)
```

### `qk_config` values

| Setting_name | Setting_value | Used for |
|---|---|---|
| `Goal_FPY` | `90` | Status thresholds everywhere |
| `Goal_FTT` | `75` | First Time Through tile status |
| `Window_months` | `6` | How far back the app loads |
| `Home_number_pattern` | `^\d{6,8}$` | Entry format warning |
| `TTW_weeks` | `12` | Trailing-average window |
| `Edit_lock_hours` | `24` | Inspector edit window |

**No threshold literal belongs anywhere in the app.** Read from this list.

> ⚠️ **All 66 defects are seeded at `Severity = 2` (Major).** No severity was supplied in the source. The severity-weighted defect index is meaningless until Quality sets real values — most obviously, gas leaks and Hi Pot shorts are presumably Critical. The Taxonomy Admin screen is where this gets fixed.

---

## `seed/sample/` — generated sample data

For building and testing the app against something that looks real. **Delete these rows before go-live.**

| File | Rows | Notes |
|---|---|---|
| `qk_home.csv` | 1,075 | 16 weeks · 2 plants · 4 facilities |
| `qk_home_defect.csv` | 1,105 | Only against failed tests |
| `qk_week_summary.csv` | 64 | **Computed** from the two above |
| `qk_test_summary.csv` | 192 | **Computed** from `qk_home` |

### Shape

- **Weeks:** 16, ending Saturdays from 2026-05-02 to 2026-08-15.
- **Plants:** `12` with facilities 1/2/3, `21` with facility 1 only — so the Admin plant switch and the facility filter both have something to prove.
- **Homes:** 10–25 per facility-week, matching the real production rate.

### States it deliberately exercises

| State | Present |
|---|---|
| Blank result (**not tested**) | 17 test slots in the most recent week |
| Weeks below the 90% goal | 47 of 64 |
| A 3+ week below-goal run → `critical` status | longest run 11 weeks |
| `Qty > 1` on a defect line | ~20% of lines |
| Homes clean on all three tests | throughout |

### The rollups are computed, not invented

`qk_week_summary` and `qk_test_summary` were derived from the home and defect rows, then verified to reconcile exactly. This matters: seeding rollups with plausible-but-unrelated numbers would reproduce the precise bug this design exists to prevent — v1's stored TTW disagreeing with its displayed TTW.

Verified before shipping:

```
PASS  every Defect_ID exists in qk_defect_type
PASS  Defect_ID belongs to its Type_ID
PASS  Type_ID belongs to its Test_ID
PASS  Severity matches taxonomy
PASS  Home_number unique across all plants
PASS  Title unique on every list
PASS  every defect line points at a real home
PASS  no defect recorded against a passed or untested test
PASS  no duplicate Type+Defect on any home
PASS  qk_week_summary reconciles to qk_home + qk_home_defect
PASS  qk_test_summary reconciles to qk_home
PASS  Week_TTW matches the single definition, facility-filtered
```

If you regenerate this data, re-run those checks. `Week_TTW` in particular is easy to get subtly wrong — it must be **facility-filtered and anchored to the row's own week**, which is exactly what v1 got wrong.

---

## Import gotchas

**Rounding.** Every rate uses **half away from zero**, matching Power Fx `Round()`. Python's built-in `round()` uses banker's rounding and will disagree at exact `.X5` values (31.25 → 31.2 rather than 31.3). If you re-verify with a script, match Power Fx or you will chase a phantom mismatch.

**`Home_number` is Text, not a number.** Excel will happily reformat it. Set the column to Text before opening, or import without opening in Excel. The sample values have no leading zeros, so nothing is lost here — but real production numbers might.

**Dates are ISO 8601** — `YYYY-MM-DD`, and `Entered_at` is `YYYY-MM-DDTHH:MM:SS`. Unambiguous regardless of locale.

**Blank means blank.** Empty cells in `Elec_result` / `Plumb_result` / `Gas_result` mean *not tested* — a genuine third state. Do not let an import tool coerce them to `0`, which would read as *failed*.

**Files are UTF-8 with BOM** so Excel opens them correctly.

**Indexes first.** Create the indexed columns named in `CLAUDE.md` §4 **before** importing anything. `qk_home` crosses SharePoint's 5,000-item list-view threshold in roughly five weeks of real production, and retrofitting an index past that point means unloading the list.

---

## Fiscal calendar — check this

`Fiscal_year`, `Fiscal_quarter` and `Week_number` in the sample data are **calendar-year placeholders**, not Champion's real fiscal calendar. They must agree with `calendar_list`, which is the single source of truth for week boundaries and fiscal mapping.

If Champion's fiscal year doesn't start in January, regenerate the sample data from `calendar_list` rather than patching these columns by hand.
