# CLAUDE.md — Quality KPI Reporting v2

**This file is build instructions for an agent.** You are building a Power Apps **canvas** app into an existing blank app, against SharePoint lists that already exist in it.

| Companion file | What it holds |
|---|---|
| `QUALITY-KPI-V2-SPEC.md` | Full design rationale — read it when this file says "see spec §X" |
| `V1-AS-BUILT.md` | The v1 production app. **Reference only. Never modify v1.** |
| `quality-kpi-v2-mockup.html` | Working UI prototype. Open it — it is the visual target |
| `seed/*.csv` | Taxonomy import files (already loaded into SharePoint) |

---

## 0. Mission

Champion Home Builders plants test **every home** they build against three quality procedures. This app records each home's pass/fail per test, the specific defects behind any failure, and rolls that into weekly KPIs per plant and facility.

```
Home (production number)
  └─ Test          Electrical · Plumbing · Gas          — fixed at 3
       └─ Type         the quality procedure run         — 9
            └─ Defect     the specific finding           — 66
```

Six screens: **Corporate Home · Dashboard · Data Entry · Week Detail · Taxonomy Admin · Access Admin.** **Every user starts on Corporate Home** (§8.0) and opens a plant's dashboard from its card.

**Ready To Ship (RTS) is out of scope for this build.** v1 tracked it as a separate inspection; it is deliberately excluded here. Do not add RTS columns, screens, or metrics.

---

## 1. Target environment

| | |
|---|---|
| Environment ID | `Default-906f2348-17ef-4fb9-943a-f0a55a301f94` |
| App ID | `ddb5db73-1171-4b0c-9aa3-e1e5797a6435` |
| Studio URL | `https://make.powerapps.com/e/Default-906f2348-17ef-4fb9-943a-f0a55a301f94/canvas?app-id=%2Fproviders%2FMicrosoft.PowerApps%2Fapps%2Fddb5db73-1171-4b0c-9aa3-e1e5797a6435` |
| App template | Responsive, **HeaderMainFooter**, single screen — extend it, don't fight it |
| Data | SharePoint lists, already connected to the app |

---

## 2. Toolchain

Build through Microsoft's official canvas-apps plugin. It owns all canvas mechanics — YAML shape, control properties, validation. **This file owns the project: data model, formulas, screens, rules.** Where they disagree about *how* to author, the plugin wins.

```
/plugin marketplace add microsoft/power-platform-skills
/plugin install canvas-apps@power-platform-skills
```

Requires the **.NET 10 SDK**. The plugin runs a `canvas-authoring` MCP server exposing `compile_canvas`, `sync_canvas`, `list_controls`, `list_data_sources`, `describe_control`, `get_data_source_schema`.

> ⚠️ **The Power Apps Studio browser tab must stay open for the whole session.** This is a live coauthoring session — closing the tab ends it and you lose unsynced work.

> ⚠️ **Do not hand-write `.pa.yaml` and expect it to load.** Outside this plugin's sync path, `.pa.yaml` is a read-only export; manual edits are ignored and lost. Author through the MCP tools, validate with `compile_canvas`, land with `sync_canvas`.

**Verify the schema before you build.** Call `get_data_source_schema` on every list and reconcile against §4. If a column name or type differs from this file, **stop and report it** — do not silently adapt, because §4 names appear inside dozens of formulas below.

---

## 3. Non-negotiables

These are the things that break the app. Violating any of them produces something that looks fine and is wrong.

**1. `Plant_ID` leads every SharePoint predicate.** SharePoint's 5,000-item list-view threshold blocks any query whose *first* filter isn't on an indexed column returning under 5,000 rows. `qk_home` crosses 5,000 across all plants in roughly five weeks of production.

**2. Filter on `Week_key` (Number), never on a date column.** Microsoft's own sources disagree on whether SharePoint delegates DateTime comparison. `Week_key` is `yyyymmdd` as a Number — unambiguously delegable, correctly sortable, cleanly indexable. `Week_end` is for display only.

**3. Never put optional logic inside a predicate.** This silently kills delegation:
```powerfx
// ❌ WRONG
Filter(qk_home_defect, Plant_ID = varPID && (IsBlank(varFacility) || Facility_ID = varFacility))
```
Branch the whole `Filter` instead — see §6.4.

**4. Compute function calls into variables before filtering.** `DateAdd(...)` inline in a predicate breaks delegation. `Set(varCutoffKey, …)` first, then compare against the variable.

**5. Turn delegation warnings on and treat every one as a build break.** A surviving warning is a silently wrong KPI later.

**6. Never encode status with colour alone.** Every status cell, pill and badge ships a **glyph and the numeric value**. This is not decoration — see §11.

**7. Aggregates run over collections, never over lists.** `Sum`, `Average`, `FirstN`, `GroupBy`, `Distinct` do not delegate to SharePoint. Filter delegably into a small collection first, then do the maths there.

**8. Wrap every write in `IfError` with a visible `Notify`.** v1 had no error handling on any write and failures were silent.

**9. Do not modify v1.** Different lists, different app. `sys_test_access` is the one shared list — you may add to it, never rename or retype its existing columns.

---

## 4. Data model

Column names are used verbatim in every formula below. `*` marks an indexed column.

### Taxonomy — read-mostly, loaded once at startup

