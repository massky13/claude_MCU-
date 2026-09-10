# Quality KPI Reporting — v2 Design Spec

**Version 2.2** — home-level grain, Test → Type → Defect taxonomy (supersedes v2.1's Test → Item → Defect)
**Status:** Design. Nothing built. v1 stays in production, untouched.
**Companion:** `CLAUDE.md` (v1 as-built baseline) · `seed/` (taxonomy import files)
**Date:** August 2026

---

## 0. What v2 is

v1 records three numbers per trade per week. v2 records **every home that gets tested**, which test it failed, and exactly which defect was found.

The hierarchy is:

```
Home (production number)
  └─ Test          Electrical · Plumbing · Gas              — fixed at 3
       └─ Type         the quality procedure performed       — 9, admin-maintained
            │          "Hi Pot (Dielectric)"  QP.06A.0
            │          "Continuity"           QP.07.0
            └─ Defect     "Bonding straps not removed" …    — 66, admin-maintained
```

**Items are not tracked.** The middle level is the **type of test** — which quality procedure was run — and each type carries its **QP reference**, so every recorded defect is traceable to a documented procedure.

Every home is tested. A plant builds many homes a week, and one home can carry defects across all three tests.

Four consequences drive the rest of this document:

1. **The grain moves from week to home.** The fact table is now one row per home, not one row per week.
2. **Volume goes up roughly 20×**, so the dashboard can no longer read raw facts — it reads rollups the app maintains.
3. **A new headline metric becomes possible**: First Time Through — the share of homes that pass *all three* tests. See §5.
4. **Data entry is rebuilt** around the production number rather than around the week. See §6.3.

### Decisions taken (confirmed with Rafa)

| Question | Answer | Consequence |
|---|---|---|
| Homes tested per facility per week | **10–25** (~54/week/plant across 3 facilities) | Rollups are mandatory; raw homes load per-week only |
| What a production number identifies | **A complete home**, which may be several floors | `Floors` column on the home record preserves v1's NCs-per-floor |
| Retest tracking | **First-time result only** | One Pass/Fail per home per test. Rework is out of scope |
| Source of production numbers | **Typed manually, nothing to validate against** | Strong client-side validation; a typo is a phantom home |
| Number of tests | **Fixed at 3** | Test results are columns on the home record, not a child table (§2.3) |

---

## 1. The scale problem

**2,000 is not a row cap on a SharePoint list.** It is the ceiling on *non-delegable* queries — Power Apps pulls the first N rows and evaluates locally, silently producing wrong answers. A delegable query streams past it. The goal is therefore **every query delegable, every collection small**.

The harder ceiling is SharePoint's **5,000-item list view threshold**. Past it, a query filtering on a non-indexed column fails outright, and the rule is stricter than it looks:

> A query over the threshold only works if it filters on an **indexed column first**, and **that first filter returns no more than 5,000 items**.

Both halves bite. This is why `Plant_ID` leads every predicate in this design — a 6-month window across all plants would exceed the threshold even though `Week_key` is indexed.

### Delegation rules that constrain this design

| | Number | Text | Boolean | DateTime | Complex (Choice/Lookup/Person) |
|---|---|---|---|---|---|
| `=` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `<` `<=` `<>` `>` `>=` | ✅ | ❌ | ❌ | ✅ | ✅ |
| `Filter`, `LookUp` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Sort`, `SortByColumns` | ✅ | ✅ | ✅ | ✅ | ❌ |
| `StartsWith` | — | ✅ | — | — | ❌ on subfields |

- **`And` / `Or` delegate. `Not` does not.**
- **`Distinct`, `GroupBy`, `FirstN`, `Last`, `LastN` never delegate.**
- **`IsBlank`** is documented non-delegable on Text and Complex; treat it as non-delegable everywhere and never put it in a predicate.
- **SharePoint's `ID` column is Text underneath** — only `=` delegates on it, never `>` / `<`.
- **Aggregates (`Sum`, `Average`, `Min`, `Max`, `CountRows`) are absent from the SharePoint table.** Treat as non-delegable; run them only over loaded collections.
- Text delegates `=` but **not** range operators — which is why `Home_number` (Text) is only ever matched with `=`.

> ⚠️ **The date conflict, and how this design sidesteps it.** Microsoft's connector table lists DateTime comparisons as delegable. Microsoft's own *SharePoint delegation improvements* blog says Date/DateTime "remain non-delegable for query/filter operations." The two sources disagree and field reports are mixed.
>
> **Resolution: never filter on a date column.** Every fact list carries **`Week_key`, a Number holding `yyyymmdd`**. Number range comparison delegates unambiguously either way, sorts correctly, and indexes cleanly. `Week_end` exists for display only.

---

## 2. Data model

Ten lists: 3 taxonomy, 2 fact, 2 rollup, plus access, calendar and config.

`qk_` prefix throughout so v2 never collides with v1 — **except `sys_test_access`, which is deliberately shared** (§2.5).

### 2.1 Taxonomy — `qk_test`, `qk_test_type`, `qk_defect_type`

Seeded from `test type and defect.csv`: **3 tests, 9 types, 66 defects**. Import files are in `seed/`.

**`qk_test`** — fixed at 3, effectively read-only.

| Column | Type | Notes |
|---|---|---|
| `Test_ID` | Number | 1 Electrical · 2 Plumbing · 3 Gas |
| `Test_name` | Text | |
| `Sort_order`, `Active` | Number, Yes/No | |

**`qk_test_type`** — 9 rows. One row per quality procedure.

| Column | Type | Notes |
|---|---|---|
| `Type_ID` | Number | `Test_ID × 10 + seq` → 11–14, 21–23, 31–32 |
| `Test_ID` | Number | FK |
| `Type_name` | Text | "Hi Pot (Dielectric)", "Continuity", "Water Lines" … |
| `QP_ref` | Text | `QP.06A.0`, `QP.07.0` … — **split out of the source string, not concatenated** |
| `Sort_order`, `Active` | Number, Yes/No | `Active` is a soft delete — history references these IDs forever |

**`qk_defect_type`** — 66 rows.

| Column | Type | Notes |
|---|---|---|
| `Defect_ID` | Number | `Type_ID × 100 + seq` |
| `Type_ID` | Number | FK |
| `Test_ID` | Number | Denormalized, so "all defects for a test" needs no join |
| `Defect_name` | Text | |
| `Severity` | Number | 1 minor · 2 major · 3 critical — **seeded at 2 throughout; Quality must set real values** |
| `Sort_order`, `Active` | | |

Distribution as supplied:

| Test | Types | Defects |
|---|---|---|
| Electrical | 4 | 26 |
| Plumbing | 3 | 27 |
| Gas | 2 | 13 |

| Type | QP ref | Defects |
|---|---|---|
| Hi Pot (Dielectric) | `QP.06A.0` | 6 |
| Continuity | `QP.07.0` | 7 |
| Operational | `QP.08.0` | 8 |
| Polarity | `QP.09.0` | 5 |
| Drain Lines | `QP.03.0` | 10 |
| Water Lines | `QP.01.0` | 12 |
| Fixtures | `QP.02.0` | 5 |
| High Pressure | `QP04.0` | 9 |
| Low Pressure | `QP05.0` | 4 |

> **`QP_ref` is a separate column, not part of the name.** The source writes types as `"Continuity per QP.07.0"`. Storing that as one string would mean a procedure renumbering rewrites every type name and breaks any text matching. Split, the QP reference becomes a badge the UI shows beside the type — so an inspector recording a defect can see exactly which procedure it came from.

> **Three properties of this taxonomy that simplify the design** — all of which the previous item-based list lacked:
>
> 1. **Every one of the 66 defect names is globally unique.** Charts, tooltips, and exports can label by defect alone. (The v2.1 draft had to render `Item — Defect` everywhere because `Missing` appeared under six items. That constraint is gone.)
> 2. **Defects per type run 4 to 12** — a narrow, even spread. The Pareto granularity bias flagged in v2.1, where Gas's 9 single-defect items outranked Electrical's 21-defect grab-bag for structural rather than quality reasons, largely disappears.
> 3. **Types map one-to-one onto quality procedures.** The taxonomy is anchored to controlled documents rather than to informal part names, so it stays stable as the QPs are revised — and a defect trend points at a specific procedure.
>
> Defects must still be keyed by `Defect_ID` and scoped by `Type_ID`; uniqueness of the *display name* is a convenience, not something to design a key around.

### 2.2 Source-data decisions and remaining observations

Two corrections were applied to `test type and defect.csv` when building the seed, **both confirmed with Rafa**:

1. **Incrementing QP suffixes collapsed.** The source wrote Water Lines as `QP.01.0`, `QP.01.1` … `QP.01.11` and Fixtures as `QP.02.0` … `QP.02.4` — one suffix per defect row, while every other group held a single constant code. That is an Excel autofill artifact. Collapsed to `QP.01.0` (12 defects) and `QP.02.0` (5 defects), giving **9 types rather than 24**, one per procedure. Taken literally it would have produced 17 types with exactly one defect each, making the type picker meaningless.
2. **One exact duplicate dropped.** Gas → Low Pressure listed `Leak at pipe/valve inside Furnace` twice (source rows 65 and 67). One copy kept; Low Pressure holds 4 defects.

Still flagged for Quality's review on the admin screen:

3. **QP reference formatting is inconsistent.** Gas uses `QP04.0` and `QP05.0` — no dot after `QP` — while all seven other types use `QP.0X.0`. Harmless to the app, but it will look wrong beside a controlled document. Worth normalising in the seed before import.
4. **Trailing whitespace** on `"P-Trap in floor leak "`. The seed is trimmed; the import must stay trimmed or `=` matching against the name breaks.
5. **No severity was supplied.** All 66 seeded at 2 (Major). Gas leaks and Hi Pot shorts are presumably 3.
6. **Gas → Low Pressure may be missing a Water heater entry.** High Pressure covers Furnace / Range / Water heater / Dryer; Low Pressure covers valve / Furnace / Range / crossover. If the dropped duplicate was meant to be Water heater, add it via the admin screen.

### 2.3 Facts — `qk_home`

**One row per home.** The centre of the model.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | Natural key `PID.FAC.HomeNumber` |
| `Plant_ID` | Number | **Indexed** — leads every predicate |
| `Facility_ID` | Number | **Indexed** |
| `Week_key` | Number | `yyyymmdd` of week end. **Indexed** |
| `Week_start`, `Week_end` | Date | Display only |
| `Home_number` | Text | The production number. **Indexed** — matched with `=` only |
| `Model` | Text | Optional |
| `Floors` | Number | A home may be several floors — preserves v1's NCs-per-floor |
| `Elec_result` | Number | `1` pass · `0` fail · blank not tested |
| `Plumb_result` | Number | same |
| `Gas_result` | Number | same |
| `Fiscal_year`, `Fiscal_quarter`, `Week_number` | Number | From `calendar_list` |
| `Entered_at` | DateTime | Drives the 24-hour edit lock |
| `Entered_by` | Text | Email, for audit |
| `Locked` | Yes/No | Admin hard lock |

> **Why three result columns instead of a child table.** You've fixed the tests at 3, so a `qk_home_test` child table would triple the fastest-growing table (~84,000 rows per 6 months across all plants instead of ~28,000), add a write per test on every save, and make the weekly rollup a two-pass aggregate.
>
> **The trade is real and worth stating:** adding a fourth test later means adding a column, a migration, and touching the app — exactly the hardcoding this spec criticises elsewhere. It is accepted here *only* because the dimension is explicitly fixed. If a fourth test ever becomes plausible, switch to the child table before go-live, not after.

Results are stored as `1`/`0` Numbers rather than Yes/No so that `Sum()` over a loaded collection gives the pass count directly, and blank cleanly means "not tested" (a Boolean cannot represent three states).

### 2.4 Facts — `qk_home_defect`

**One row per home × defect type**, carrying a quantity.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | `PID.FAC.HomeNumber.DefectID` |
| `Plant_ID`, `Facility_ID`, `Week_key` | Number | All **indexed** |
| `Home_number` | Text | **Indexed** |
| `Test_ID`, `Type_ID`, `Defect_ID` | Number | FKs |
| `Qty` | Number | Default 1; >1 for repeats ("3 receptacles with reverse polarity") |
| `Severity` | Number | **Denormalized at write time** so historical severity survives taxonomy edits |
| `Notes` | Text | Optional |

Aggregate rather than itemise: three receptacles with reverse polarity is one row with `Qty: 3`, not three rows.

`Test_ID` and `Type_ID` are both stored even though `Type_ID` implies `Test_ID` — it lets "all defects for a test" filter without a join, on a list where every extra query hop matters.

### 2.5 Access — `sys_test_access`, shared with v1 and extended

Per your instruction, v2 uses **the same access table as v1** — one access list, one place to manage people.

Existing columns are untouched: `Inspector_name`, `Inspector_email`, `Plant_ID`, `Plant_name` (`Plant_ID = 0` = Corporate).

**Add two columns:**

| Column | Type | Notes |
|---|---|---|
| `Role` | Text | `Administrator` · `Quality Manager` · `Inspector` · `Reader` |
| `Active` | Yes/No | Revoke without deleting history |

> **Adding columns to a SharePoint list does not break v1.** v1 reads only `Inspector_email` and `Plant_ID` and ignores everything else. Both apps share the list safely — but the reverse is not true: **do not rename or retype existing columns**, and treat `sys_test_access` as jointly owned from now on.

Role semantics:

| Role | Scope | Can do |
|---|---|---|
| **Administrator** | All plants (`Plant_ID = 0`) | Everything, plus maintain the taxonomy and the access list |
| **Quality Manager** | One plant | Enter and edit their plant's data with no 24-hour limit; cannot touch taxonomy |
| **Inspector** | One plant | Enter data; edit only within 24 hours |
| **Reader** | One plant | View only |

Assigning Quality Managers to plants is done on the Access Admin screen (§6.6) — pick the person, pick the plant, pick the role.

### 2.6 Rollups — `qk_week_summary`, `qk_test_summary`

**Derived, not entered.** The app rebuilds them on every save (§6.4). The dashboard reads only these.

**`qk_week_summary`** — one row per plant × facility × week.

| Column | Type | Notes |
|---|---|---|
| `Title` | Text | `PID.FAC.yyyymmdd` |
| `Plant_ID`, `Facility_ID`, `Week_key` | Number | **Indexed** |
| `Week_start`, `Week_end` | Date | |
| `Homes_tested` | Number | |
| `Homes_clean` | Number | Passed all three tests |
| `FTT_pct` | Number | `Homes_clean / Homes_tested × 100` |
| `Total_tests`, `Total_pass`, `Total_fail` | Number | |
| `Week_FPY` | Number | 0–100 |
| `Week_TTW` | Number | 0–100, trailing 12 weeks |
| `Total_floors`, `Total_defects` | Number | |
| `Defects_per_floor`, `Defects_per_home` | Number | |
| `Fiscal_year`, `Fiscal_quarter`, `Week_number` | Number | |

**`qk_test_summary`** — one row per week × facility × test (3 per facility-week).

`Plant_ID`, `Facility_ID`, `Week_key` (all indexed), `Test_ID`, `Homes_tested`, `Pass`, `Fail`, `Test_FPY`.

### 2.7 Config and calendar

**`qk_config`** — `Setting_name`, `Setting_value`. Holds `Goal_FPY` (90), `Goal_FTT`, `Window_months` (6), `Home_number_pattern`. Retires every hardcoded threshold.

**`calendar_list`** — carried over unchanged, still the source of truth for week boundaries and fiscal mapping. **Add `W_key`** (Number `yyyymmdd`) so week-key derivation is a lookup, not string math.

**`Plant_List`** — unchanged.

### 2.8 Required indexes — create before loading data

| List | Indexed columns |
|---|---|
| `qk_home` | `Plant_ID`, `Week_key`, `Facility_ID`, `Home_number` |
| `qk_home_defect` | `Plant_ID`, `Week_key`, `Facility_ID`, `Home_number` |
| `qk_week_summary` | `Plant_ID`, `Week_key`, `Facility_ID` |
| `qk_test_summary` | `Plant_ID`, `Week_key`, `Facility_ID` |
| taxonomy lists | own `*_ID` and parent FK |

20 indexed columns per list is the limit — this uses at most 4. SharePoint's automatic indexing **stops at 20,000 items** and picks columns by observed usage, so it cannot be relied on. Adding an index after a list passes 5,000 items ranges from awkward to impossible (the documented workaround is *delete rows until you're under, index, reload*).

---

## 3. Volume model

At 18 homes/facility/week × 3 facilities = **54 homes/week/plant**, 26 weeks, ~20 plants.

| List | Per plant / 6 mo | All plants / 6 mo | Crosses 5,000 (all plants) | Loaded at startup? |
|---|---|---|---|---|
| `qk_week_summary` | **78** | 1,560 | ~4 years | ✅ yes |
| `qk_test_summary` | **234** | 4,680 | ~17 months | ✅ yes |
| `qk_home` | ~1,400 | ~28,000 | **~5 weeks** | ❌ per-week only |
| `qk_home_defect` | ~1,050 | ~21,000 | **~6 weeks** | ❌ on drill-in only |

Startup loads **312 rows** — the two rollups for one plant. Everything else is fetched on demand.

Note the third column. `qk_home` crosses the list view threshold in **about five weeks of production across all plants**. That is the single most important number in this document: without indexes in place from day one, the app breaks roughly a month after go-live, and fixing it then means unloading the list.

Over three years `qk_home` reaches ~170,000 items and `qk_home_defect` ~125,000. Fine for SharePoint *with* indexes and delegable predicates.

> **When to leave SharePoint.** If cross-plant benchmarking (the Application Brief's stated end goal) needs to aggregate all plants at once, or the defect table passes ~500k, move the fact lists to **Dataverse** — it delegates aggregates and has no threshold. This schema ports mechanically: no Lookup columns, no Choice columns, no calculated columns anywhere in the fact tables.

---

## 4. Loading strategy

### 4.1 The one rule

> **Delegation applies to the query against the data source, never to math over a collection.**
> Filter delegably into a small collection, then aggregate locally — `GroupBy`, `Average`, `Sum`, `FirstN` all work without limit there.

### 4.2 Startup

```
Set(varUserEmail, Lower(Trim(User().Email)));
ClearCollect(col_access, Filter(sys_test_access, Active = true));
Set(varAccessRow, LookUp(col_access, Lower(Trim(Inspector_email)) = varUserEmail));

Set(varRole, Coalesce(varAccessRow.Role, "Inspector"));
Set(varIsAdmin,   varRole = "Administrator");
Set(varIsQM,      varRole = "Quality Manager");
Set(varCanEdit,   varRole <> "Reader");

Set(varPID,
    If(!IsBlank(varAccessRow),
       varAccessRow.Plant_ID,
       Value(Left(Office365Users.UserProfileV2(User().Email).officeLocation, 3))
    )
);

// Taxonomy — 3 + 9 + 66 rows. Load once, never reload on plant change.
ClearCollect(col_test,   Filter(qk_test,        Active = true));
ClearCollect(col_type,   Filter(qk_test_type,   Active = true));
ClearCollect(col_defect, Filter(qk_defect_type, Active = true));
ClearCollect(col_calendar, calendar_list);
ClearCollect(col_config,   qk_config);
Set(varGoalFPY, Value(LookUp(col_config, Setting_name = "Goal_FPY", Setting_value)));

If(varIsAdmin, Set(varPID, Blank()), Select(btn_load_plant));
```

Administrators start with no plant selected and a "choose a plant" empty state.

### 4.3 The single load routine

Canvas apps have no GA user-defined procedure, so the routine lives in the `OnSelect` of a hidden **`btn_load_plant`** and is invoked with `Select(btn_load_plant)` from `OnStart`, the plant dropdown, and the refresh icon.

```
Set(varCutoffDate, DateAdd(Today(), -Value(LookUp(col_config,Setting_name="Window_months",Setting_value)), TimeUnit.Months));
Set(varCutoffKey,  Value(Text(varCutoffDate, "yyyymmdd")));

ClearCollect(col_week, Filter(qk_week_summary, Plant_ID = varPID && Week_key >= varCutoffKey));
ClearCollect(col_test_sum, Filter(qk_test_summary, Plant_ID = varPID && Week_key >= varCutoffKey));

Clear(col_homes); Clear(col_home_defects);
Set(varLoadedAt, Now());
```

- `Plant_ID = varPID` **first** — indexed equality leads the delegated query.
- `varCutoffKey` is a Number in a variable. `DateAdd(...)` inline in the predicate would break delegation.
- No `Sort` in the query; sort the collection afterwards.

### 4.4 Loading a week's homes (entry screen)

```
Set(varWeekKey, Value(Text(drop_week.Selected.W_end, "yyyymmdd")));
ClearCollect(col_homes,
    Filter(qk_home,
        Plant_ID = varPID && Week_key = varWeekKey && Facility_ID = varFacility
    )
);
```

Three Number equalities on indexed columns. ~18 rows.

### 4.5 Loading one home's defects

```
ClearCollect(col_home_defects,
    Filter(qk_home_defect,
        Plant_ID = varPID && Home_number = varHomeNumber
    )
);
```

`Home_number` is Text — `=` delegates, range operators do not. Never write `Home_number > …`.

### 4.6 Optional predicates — the trap

Never put optional logic inside a predicate. This looks reasonable and silently destroys delegation:

```
// ❌ WRONG — IsBlank and || force local evaluation
Filter(qk_home_defect, Plant_ID = varPID && (IsBlank(varFacility) || Facility_ID = varFacility))
```

Branch the whole `Filter` instead:

```
// ✅ RIGHT
If(IsBlank(varFacility),
   ClearCollect(col_win, Filter(qk_home_defect, Plant_ID = varPID && Week_key >= varCutoffKey)),
   ClearCollect(col_win, Filter(qk_home_defect, Plant_ID = varPID && Week_key >= varCutoffKey && Facility_ID = varFacility))
)
```

This pattern applies everywhere in the app.

---

## 5. Metrics

All rates stored **0–100**, 1 decimal. Thresholds come from `qk_config`, never literals.

**Test FPY** — per test, per facility-week:
```
Test_FPY = homes passing that test / homes tested for that test × 100
```

**Week FPY** — across all three tests:
```
Week_FPY = Σ passes / Σ tests × 100        (Σ tests = homes × 3, less any not tested)
```

**First Time Through (FTT)** — **new, and the reason home-level grain is worth the complexity:**
```
FTT = homes passing ALL THREE tests / homes tested × 100
```

> **Expect FTT to look much worse than FPY, and brief people on it before launch.** They measure different things: FPY is the share of *tests* passed, FTT the share of *homes* with nothing wrong at all. At 87% FPY, FTT lands near 66% if failures are independent — that is not a regression, it is the first honest view of how many homes leave the line clean. Show both side by side and label them clearly.

**TTW (trailing twelve weeks)** — **one definition, used by display and storage alike**, fixing v1's split:
```
TTW(week, facility) = Average(
    FirstN(SortByColumns(
        Filter(col_week, Facility_ID = facility && Week_key <= week.Week_key),
        "Week_key", SortOrder.Descending), 12),
    Week_FPY)
```
Facility-filtered and anchored to the row's week. Runs over a collection, so `FirstN`/`Average` being non-delegable is irrelevant.

**Defect rates** — `Defects_per_floor` preserves continuity with v1's NCs-per-floor; `Defects_per_home` is the new natural unit:
```
Defects_per_floor = Σ Qty / Σ Floors
Defects_per_home  = Σ Qty / Homes_tested
```

**Pareto** — share and cumulative share:
```
share_i = Σ Qty_i / Σ Qty × 100 ;  cumulative_i = Σ(share_1..i)
```
Offered at **two levels — by defect (default) and by type of test**. Defect level is the default because all 66 names are unique and the 4–12 spread per type is even enough not to bias the ranking. The type-level view answers a different and equally useful question: *which quality procedure is failing most*, which points straight at a QP rather than at a symptom.

**Severity-weighted index** — `Σ (Qty × Severity)`. Surfaces the facility with few but critical defects.

### The 24-hour edit lock

Implements what the Application Brief already promises users, with role awareness:

```
If( varIsAdmin || varIsQM
    || (!varHome.Locked && DateDiff(varHome.Entered_at, Now(), TimeUnit.Hours) < 24),
    DisplayMode.Edit, DisplayMode.View)
```

Show a countdown ("editable for 6 more hours") rather than silently disabling controls.

---

## 6. Screens

Six screens.

### 6.1 Dashboard

Filter row scoping everything below it: `[Plant ▾]` (Admin only) · `[Facility ▾]` · `[Window ▾]` · refresh.

**Stat tiles:** First Time Pass Rate · **First Time Through** · TTW · Defects per Home · Homes Tested.

**Q matrix** — kept; it is how plant users recognise the app. Week × test grid for the current month, now with a fourth column for FTT. Every cell carries a **status glyph and the value**, not colour alone. Cells drill into the week.

**Weekly table** — homes tested, per-test pass/fail, FPY, FTT, TTW, defects, defects/floor.

**FPY trend** — single line, one axis, goal as a labelled reference rule.

**Defect Pareto** — over the window, with a toggle between defect level and type-of-test level.

### 6.2 Week detail

Per-test cards for the week, then the homes tested that week with their three results and defect counts, then the week's defect lines. Drill from a home into its full defect list.

### 6.3 Data entry — production number first

The screen you specified, and the biggest UX change from v1.

**Step 1 — context.** `[Facility ▾]` and `[Week ending ▾]` (from `calendar_list`, no future weeks). Sets `varFacility` and `varWeekKey` and loads the week's homes (§4.4).

**Step 2 — add a home.** A text input for the production number and an **Add** button. On Add:

```
// validate
If(IsBlank(txt_home.Text), Notify("Enter a home production number.", Error),
   !IsBlank(LookUp(qk_home, Plant_ID = varPID && Home_number = Trim(txt_home.Text))),
       Notify("Home " & txt_home.Text & " already exists for this plant.", Error),
   // create with all three tests untested
   IfError(
     Patch(qk_home, Defaults(qk_home), {
        Title: varPID & "." & varFacility & "." & Trim(txt_home.Text),
        Plant_ID: varPID, Facility_ID: varFacility, Week_key: varWeekKey,
        Week_start: varWeekStart, Week_end: varWeekEnd,
        Home_number: Trim(txt_home.Text), Floors: Value(txt_floors.Text),
        Fiscal_year: varFY, Fiscal_quarter: varFQ, Week_number: varWeekNo,
        Entered_at: Now(), Entered_by: varUserEmail
     }),
     Notify("Could not save home. " & FirstError.Message, Error)
   );
   Reset(txt_home); Select(btn_load_week)
)
```

> **The duplicate check has no week filter, and that is deliberate.** A home is built once. The same production number appearing in two different weeks is a typo or a double-entry, not a legitimate record — so uniqueness is per plant, for all time. `Home_number` is Text, so `=` delegates; it is indexed, so the check stays fast as the list grows.

**Step 3 — the week's gallery.** Every production number entered for that facility-week, each row showing the number, three test chips (Pass / Fail / — not tested), floors, and defect count. Sorted newest first. A progress line reads *"14 homes entered · 9 fully tested · 5 pending."*

Since production numbers are typed with nothing to validate against, the gallery is the safety net: inspectors see the whole week at a glance and spot a mistyped number immediately.

**Step 4 — test a home.** Selecting a home opens its panel: three cards, one per test, each a **Pass / Fail** toggle.

Setting a test to **Fail** opens its defect rows beneath it:

```
[ Type of test ▾ ] → [ Defect ▾ ] → [ Qty ]   🗑
```

Both dropdowns bind to in-memory collections, so they are instant:

```
Type:    Sort(Filter(col_type,   Test_ID = ThisItem.Test_ID), Sort_order)
Defect:  Sort(Filter(col_defect, Type_ID = ThisItem.Type_ID), Sort_order)
```

Three behaviours specific to this taxonomy:

- **The type picker is short.** Electrical has 4 types, Plumbing 3, Gas 2 — so the first dropdown is a quick, near-glanceable choice rather than a scroll through 20 items. That is a real entry-speed gain over the item-based draft.
- **The QP reference shows beside the type**, e.g. `Continuity · QP.07.0`, so an inspector recording a defect sees which procedure it belongs to without leaving the screen.
- **Defect names stand alone.** All 66 are globally unique (§2.1), so pickers, galleries and charts can label by defect name without a parent prefix.

Because a home can fail the same test under more than one procedure — a Plumbing failure with both a Water Lines and a Drain Lines defect — defect rows are **not** limited to one type per test.

**Step 5 — save.** Validation before write:

| Rule | Behaviour |
|---|---|
| Test = Fail with no defect lines | **Warn** — "Electrical failed but no defect recorded." Allow, flag the home as incomplete |
| Test = Pass with defect lines | **Block** — a contradiction |
| `Floors` < 1 | **Block** |
| Duplicate `Type + Defect` on one home | **Block** — merge into `Qty` instead |
| Home number not matching `Home_number_pattern` | **Warn** |

Then: patch `qk_home` → reconcile `qk_home_defect` (add new, patch changed, remove deleted) → recompute the week's rollups (§6.4). **Every write wrapped in `IfError` with a visible `Notify`** — v1 had no error handling on any write.

### 6.4 Rollup maintenance

After any save, the app rebuilds that facility-week's rollups **from scratch** rather than incrementing them:

```
// col_homes already holds this facility-week (~18 rows)
Set(varHomes, CountRows(col_homes));
Set(varPass,  Sum(col_homes, Coalesce(Elec_result,0) + Coalesce(Plumb_result,0) + Coalesce(Gas_result,0)));
Set(varTests, Sum(col_homes, If(IsBlank(Elec_result),0,1) + If(IsBlank(Plumb_result),0,1) + If(IsBlank(Gas_result),0,1)));
Set(varClean, CountRows(Filter(col_homes, Elec_result=1 && Plumb_result=1 && Gas_result=1)));
// … then Patch qk_week_summary and the three qk_test_summary rows by Title
```

> **Rebuilding beats incrementing.** The row set is ~18 homes, so a full recompute costs nothing and is **self-healing** — a failed save, an edit, or a deletion can never leave the rollup drifted from the facts. Incremental counters would drift silently, which is exactly the class of bug that made v1's stored TTW disagree with its displayed TTW.

### 6.5 Taxonomy Admin — Administrators only

Three panes: **Tests** (read-only, 3) → **Types of test** (CRUD, with `QP_ref`) → **Defects** (CRUD, with Severity).

- `Active` toggles, never deletes — `qk_home_defect` references these IDs forever.
- Deactivating something already referenced warns with the usage count.
- New `Type_ID` / `Defect_ID` allocated as `max + 1` within the parent.
- `QP_ref` is editable, so a procedure renumbering is a field edit rather than a rename of every type.
- Surfaces the §2.2 review items: inconsistent `QP_ref` formatting, types with an unusually small or large defect count, and any defect still sitting at the default severity.

This is the screen that keeps the taxonomy a data concern rather than a release concern — Quality adds a defect type without the app being republished.

### 6.6 Access Admin — Administrators only

Maintains `sys_test_access`, shared with v1.

- List of people, filterable by plant and role.
- **Add** via `Office365Users.SearchUserV2`, then assign **Plant** and **Role**.
- Assigning a Quality Manager is exactly this: pick the person, pick the plant, set Role = `Quality Manager`.
- Edit changes plant, role, or `Active`. Deactivate rather than delete.
- A warning when a plant has no Quality Manager assigned.

> Because v1 reads this same list, a person added here immediately has v1 access too. That is the intent — one access list — but it must be understood before go-live, and it means **`sys_test_access` is jointly owned**: no renaming or retyping existing columns while v1 lives.

---

## 7. UI and design system

Brand chrome is retained from v1 (blue `#0F6CBD` headers, `#590000` accents). Data colours are replaced.

**Categorical — test identity**, fixed slots, assigned by entity, never cycled:

| Slot | Test | Light | Dark |
|---|---|---|---|
| 1 | Electrical | `#2a78d6` | `#3987e5` |
| 2 | Plumbing | `#eb6834` | `#d95926` |
| 3 | Gas | `#1baf7a` | `#199e70` |

Both modes validated all-pairs (worst CVD ΔE 9.2 light / 9.4 dark; normal-vision 24.0 / 20.9). Light-mode aqua sits at 2.82:1 contrast — discharged by the design carrying visible direct labels and a table view on every chart.

**Status — replaces v1's raw red/green:** good `#0ca30c` · warning `#fab219` · serious `#ec835a` · critical `#d03b3b` · no data `#e1e0d9`.

> **v1 encodes pass/fail as pure red against green with no secondary channel.** For a deuteranopic reader — roughly 6% of men, in a manufacturing workforce — the Q matrix is close to unreadable. **v2 rule: a status colour never carries meaning alone.** Every status ships a glyph (`▲` `▼` `–`) and the value.

**Charts:** stat tiles for single numbers; status cell grid for the Q matrix; single-line trend with a goal reference rule (v1 plotted nine series including `ID` and `Period_day` as data); grouped bars for test comparison; Pareto as **bars-as-percentage-share plus cumulative line on one 0–100% axis** — a conventional dual-axis Pareto invents a relationship between two arbitrary scales. Pareto bars are one colour; colouring nominal bars by magnitude double-encodes length as hue.

**Interaction:** one filter row above everything it scopes; hover and keyboard focus show the same tooltip; tooltips enhance but never gate a value; ≥24px hit targets; hold the previous render at reduced opacity on refresh rather than flashing a skeleton; a table-view twin for every chart.

---

## 8. Build sequence

| Phase | Work | Exit criteria |
|---|---|---|
| **0. Provision** | Create 10 lists. **Apply indexes before loading any data.** Import `seed/qk_test.csv`, `qk_test_type.csv`, `qk_defect_type.csv`. Add `Role` + `Active` to `sys_test_access`. Seed `qk_config` | Indexes verified; Quality has reviewed the taxonomy and set severities |
| **1. Shell** | New app: `OnStart`, `btn_load_plant`, role resolution, plant/facility filters, taxonomy load | Admin plant switch reloads in <2s; **delegation warnings: zero** |
| **2. Entry** | Production-number flow, week gallery, three test cards, defect rows, validation, save, rollup rebuild | Round-trip a week of 18 homes; duplicate and contradiction rules fire correctly |
| **3. Dashboard** | Stat tiles, Q matrix, weekly table, FPY trend | FPY reconciles against v1 for an overlapping week |
| **4. Analysis** | Week detail, home drill-in, Pareto, severity index | Pareto matches a hand-computed sample at both defect and type level |
| **5. Admin** | Taxonomy Admin, Access Admin with roles | Quality adds a defect type and assigns a QM unaided |
| **6. Pilot** | One plant runs v1 and v2 in parallel for 4 weeks | Weekly FPY matches; pilot plant prefers v2 |
| **7. Cutover** | Remaining plants; v1 read-only | — |

**No historical migration.** v1 has no home-level data, so v2 starts clean. v1's weekly history stays in v1 (read-only) for trend continuity; if a combined long-run trend is needed later, load both into Power BI rather than back-filling fake homes.

**Do not skip the phase-1 delegation check.** Turn on delegation warnings in Studio and treat each one as a build break.

---

## 9. Open questions

1. **Severity.** All 66 defects seeded at 2 (Major). Who sets the real values, and is 1/2/3 the right scale? Gas leaks are presumably 3.
2. **Taxonomy cleanup before go-live** — the §2.2 items, particularly Electrical → Misc (21 defects) and the Plumbing item overlaps. Cheap now, expensive after history accumulates.
3. **`Floors` per home** — is this known at test time, and does the inspector enter it or does it come from the model?
4. **Is `Model` worth capturing?** It would let you compare defect rates across floor plans, which the Application Brief hints at as an end goal.
5. **Homes that are never tested.** With no production schedule to reconcile against, the app cannot know a home was missed. Is there any list of scheduled homes, even a weekly export?
6. **Cross-plant benchmarking** — Power BI over these lists, or a screen inside the app? If in-app, it pushes hard toward Dataverse.
7. **Does RTS (Ready To Ship) survive?** v1 tracked it separately from system testing. This spec folds defects into the home record; if RTS is a distinct downstream inspection, it needs its own result columns or its own list.

---

## Sources

- [Understand delegation in a canvas app — Microsoft Learn](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/delegation-overview)
- [Power Apps delegable functions and operations for SharePoint — Microsoft Learn](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/connections/connection-sharepoint-online#power-apps-delegable-functions-and-operations-for-sharepoint)
- [SharePoint delegation improvements — Microsoft Power Platform Blog](https://www.microsoft.com/en-us/power-platform/blog/power-apps/sharepoint-delegation-improvements/)
- [List View Threshold for large lists and libraries — Microsoft Support](https://support.microsoft.com/en-us/sharepoint/lists/data-and-lists/list-view-threshold-for-large-lists-and-libraries)
- [Living Large with Large Lists and Large Libraries — Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-365/community/large-lists-large-libraries-in-sharepoint)