**`qk_test`** (3 rows) — `Test_ID`* · `Test_name` · `Sort_order` · `Active`
`1 = Electrical`, `2 = Plumbing`, `3 = Gas`. Fixed. Do not build UI to add a fourth.

**`qk_test_type`** (9 rows) — `Type_ID`* · `Test_ID`* · `Type_name` · `QP_ref` · `Sort_order` · `Active`

| Type_ID | Test | Type_name | QP_ref | Defects |
|---|---|---|---|---|
| 11 | Electrical | Hi Pot (Dielectric) | QP.06A.0 | 6 |
| 12 | Electrical | Continuity | QP.07.0 | 7 |
| 13 | Electrical | Operational | QP.08.0 | 8 |
| 14 | Electrical | Polarity | QP.09.0 | 5 |
| 21 | Plumbing | Drain Lines | QP.03.0 | 10 |
| 22 | Plumbing | Water Lines | QP.01.0 | 12 |
| 23 | Plumbing | Fixtures | QP.02.0 | 5 |
| 31 | Gas | High Pressure | QP04.0 | 9 |
| 32 | Gas | Low Pressure | QP05.0 | 4 |

**`qk_defect_type`** (66 rows) — `Defect_ID`* · `Type_ID`* · `Test_ID` · `Defect_name` · `Severity` · `Sort_order` · `Active`

`Severity`: 1 minor · 2 major · 3 critical. **All 66 currently seeded at 2** — the severity-weighted index is meaningless until Quality sets real values. Surface that on the admin screen; don't invent values.

All 66 `Defect_name` values are globally unique, so charts and pickers may label by defect name alone.

### Facts

**`qk_home`** — one row per home. The centre of the model.

`Title` (`PID.FAC.HomeNumber`) · `Plant_ID`* · `Facility_ID`* · `Week_key`* · `Home_number`* (Text) · `Week_start` · `Week_end` · `Model` · `Floors` · `Elec_result` · `Plumb_result` · `Gas_result` · `Fiscal_year` · `Fiscal_quarter` · `Week_number` · `Entered_at` · `Entered_by` · `Locked`

Results are **Number**: `1` pass, `0` fail, **blank = not tested**. Three states, so never a Boolean. Storing 1/0 lets `Sum()` give the pass count directly.

Three result columns rather than a child table because the tests are fixed at 3 — see spec §2.3 for the trade.

**`qk_home_defect`** — one row per home × defect type.

`Title` (`PID.FAC.HomeNumber.DefectID`) · `Plant_ID`* · `Facility_ID`* · `Week_key`* · `Home_number`* · `Test_ID` · `Type_ID` · `Defect_ID` · `Qty` · `Severity` · `Notes`

`Qty` aggregates repeats — three receptacles with reverse polarity is one row with `Qty: 3`. `Severity` is **denormalised at write time** so historical severity survives taxonomy edits.

### Rollups — derived, never entered

**`qk_week_summary`** — one row per plant × facility × week.
`Title` (`PID.FAC.yyyymmdd`) · `Plant_ID`* · `Facility_ID`* · `Week_key`* · `Week_start` · `Week_end` · `Homes_tested` · `Homes_clean` · `FTT_pct` · `Total_tests` · `Total_pass` · `Total_fail` · `Week_FPY` · `Week_TTW` · `Total_floors` · `Total_defects` · `Defects_per_floor` · `Defects_per_home` · `Fiscal_year` · `Fiscal_quarter` · `Week_number`

**`qk_test_summary`** — one row per week × facility × test (3 per facility-week).
`Title` · `Plant_ID`* · `Facility_ID`* · `Week_key`* · `Test_ID` · `Homes_tested` · `Pass` · `Fail` · `Test_FPY`

### Supporting

**`sys_test_access`** — **shared with v1.** `Inspector_name` · `Inspector_email` · `Plant_ID` · `Plant_name` · `Facility_ID` (Text) · `Role` · `Active`. `Plant_ID = 0` means Corporate.

| Role | Scope | Can do |
|---|---|---|
| `Administrator` | all plants | everything, plus taxonomy and access maintenance |
| `Quality Manager` | one plant | enter and edit their plant's data, no time limit; set their plant's defect costs (§8.4) |
| `Inspector` | one plant | enter and edit their plant's data, no time limit |
| `Reader` | one plant | view only |

There is no time-based edit restriction — see §7, "Editability" (the 24-hour lock originally speced here was built, then deliberately removed: it limited legitimate corrections without a compensating benefit).

**`qk_plant_defect_cost`** — per-plant defect cost overrides. `Title` (`PID.DefectID`) · `Plant_ID`* · `Defect_ID`* · `Defect_cost` · `Updated_at` · `Updated_by`. Holds **only** the costs a plant has changed; a defect with no row uses the corporate default `qk_defect_type.Defect_cost`. `EffectiveDefectCost(defectId)` in `App.Formulas` resolves plant override → corporate default over `col_plant_cost` (the current plant's rows, ≤ 66, loaded by `RunFullPlantLoad`). The cost is **stamped** onto `qk_home_defect.Defect_cost` when a defect is recorded, so a cost change applies going forward only — past weeks never reprice, and a re-saved floor keeps its original stamp.

**`qk_config`** — the setting name lives in **`Title`**, not a `Setting_name` column; the value is `Setting_value`.

| Title | Value | Used for |
|---|---|---|
| `Goal_FTPR_green` | `80` | First Time Pass Rate / TTW — green at or above (`BandStatusFTPR`) |
| `Goal_FTPR_yellow` | `65` | First Time Pass Rate / TTW — yellow at or above, red below |
| `Goal_green` | `90` | Per-test pass rates — green at or above (`BandStatusOf`) |
| `Goal_yellow` | `80` | Per-test pass rates — yellow at or above, red below |
| `Goal_FPY` | `90` | Legacy — still loaded, drives nothing |
| `Goal_FTT` | `75` | Legacy — still loaded, drives nothing |
| `Window_months` | `6` | Load window |

**No threshold literal appears anywhere in the app.** Read from here. (`Home_number_pattern` was removed from this list — see §9.)

**`calendar_list`** — `W_start` · `W_end` · `Week` · `Month` · `Year` · `Fiscal_year` · `Fiscal_quarter`. The only source of truth for week boundaries and fiscal mapping. Never derive Sunday/Saturday arithmetically. **There is no `W_key` column** — derive it the same way everywhere: `Value(Text(W_end, "yyyymmdd"))`.

**`Plant_List`** — one row per **plant × facility**, not one row per plant. Columns actually used by the app: `Plant_ID` · `Plant_Names` · `Facility` (Text — the facility number, e.g. `"1"`). There is no numeric facility-ID column on this list; cast with `Value(Facility)` wherever the fact tables' numeric `Facility_ID` is needed. (The live list also carries several plant-admin/contact columns — City, Champion, GM_email, etc. — not modeled here because nothing in this app reads them.)

---

## 5. `App.OnStart`

```powerfx
Set(varUserEmail, Lower(Trim(User().Email)));
ClearCollect(col_access, Filter(sys_test_access, Active = true));
Set(varAccessRow, LookUp(col_access, Lower(Trim(Inspector_email)) = varUserEmail));

Set(varRole,      Coalesce(varAccessRow.Role, "Reader"));
Set(varIsAdmin,   varRole = "Administrator");
Set(varIsQM,      varRole = "Quality Manager");
Set(varCanEdit,   varRole <> "Reader");

Set(varPID,
    If(!IsBlank(varAccessRow),
       varAccessRow.Plant_ID,
       Value(Left(Office365Users.UserProfileV2(User().Email).officeLocation, 3))
    )
);

// Taxonomy: 3 + 9 + 66 rows. Load once; never reload on plant change.
ClearCollect(col_test,     Filter(qk_test,        Active = true));
ClearCollect(col_type,     Filter(qk_test_type,   Active = true));
ClearCollect(col_defect,   Filter(qk_defect_type, Active = true));
ClearCollect(col_calendar, calendar_list);
ClearCollect(col_config,   qk_config);
ClearCollect(col_plants,   Plant_List);

Set(varGoalFPY,      Value(LookUp(col_config, Title = "Goal_FPY",      Setting_value)));
Set(varGoalFTT,      Value(LookUp(col_config, Title = "Goal_FTT",      Setting_value)));
Set(varGoalGreen,      Value(LookUp(col_config, Title = "Goal_green",       Setting_value)));
Set(varGoalYellow,     Value(LookUp(col_config, Title = "Goal_yellow",      Setting_value)));
Set(varGoalFTPRGreen,  Value(LookUp(col_config, Title = "Goal_FTPR_green",  Setting_value)));
Set(varGoalFTPRYellow, Value(LookUp(col_config, Title = "Goal_FTPR_yellow", Setting_value)));
Set(varWindowMonths, Value(LookUp(col_config, Title = "Window_months", Setting_value)));

// The user's own plant - IsForeignPlant() compares the plant being viewed against it.
Set(varOwnPID, varPID);
// Everyone starts on scr_CorporateHome; opening a plant's card loads it. Admins start with none.
If(varIsAdmin, Set(varPID, Blank()))
```

Identity is two-tier: an explicit `sys_test_access` grant wins; otherwise the first 3 characters of the user's AD `officeLocation` are parsed as the plant number.

**Role denies by default.** A user with no active `sys_test_access` row resolves to `Reader`, not `Inspector` — they are an "Other User" in the role table (§8.0a) and get view-only access. The `officeLocation` fallback still gives them a `varPID` so the dashboard has a plant to load; it grants no write rights.

---

## 6. Loading

### 6.1 The rule

**Delegation applies to the query, never to maths over a collection.** Filter delegably into something small, then aggregate locally without limit.

### 6.2 `btn_load_plant.OnSelect` — the only place plant data loads

A hidden button on the dashboard. Canvas apps have no user-defined procedure, so this is the reuse mechanism. Call it with `Select(btn_load_plant)` from `OnStart`, the plant dropdown, and the refresh icon. **Never duplicate this logic anywhere.**

```powerfx
Set(varCutoffDate, DateAdd(Today(), -varWindowMonths, TimeUnit.Months));
Set(varCutoffKey,  Value(Text(varCutoffDate, "yyyymmdd")));

ClearCollect(col_week,
    Filter(qk_week_summary, Plant_ID = varPID && Week_key >= varCutoffKey));

ClearCollect(col_test_sum,
    Filter(qk_test_summary, Plant_ID = varPID && Week_key >= varCutoffKey));

Clear(col_homes); Clear(col_home_defects);
Set(varLoadedAt, Now())
```

~312 rows for a plant over 6 months. `qk_home` and `qk_home_defect` are **never bulk-loaded.**

### 6.3 On demand

Week's homes (entry screen, ~18 rows) — three Number equalities on indexed columns:
```powerfx
ClearCollect(col_homes,
    Filter(qk_home, Plant_ID = varPID && Week_key = varWeekKey && Facility_ID = varFacility))
```

One home's defects:
```powerfx
ClearCollect(col_home_defects,
    Filter(qk_home_defect, Plant_ID = varPID && Home_number = varHomeNumber))
```
`Home_number` is Text — `=` delegates, range operators do not. Never write `Home_number > …`.

Pareto window (~1,000 rows) — branch, never guard inside the predicate:
```powerfx
If(IsBlank(varFacility),
   ClearCollect(col_defect_window,
       Filter(qk_home_defect, Plant_ID = varPID && Week_key >= varCutoffKey)),
   ClearCollect(col_defect_window,
       Filter(qk_home_defect, Plant_ID = varPID && Week_key >= varCutoffKey && Facility_ID = varFacility))
)
```

---

## 7. Metrics

All rates stored **0–100**, one decimal.

**The app has one quality metric: First Time Pass Rate, on a floor basis.**

```powerfx
// First Time Pass Rate - one week: floors that passed ALL THREE tests / floors tested
First_Time_Pass_Rate = Floors_clean / Floors_tested * 100      // qk_week_summary.FTT_pct

// TTW - the same ratio over the trailing 12 weeks, WEIGHTED (never an average of weekly %)
TTW = Sum(Floors_clean) / Sum(Floors_tested) * 100              // qk_plant_summary.FTT_pct

// Per test, per facility-week - labelled "Electrical pass rate" etc., never "FPY"
Test_FPY = homes passing that test / homes tested for that test * 100

Defects_per_home  = Sum(Qty) / Homes_tested
Defects_per_floor = Sum(Qty) / Sum(Floors)
```

The headline figure is the **last completed** calendar week (`PastWeekKey` in `App.Formulas`) — the week in progress is always mid-entry. Cross-plant and cross-facility figures are always weighted sums of `Floors_clean` / `Floors_tested`, never a mean of percentages.

**Retired from the UI:** the composite test-weighted FPY — `Week_FPY` (`Sum(passes) / Sum(tests)`) and `TTW_FPY`. `scr_Entry` still writes both columns so nothing downstream breaks, but no screen displays or grades them. For a single test a floor and a test are the same thing, so the per-test pass rates (`Test_FPY`) stay.

**Labels:** "First Time Pass Rate" for the weekly figure, "TTW" / "Trailing 12 weeks" for the 12-week figure. No user-facing "FTT", "First Time Through" or "FPY".

> **TTW had one definition in v1 for display and a different one for storage, and they disagreed.** There is one definition here — `qk_plant_summary.FTT_pct`, written by `btn_rebuild_rollups`. Use it in both places.

### Status

Two band functions in `App.Formulas`, same shape, both returning `"good" | "warning" | "critical" | "nodata"`. A missing config row grades `"nodata"`.

```powerfx
// First Time Pass Rate and TTW - every tile, badge, card and the trend goal rule
BandStatusFTPR(value) =
    If(IsBlank(value), "nodata",
       IsBlank(varGoalFTPRGreen) || IsBlank(varGoalFTPRYellow), "nodata",
       value >= varGoalFTPRGreen,  "good",      // qk_config Goal_FTPR_green
       value >= varGoalFTPRYellow, "warning",   // qk_config Goal_FTPR_yellow
       "critical")

// Per-test pass rates (Test_FPY) only - same shape, Goal_green / Goal_yellow
BandStatusOf(value)
```

The floor-basis rate needs its own, lower bands: a floor must clear three tests to count, so a plant at 90% on every test lands near 73% here.

### Editability

No time-based edit lock. The Application Brief originally called for a 24-hour edit window (v1 promised it and never built it); v2 built it, then removed it deliberately — it limited legitimate corrections (e.g. a QM fixing a typo from last week) without a compensating benefit.

```powerfx
Set(varHomeEditable, varCanEdit)   // varCanEdit = varRole <> "Reader" (App.OnStart)
```

Readers get `DisplayMode.View` everywhere on `scr_Entry`; every other role can edit any home, any time. `qk_home.Locked` still exists in the schema but is no longer read by the app — it is not wired to anything.

---

## 8. Screens

Follow the HeaderMainFooter responsive template — **§10 defines the responsive behaviour and is binding**. Naming: `scr_` screens · `cnt_` containers · `gal_` galleries · `lbl_` labels · `txt_` inputs · `drp_` dropdowns · `btn_` buttons · `ico_` icons · `col_` collections · `var_`/`var` globals.

The mockup (`quality-kpi-v2-mockup.html`) is the visual target. Open it before building each screen.

### 8.0 `scr_CorporateHome` — landing screen for every user

A region × plant comparison, and the landing screen for **everyone**: `App.StartScreen = scr_CorporateHome`, unconditionally. Users see how their plant compares with the others in their region. The user's own plant (`varOwnPID`) has a thick Champion Blue outline, a light fill and a bold name, and their region's card has a heavier outline.

**Plant scope.** Any user can open **any** plant's dashboard from its card. `varOwnPID` is the user's own plant; `varPID` is the plant being viewed. `IsForeignPlant()` (in `App.Formulas`) is true for a plant user viewing a plant that isn't theirs. **Administrators and Corporate users (`varOwnPID = 0`) are unrestricted** on every plant. On another plant's dashboard:
- the nav shows only Corporate Home and Dashboard;
- the header reads "(view only)";
- tiles and Q-matrix cells don't drill into Week Detail;
- `HasPlantCostScope()` is false, so the user can never touch that plant's costs.

On their own plant a user gets the full nav their role allows.

Deliberately scoped to **`qk_week_summary` only** — never `qk_home`/`qk_home_defect`. A trailing-12-week window across every plant stays a few hundred rows regardless of plant count, unlike home/defect-level drill-down, which is what makes full cross-plant benchmarking a later, Dataverse-scale phase (§13). Facilities are consolidated up to plants (`Plant_List` is one row per plant × facility); regions come from `Plant_List.Plant_Region`, generated as columns rather than hardcoded, same principle as the Q-matrix's test columns (§8.1).

Every figure is First Time Pass Rate on the floor basis (§7), graded with `BandStatusFTPR`. Plant badges show each plant's latest reported week (`qk_plant_summary.FTT_latest_pct`); region badges show the last completed week, `Sum(Floors_clean) / Sum(Floors_tested)` over the region's plants from `qk_week_summary`. `Plant_TTW` (`qk_plant_summary.FTT_pct`) and `Region_TTW` (`Sum(Floors_clean) / Sum(Floors_tested)` over the plants' 12-week components) are computed on the same floor basis. All of them are **weighted**, never an average of each plant's own percentage. Selecting a plant calls the app-level `SelectCorporatePlant()` UDF, which sets `varPID`/`varFacility` and navigates to `scr_Dashboard`. `scr_Dashboard.OnVisible` does the load (`RunFullPlantLoad()`), so it isn't duplicated. Every screen's nav has a Corporate Home tab to get back here.

### 8.1 `scr_Dashboard`

**Header** — Q logo · "Quality KPI Reporting" · `lbl_plant` (`PID – plant name`) · `ico_theme` · `ico_admin` (visible `varIsAdmin`).

**Filter row**, scoping everything below it — never per-card filters:
`drp_plant` (visible `varIsAdmin`, `OnChange` sets `varPID` then `Select(btn_load_plant)`) · `drp_facility` · `drp_window` · `lbl_loaded`.

**`cnt_tiles`** — stat tiles: **First Time Pass Rate (last wk)** (`tile_ftt`) · **TTW** (`tile_ttw`, `qk_plant_summary.FTT_pct`) · Defects per Home · Cost · Floors Tested. Each shows value, delta or context, and a status accent. The two rate tiles grade with `BandStatusFTPR` (§7).

**`cnt_qmatrix`** — the app's signature visual, kept from v1. `gal_qmatrix` over `Filter(col_calendar, Month = Month(Today()), Year = Year(Today()))`, one row per week of the current month. Columns are **generated from `col_test`**, not hardcoded: one per-test pass rate per column, graded with `BandStatusOf`. There is no First Time Pass Rate column; that figure lives on `tile_ftt`, the trend and `scr_History`. Each cell carries **glyph + value + colour** and drills into the week.

**`cnt_trend`** — one axis. The overall line is **First Time Pass Rate** (`FTT_pct`, labelled so in the legend), plus the three per-test pass-rate lines in their slot colours, and the goal as a labelled horizontal reference rule at `Goal_FTPR_green`. Nothing else. v1 plotted nine series including `ID` and `Period_day` as data — do not repeat that.

**`gal_weeks`** (on `scr_History`) — week ending · floors · per-test pass/fail · First Time Pass Rate · defects · defects per floor. Row select opens `scr_WeekDetail`.

**`cnt_pareto`** — defect Pareto over the window, with a **By defect / By type** toggle. Bars are each item's **percentage share** so bars and the cumulative line share one 0–100% axis. **Never a dual-axis Pareto.** All bars one colour.

### 8.2 `scr_Entry` — production number first

The most important screen. Build it before the dashboard.

**Header** — `drp_facility` sits on the right side of the header band (not the context bar), matching `scr_Dashboard`'s facility scoping. `OnChange` sets `varFacility` (cast from `Plant_List`'s `Facility` text column via `Value(...)` — see §4) and reloads.

**Context bar (`cnt_filters_en`)** — `gal_week_picker`, a horizontal gallery of week buttons, replaces a `drp_week` dropdown. It shows the **10 consecutive weeks running from 1 week ahead of today back to 9 weeks before that** — i.e. it deliberately includes the upcoming week so an inspector can start logging homes before the week fully closes out. Tapping a button sets `varWeekStart`/`varWeekEnd`/`varWeekKey` (derived from `W_end`, since `calendar_list` has no `W_key` — see §4) and loads §6.3. This supersedes §9's old "week ending in the future → Block" rule; see §9.

**Add row** — `txt_home_number`, `txt_floors`, `btn_add_home`:

```powerfx
If( IsBlank(txt_home_number.Text),
    Notify("Enter a home production number.", NotificationType.Error),

    !IsBlank(LookUp(qk_home, Plant_ID = varPID && Home_number = Trim(txt_home_number.Text))),
    Notify("Home " & txt_home_number.Text & " already exists for this plant.", NotificationType.Error),

    IfError(
        Patch(qk_home, Defaults(qk_home), {
            Title:       varPID & "." & varFacility & "." & Trim(txt_home_number.Text),
            Plant_ID:    varPID,
            Facility_ID: varFacility,
            Week_key:    varWeekKey,
            Week_start:  varWeekStart,
            Week_end:    varWeekEnd,
            Home_number: Trim(txt_home_number.Text),
            Floors:      Value(txt_floors.Text),
            Fiscal_year: varFY, Fiscal_quarter: varFQ, Week_number: varWeekNo,
            Entered_at:  Now(),
            Entered_by:  varUserEmail
        }),
        Notify("Could not save home. " & FirstError.Message, NotificationType.Error)
    );
    Reset(txt_home_number);
    Select(btn_load_week)
)
```

> **The duplicate check has no week filter, deliberately.** A home is built once — the same production number in two weeks is a typo, not a record. Uniqueness is per plant, for all time. `Home_number` is indexed Text, so `=` stays delegable and fast.

**`gal_week_homes`** — every home for the facility-week: number, three test chips (Pass / Fail / not tested), floors, defect count. Above it a progress line: *"14 homes entered · 9 fully tested · 5 pending."* Production numbers are typed with nothing to validate against, so this gallery is the safety net — an inspector spots a mistyped number at a glance.

**`cnt_home_detail`** — three test cards, one per `col_test`, each a **Pass / Fail** toggle. Setting a test to **Fail** opens defect rows beneath it:

```
[ Type of test ▾ ] → [ Defect ▾ ] → [ Qty ]  🗑
```

Both dropdowns bind to in-memory collections:
```powerfx
Type:   Sort(Filter(col_type,   Test_ID = ThisItem.Test_ID), Sort_order)
Defect: Sort(Filter(col_defect, Type_ID = ThisItem.Type_ID), Sort_order)
```

- Show the **QP reference beside the type** — `Continuity · QP.07.0` — so an inspector sees which procedure they're recording against.
- A home can fail one test under more than one procedure. **Do not limit defect rows to one type per test.**
- A new defect row must default to a **defect not already recorded on that home**, so adding a row never creates a duplicate.

**Save** — validate (§9), then patch `qk_home`, reconcile `qk_home_defect` (add / patch / remove), then rebuild the rollups (§8.6).

### 8.3 `scr_WeekDetail`

Per-test cards for the week (subtitle "Pass rate"), a First Time Pass Rate card, then the homes tested with their three results and defect counts, and each home's defect lines. Reached from a Q-matrix cell or a `gal_weeks` row.

### 8.4 `scr_TaxonomyAdmin` — Administrators, plus a cost-only mode for Quality Managers

Three panes: **Tests** (read-only, 3) → **Types of test** (CRUD, with editable `QP_ref`) → **Defects** (CRUD, with `Severity`).

- `Active` toggles, never deletes — `qk_home_defect` references these IDs forever.
- Deactivating something already referenced warns with its usage count.
- New IDs allocated `max + 1` within the parent.
- Surface these review flags: inconsistent `QP_ref` formatting (Gas uses `QP04.0` / `QP05.0` without the dot while the other seven use `QP.0X.0`), and how many defects still sit at default severity.

This screen is what keeps the taxonomy a data concern rather than a release concern.

**Plant defect costs.** Whenever a real plant is in scope (`HasPlantCostScope()`: `varPID > 0`), each defect row shows the corporate cost beside the plant's cost ("Plant = corp" when there's no override), and the edit row adds a plant-cost input. Saving a blank plant cost removes the override. Writes go to `qk_plant_defect_cost` from `btn_defect_save`. This is inline, not a UDF, because the compiler flags any data-writing UDF as non-delegable at its call site.

- **Quality Managers** (`IsCostOnlyMode()` = `varIsQM && !varIsAdmin`) reach this screen from a **"Defect Costs"** nav tab, shown only to a QM with a plant. They see active types and defects only, and every add / rename / QP_ref / activate control is hidden. The only action is **Edit cost** for their own plant; they never write `qk_defect_type`.
- **Administrators** keep the full taxonomy. With a plant picked they can also set that plant's costs from the same row.
- UI gating is not security. QMs need Contribute on `qk_plant_defect_cost`, and should have read-only access to `qk_test_type` / `qk_defect_type` in SharePoint.

### 8.5 `scr_AccessAdmin` — `varIsAdmin` only

Maintains `sys_test_access`. List filterable by plant and role; add via `Office365Users.SearchUserV2`; assign **Plant** and **Role**; deactivate rather than delete. Warn when a plant has no active Quality Manager.

> This list is **shared with v1** — a person added here gets v1 access immediately. That is intended, but it means `sys_test_access` is jointly owned. Add columns freely; never rename or retype existing ones.

### 8.6 Rollup maintenance

After **any** save, rebuild that facility-week's rollups **from scratch** — never increment:

```powerfx
Set(varHomes, CountRows(col_homes));
Set(varPass,  Sum(col_homes, Coalesce(Elec_result,0) + Coalesce(Plumb_result,0) + Coalesce(Gas_result,0)));
Set(varTests, Sum(col_homes, If(IsBlank(Elec_result),0,1) + If(IsBlank(Plumb_result),0,1) + If(IsBlank(Gas_result),0,1)));
Set(varClean, CountRows(Filter(col_homes, Elec_result = 1 && Plumb_result = 1 && Gas_result = 1)));
// … then Patch qk_week_summary and the three qk_test_summary rows, keyed by Title
```

The row set is ~18 homes, so a full recompute costs nothing and is **self-healing** — a failed save, an edit or a delete can never leave the rollup drifted. Incremental counters drift silently, which is exactly the bug class that made v1's stored TTW disagree with its displayed TTW.

---

## 9. Validation rules

| Rule | Behaviour |
|---|---|
| Home number blank | **Block** |
| Home number already exists for this plant (any week) | **Block** |
| `Floors` < 1 | **Block** |
| Test = Fail with no defect lines | **Warn**, allow, flag the home incomplete |
| Test = Pass with defect lines | **Block** — a contradiction |
| Duplicate `Type + Defect` on one home | **Block** — merge into `Qty` |

Warnings must never block a plant from reporting. Blocks are reserved for data that would be wrong.

**Home numbers have no format requirement.** v1's plants use their own production-number coding, not a shared pattern — do not validate `Home_number` against any regex, and do not add a `Home_number_pattern`-style config setting back.

**There is no "week ending in the future" block.** `gal_week_picker` (§8.2) intentionally makes the upcoming week selectable, so a home can legitimately be entered against a week whose `Week_end` hasn't arrived yet.

---

## 10. Responsive layout

**Target hardware: PC screens and tablets on the shop floor.** Both are first-class. Phones are not a design target but must not break.

### App-level settings — verify these first

`Settings → Display`:

| Setting | Value |
|---|---|
| **Scale to fit** | **OFF** |
| **Lock aspect ratio** | **OFF** |
| **Lock orientation** | **OFF** — tablets get used both ways |

> ⚠️ **Scale to fit is ON by default in Power Apps.** The responsive template normally ships with it off, but **confirm it in Studio before building anything**. With it on, the app renders at its design size and merely zooms — letterboxed on a desktop, and shrinking this app's dense tables below readable size on a laptop. Every layout formula below is inert unless this is off.

On the App object, `Properties → Advanced`:

```
App.MinScreenWidth  = 720
App.MinScreenHeight = 640
App.SizeBreakpoints = [600, 900, 1200]      // the default; do not change
```

The floor of 720 sits just below the narrowest tablet in portrait. Above it the layout adapts; below it the user scrolls rather than the layout degrading into something unusable.

### Breakpoints and what each one is for

| `ScreenSize` | Width | Device | Design target |
|---|---|---|---|
| `Small` | < 600 | phone | **Not a target.** Must remain usable, not pretty |
| `Medium` | 600–900 | **tablet portrait** | Full support |
| `Large` | 900–1200 | **tablet landscape** | Full support |
| `ExtraLarge` | ≥ 1200 | **desktop / laptop** | Full support, the primary layout |

Branch with `Parent.Size` / `Screen.Size`, never with raw pixel widths.

### Per-screen behaviour

**`scr_Dashboard`**

| | ExtraLarge | Large | Medium | Small |
|---|---|---|---|---|
| Stat tiles | 5 across | 3 + 2 | 2 across | 1 across |
| Q matrix / trend | side by side | stacked | stacked | stacked |
| Pareto / by-test | side by side | stacked | stacked | stacked |
| Weekly table | all 10 columns | drop `Per floor` | also drop the 3 per-test pass/fail columns | as Medium |
| Charts | chart view | chart view | chart view | **table view by default** |

Dropped table columns are not lost — the per-test figures live on `scr_WeekDetail`, one tap away. Hide with `Visible`, e.g. `Parent.Size >= ScreenSize.Large`.

**`scr_Entry` — the one that matters most on a tablet**

| | ExtraLarge / Large | Medium / Small |
|---|---|---|
| Layout | Homes list **beside** home detail | **List, then detail** |

At Medium and below the two panes stop coexisting. The gallery fills the screen; selecting a home replaces it with the test panel plus a back chevron; saving or backing out returns to the list.

```powerfx
// cnt_home_list.Visible
Parent.Size >= ScreenSize.Large || IsBlank(varSelectedHome)

// cnt_home_detail.Visible
Parent.Size >= ScreenSize.Large || !IsBlank(varSelectedHome)

// ico_back.Visible   (returns to the list on narrow screens)
Parent.Size < ScreenSize.Large && !IsBlank(varSelectedHome)
```

> v1 half-built this pattern — a context variable named `itemSelected` was read in several visibility expressions but only ever set to `false`, so the collapse never fired (`V1-AS-BUILT.md` §10 item 12). Build it properly and test it at tablet width.

**`scr_WeekDetail`** — test cards wrap from 4 across to 2 to 1. The homes table drops its defect-detail column at Medium; tapping a row still opens the full list.

**`scr_TaxonomyAdmin`** — three panes (Tests → Types → Defects) at ExtraLarge and Large. At Medium and below, **one pane at a time with drill-down navigation** and a breadcrumb back, same pattern as the entry screen.

**`scr_AccessAdmin`** — drop the email column at Medium; it stays visible on the edit form.

### Touch

Tablets are touch-first. On `Medium` and below:

- **Minimum 44px height on every interactive control.** v1 used 32px text inputs — too small for a gloved hand on a shop floor.
- The Pass / Fail toggles, `+ Add`, `+ Defect`, and gallery rows are the controls that matter most. Size them generously; they are tapped hundreds of times a week.
- Keep at least 8px between adjacent tap targets so a near-miss does nothing rather than the wrong thing.
- **The on-screen keyboard shrinks the viewport.** Keep the home-number input near the top of `scr_Entry`, and make sure the defect `Qty` fields stay reachable — put the defect rows in a scrollable container rather than relying on the screen scrolling.

### Verify before calling a screen done

Test each screen at four widths: **1920, 1280, 1024, 768**. The last two are tablet landscape and portrait and are where this app breaks if it is going to.

Check at each: nothing clipped, no horizontal scrollbar, all text readable without zooming, every tap target comfortable, and the entry screen's list-then-detail swap working at 768.

---

## 11. Design tokens

### Chrome — Champion Home Builders brand

| Role | Hex |
|---|---|
| Primary / headers | `#02426D` Champion Blue |
| Accent | `#FFB500` Gold |
| Light accent / fills | `#9CD4EA` Light Aqua |
| Alert | `#E15242` Red |

### Test series — validated, assign by entity, never cycle

| Slot | Test | Hex |
|---|---|---|
| 1 | Electrical | `#1B6FA8` |
| 2 | Plumbing | `#eb6834` |
| 3 | Gas | `#1baf7a` |

Slot 1 is a mid-tone Champion Blue so the data reads on-brand. This set passes colourblind separation on all pairs (worst CVD ΔE 9.2, normal-vision ΔE 22.5). **Do not substitute the raw brand palette for series colours** — Champion Blue and Light Aqua are both cool and sit ΔE 13 apart, below the legibility floor; they were tested and rejected.

### Status

| Role | Hex | Meaning |
|---|---|---|
| good | `#0ca30c` | at or above target |
| warning | `#FFB500` | within 5 points (brand gold) |
| serious | `#ec835a` | below target |
| critical | `#E15242` | below target 3+ consecutive weeks (brand red) |
| no data | `#e1e0d9` | grey |

> **Why glyphs are mandatory.** Run any red/green status pair through a colourblind check and it fails hard — `#E15242` against `#0ca30c` measures ΔE 2.3 under deuteranopia, effectively identical. Roughly 6% of men have some form of it, in a manufacturing workforce. v1 encoded pass/fail as pure red against green with no second channel, which makes its Q matrix close to unreadable for those users.
>
> **Every status ships `▲` / `▼` / `–` plus the numeric value.** Colour is the fast channel, never the only one. This applies to Q-matrix cells, table pills, badges and chips.

Typography: the app's existing theme font. Proportional figures on tile values; tabular figures in table columns and axis ticks.

---

## 12. Build order

Build and verify one phase at a time. Do not start a phase before the previous one's check passes.

| # | Phase | Check before moving on |
|---|---|---|
| 1 | Verify list schemas against §4 via `get_data_source_schema` | Every column matches, or discrepancies reported |
| 2 | **Set Scale to fit / Lock aspect ratio / Lock orientation OFF**; set `MinScreenWidth` 720, `MinScreenHeight` 640 | Confirmed in `Settings → Display` before any layout work |
| 3 | `App.OnStart`, `btn_load_plant`, role resolution, filter row | Admin plant switch reloads under 2s · **zero delegation warnings** |
| 4 | `scr_Entry` — add home, gallery, test cards, defect rows, save, rollup rebuild | Round-trip a week of ~18 homes; every §9 rule fires correctly |
| 5 | `scr_Dashboard` — tiles, Q matrix, trend, week gallery | First Time Pass Rate reconciles against a hand-computed week |
| 6 | `scr_WeekDetail`, Pareto | Pareto matches a hand-computed sample at both levels |
| 7 | `scr_TaxonomyAdmin`, `scr_AccessAdmin` | An admin adds a defect type and assigns a QM unaided |
| 8 | Full pass | Delegation warnings zero · every write has `IfError` · every status has a glyph · **every screen checked at 1920 / 1280 / 1024 / 768** |

Entry comes before the dashboard deliberately: the dashboard reads rollups that only exist once entry writes them.

---

## 13. Out of scope

- **RTS (Ready To Ship)** — excluded from this build by decision.
- **Historical migration** — v1 has no home-level data. v2 starts clean; v1 keeps its own history read-only.
- **Cross-plant benchmarking beyond `scr_CorporateHome`'s week-summary rollup** — the Application Brief's end goal, but a later phase. `scr_CorporateHome` (§8.0) covers the narrow case of comparing plants/regions on `qk_week_summary` alone; anything needing home- or defect-level detail *across* plants is what pushes the loading design toward Dataverse, not this.
- **Retest tracking** — first-time result only. Rework is not recorded.

## 14. Known gaps — raise, don't invent

1. **All 66 defects sit at severity 2.** The weighted index is meaningless until Quality sets real values.
2. **`QP_ref` formatting is inconsistent** — Gas omits the dot after `QP`.
3. **Gas → Low Pressure may be missing a Water heater entry.** High Pressure covers Furnace / Range / Water heater / Dryer; Low Pressure covers valve / Furnace / Range / crossover. A duplicate row was dropped during seeding and may have been meant as Water heater.
4. **`Floors` source is unconfirmed** — whether the inspector types it or it derives from the model.
5. **No production schedule to reconcile against**, so the app cannot know a home was never tested.

If you hit any of these mid-build, surface it. Do not fill the gap with invented data.
