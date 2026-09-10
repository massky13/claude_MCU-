# V1 As-Built — Quality KPI Reporting (Power Apps Canvas App)

> Reference only. This is the **v1 production app**, kept for behavioural comparison during the v2 build.
> The active build instructions are in `CLAUDE.md`.

> **Purpose of this file.** It is the baseline description of the application *as it exists and works today*. Read it before proposing or making any change. When you change the app, update this file in the same pass so it never drifts from reality.

---

## ⚠️ Read this first — there are two applications

| | **v1 — production** | **v2 — in design** |
|---|---|---|
| Status | Live. Plants report into it weekly. | Design only. Nothing built. |
| Described by | **This file** (§1–§10) | **`QUALITY-KPI-V2-SPEC.md`** |
| Data | `system_testing_data`, `sys_test_access` | new `qk_*` lists, built alongside |
| Grain | 3 trades × pass/fail per week | Category → Sub-category → Item → Defect type |

**The decision is to run them in parallel.** v1 stays untouched and in production while v2 is built on new lists; cutover happens per-plant after a pilot. So:

- **Do not "improve" v1** while implementing v2. Bugs listed in §10 are fixed *in the v2 design*, not by patching production underneath reporting plants.
- **Do not reuse v1 list names.** v2 lists are `qk_`-prefixed precisely so the two never collide.
- Changes to v1 are limited to genuine production incidents, and each one must be reflected here.

§11 records how each v1 defect is resolved in v2.

---

## 1. What the app is

A Power Apps **canvas app** used by Champion Home Builders plants to report and view **weekly quality KPIs** for manufactured-home production. Two metric families are tracked:

- **System Testing** — pass/fail floor counts for three trades: **Electrical (E)**, **Plumbing (P)**, **Gas (G)**.
- **RTS (Ready To Ship)** — number of floors inspected and number of non-conformities (NCs) found that week.

Reporting cadence is **weekly, Sunday → Saturday**. Every metric is compared to a target; above target renders **green**, below target **red**, and missing data **gray**. That three-state color rule is the app's core visual language and appears on every screen.

Scope of a user's world is their **plant**, and within a plant their **facility**. Users can only see, add, or edit their own plant's data. Corporate users (`PID = 0`) see all plants and additionally get the access-management screen.

Long-term intent stated in the Application Brief: the data plants enter feeds a **larger filtered report**, so plants can track their own quality trend *and* benchmark against comparable plants (similar output and floor models).

---

## 2. Source files in this repo

Exported Power Apps YAML source (`.pa.yaml`, Power Fx in canvas-app source format):

| File | Contents |
|---|---|
| `App.pa.yaml` | `OnStart`, `StartScreen`, theme |
| `Welcome screen.pa.yaml` | Main dashboard — "Q" matrix, weekly table, trend chart |
| `Records screen.pa.yaml` | Data entry — list / edit / new / delete |
| `Access List screen.pa.yaml` | Corporate-only access management |
| `Application Brief.docx` | End-user brief + two UI screenshots (the design intent of record) |
| `QUALITY-KPI-V2-SPEC.md` | **v2 architecture** — schemas, delegation strategy, screens, build sequence |
| `quality-kpi-v2-mockup.html` | Interactive v2 UI mockup (open in a browser) |

`Application Brief.docx` is the **user-facing contract**. Where the brief and the code disagree, treat it as a defect to raise, not as license to change either silently. Known disagreements are listed in §10.

---

## 3. Data sources

All are SharePoint lists unless noted. There is **no Dataverse**; the app writes directly to SharePoint.

### `system_testing_data` — the fact table (one row per plant × facility × week)

| Field | Type / meaning |
|---|---|
| `ID` | SharePoint item ID (primary key used by all Patch/Remove lookups) |
| `Title` | Composite natural key: `PID & "." & Facility & "." & yyyymmdd` of week end |
| `Plant_ID` | Plant number |
| `Facility_ID` | Facility number within the plant (1, 2, 3) |
| `Week_start` / `Week_end` | Sunday / Saturday of the reporting week |
| `MMDD_field` | `"mm/dd"` text of week end. **Note:** the line chart binds to `MM_x002f_DD_field` (SharePoint's URL-encoded internal name for the same column). |
| `Period` | `"yyyy-mm"` |
| `Period_year`, `Period_month`, `Period_day` | Calendar parts |
| `Fiscal_year`, `Fiscal_quarter` | Fiscal calendar, sourced from `calendar_list` |
| `Week_number` | Week number from `calendar_list` |
| `Electrical_pass` / `Electrical_fail` | Floor counts |
| `Plumbing_pass` / `Plumbing_fail` | Floor counts |
| `Gas_pass` / `Gas_fail` | Floor counts |
| `Week_FPY` | Stored **first-time pass rate as 0–100**, rounded up to 1 decimal |
| `Week_TTW` | Stored **trailing-twelve-week** average of `Week_FPY` |
| `Goal_FPY` | Written as `0.9` on new records (see §10 — unit mismatch with `Week_FPY`) |
| `RTS_floors` | Floors inspected this week |
| `RTS_NCS` | Non-conformities this week |
| `RTS_AVG` | `RTS_NCS / RTS_floors` — NCs per floor |

### `sys_test_access` — who may see what

`ID`, `Title` (`email & "." & Plant_ID`), `Inspector_name`, `Inspector_email`, `Plant_ID`, `Plant_name`.
`Plant_ID = 0` means **Corporate admin**.

### `calendar_list` — the fiscal/weekly calendar

`W_start`, `W_end`, `Week`, `Month`, `Year`, `Fiscal_year`, `Fiscal_quarter`.
This list is the **single source of truth for week boundaries and fiscal mapping** — the app never derives Sunday/Saturday arithmetically for saved records, it looks it up here.

### `Plant_List` — plant reference

`Plant_ID`, `Plant_Names`.

### Connectors

- **Office365Users** — `UserProfileV2` (reads `officeLocation` to infer plant), `SearchUserV2` (people picker on the Access screen).
- **PowerAppsforMakers** — `GetAppVersions` for the published-date stamp on the Welcome screen. **The app ID is hardcoded**: `8246a9fb-d4ea-4d9b-8032-9bcaec9648ea`.

---

## 4. Collections and global state

| Name | Kind | Set where | Holds |
|---|---|---|---|
| `PID` | global var | `App.OnStart`, `drop_facility_main_1.OnChange` | Current plant ID; `0` = Corporate |
| `var_facility` | global var | `App.OnStart` = `1` | Initial facility; largely superseded by the dropdown |
| `col_sys_access` | collection | `App.OnStart`, Access screen | Copy of `sys_test_access` |
| `col_sys_test` | collection | `App.OnStart` and every facility/plant change | Working copy of `system_testing_data`, filtered to plant (+ facility) |
| `col_calendar` | collection | `App.OnStart` | Copy of `calendar_list` |
| `col_plant_options` | collection | Access screen `OnVisible` | `"Corporate"` (ID 0) + all plants, sorted |
| `accSearchResults` | collection | Access screen search button | Office365 people-search results |
| `var_lastweekgas` | global var | `App.OnStart` | Gas pass ratio (0–1) for the **last calendar week of the current month** — drives the Q-tail color only |
| `varTTW` | global var | Records screen saves | Freshly computed trailing-12-week average |
| `selectedRecord` | global var | Records screen | Record being edited |
| `selectedDate` | global var | New-record save | Parsed week-end date, used for validation |
| `deleteMode`, `editMode`, `newMode`, `itemSelected`, `deleteCancelled` | screen context | Records screen | Which panel is visible |
| `accSelected`, `accNewMode`, `accDeleteMode`, `accPickedUser` | screen context | Access screen | Which panel is visible |

**Pattern to preserve:** the app maintains `col_sys_test` as a client-side mirror and **dual-writes** every change to both the collection and the SharePoint list. See §10 — this is deliberate today but is the app's biggest structural liability.

---

## 5. Startup and identity resolution (`App.OnStart`)

```
If ( User().Email in sys_test_access.Inspector_email,
     Set(PID, LookUp(sys_test_access, Inspector_email = User().Email, Value(Plant_ID))),
     Set(PID, Value(Left(Office365Users.UserProfileV2(User().Email).officeLocation, 3))) );
Set(var_facility, 1);
ClearCollect(col_sys_access, sys_test_access);
ClearCollect(col_sys_test, Filter(system_testing_data, Plant_ID = PID && Facility_ID = var_facility));
ClearCollect(col_calendar, calendar_list);
IfError(Set(var_lastweekgas, <gas pass ratio for last week of month>), "");
```

Two-tier identity model, in priority order:

1. **Explicit grant** — email found in `sys_test_access` → use that row's `Plant_ID`.
2. **Fallback by AD** — first 3 characters of the user's Entra/AD `officeLocation` parsed as a number.

The brief's line *"the system will ask for some access permissions during the first launch"* refers to the standard connector-consent prompt (SharePoint + Office365Users), not to an in-app flow.

---

## 6. Screens

### 6.1 Welcome screen — the dashboard (`StartScreen`)

Header (blue `RGBA(15,108,189,1)`, 75px): Q logo · title **"Quality KPI Reporting"** · `PID – <Plant name>` (or `"Corporate"`) · facility dropdown `drop_facility_main` · user photo · gear icon (`btn_access_gear`, visible only when `PID = 0`).

Body is a two-column auto-layout that wraps on small screens.

**Left column — the "Q" matrix.** This is the app's signature visual. `gal_qmatrix` is a vertical gallery over `Filter(col_calendar, Month = Month(Today()), Year = Year(Today()))` — one row per week of the current calendar month. Each row is split into four equal quarters of `TemplateWidth`:

| Quarter | Control | Shows |
|---|---|---|
| 1 | `txt_q_electrical` (via header icon `Letter E2`) | Electrical pass % |
| 2 | `txt_q_plumbing` | Plumbing pass % |
| 3 | `txt_q_gas` | Gas pass % |
| 4 | `txt_RTS` | `RoundUp(RTS_AVG, 0)` |

`txt_left` / `txt_right` overlay the week's `W_start` and `W_end`.

The **letter "Q" shape** is drawn by three geometry controls positioned relative to the gallery, *not* by an image:

- `Rectangle3` — the white "hole" in the middle of the Q.
- `Rectangle5` — the base of the Q's tail.
- `Triangle1` — the diagonal tail, colored by `var_lastweekgas`.

All three are anchored with expressions referencing `gal_qmatrix.X/Y/TemplateWidth/TemplateHeight/TemplatePadding` and `CountRows(gal_qmatrix.AllItems)`. **Changing the gallery's template size, padding, or row count will visibly break the Q.** Verify the shape after any layout change.

**Right column — the weekly detail table.** `Gallery1` over the plant/facility-filtered `col_sys_test`, sorted `Week_end` descending. Columns: Week ending · E pass/fail · P pass/fail · G pass/fail · First Time Pass Rate · TTW · RTS (`NCs / Floors -> NC/Floor`). Each cell independently red/green by the 90% rule. The red pencil `Icon2` navigates to the Records screen; it is **hidden when the facility dropdown is on "All"**, since you cannot enter data for an aggregate.

Below the table, `LineChart1` plots `Week_FPY` (and eight other series) over `MM_x002f_DD_field`, `YAxisMax = 100`.

Bottom-left: `drop_facility_main_1` (plant selector, visible only to users present in `col_sys_access`) and a small label stamping the app's publish date via `PowerAppsforMakers`.

A hidden button `ButtonCanvas2` ("Q control") contains an older `WeeklyRanges` generation routine. It is **dead code kept for reference** — `gal_qmatrix` reads `col_calendar`, not `WeeklyRanges`.

### 6.2 Records screen — data entry

Header: "Weekly Quality Tests" · Home icon (`Back()`) · offline-sync status icon bound to `Connection.Sync`.

Left sidebar: **New** (`+`) · facility label · `gal_edit_items`, a gallery over `col_sys_test` filtered to the selected facility, sorted by `Week_end` descending, each row showing `Week_start to Week_end` and `FTR: <Week_FPY> - NCs/Floor: <RTS_AVG>`.

Right pane shows exactly one of four panels, driven by context variables:

| Panel | Visible when |
|---|---|
| `EditContainer` | `!deleteMode && !newMode && editMode && !IsBlank(selectedRecord)` |
| `NewContainer` | `newMode` |
| `DeleteContainer` | `deleteMode` |
| (nothing) | no selection |

**Edit panel** — header shows week end, fiscal year, quarter, plus a trash icon. Two cards:
- *System Testing* — six numeric text inputs (`txt_electricalpass/fail`, `txt_plumbingpass/fail`, `txt_gaspass/fail`), each defaulting to the matching `selectedRecord` field, with a live **First Time Rate** readout.
- *Ready To Ship* — `txt_numberoffloors`, `txt_numberofnonconformities`, live **NCs per Floor**.

Save (`ButtonCanvas4`) patches `col_sys_test` **and** `system_testing_data` with identical field sets, recomputes `varTTW`, patches `Week_TTW` into both, then resets every input and clears `selectedRecord`.

**New panel** — adds a fiscal-year dropdown (`Dropdown2`), quarter dropdown (`Dropdown2_1`, cascading), and week-end dropdown (`drop_week_end_date`, cascading off both, sourced from `calendar_list`). Same two cards with `_1`-suffixed duplicate controls. Save (`ButtonCanvas4_1`) validates before writing:

1. Week end in the future → `Notify(..., NotificationType.Error)`, abort.
2. Record already exists for this `Week_end` + `Plant_ID` + `Facility_ID` → error, abort.
3. Otherwise `Patch(col_sys_test, Defaults(...))` and `Patch(system_testing_data, Defaults(...))` with the full field set including the composite `Title`, recompute `varTTW`, patch it into both, reset inputs, `Back()`.

**Delete panel** — plain confirm/cancel; on confirm, `Remove` from both the collection and the list.

### 6.3 Access List screen — Corporate only

Reached from the Welcome screen gear icon, which is visible only when `PID = 0`. **Access to this screen is gated by control visibility, not by a security boundary** (see §10).

`OnVisible` refreshes `col_sys_access` and builds `col_plant_options` as `"Corporate"` (ID 0) followed by the alphabetized plant list.

- **List** — `acc_gallery` over `col_sys_access`, sorted by plant then inspector name; each row shows name, email, and plant (`Plant_ID = 0` renders as "Corporate Admin").
- **Edit panel** — name, email, plant dropdown; Save patches `sys_test_access` by `ID` and re-pulls `col_sys_access`.
- **Add panel** — search box → `Office365Users.SearchUserV2({searchTerm: ..., top: 15})` → results gallery (display name, mail, office location) → pick a user → choose plant → **Add to Access List**. The Add button is disabled until a user is picked. Writes `Title` as `email & "." & Plant_ID`.
- **Delete panel** — confirm/cancel, `Remove` from `sys_test_access`.

---

## 7. Metric definitions (authoritative)

Let `Ep/Ef`, `Pp/Pf`, `Gp/Gf` be pass/fail floor counts.

**Per-trade pass rate** (used for the Q-matrix cells and the E/P/G columns):
```
rate = pass / (pass + fail)
```
Rendered as `RoundUp(rate, 2) * 100 & "%"` in the Q matrix.

**Week FPY / First Time Pass Rate / FTR** — all three names refer to the same number:
```
Week_FPY = RoundUp( (Ep + Pp + Gp) / (Ep + Pp + Gp + Ef + Pf + Gf) * 100 , 1 )
```
Stored on the 0–100 scale.

**TTW (trailing twelve weeks)** — mean `Week_FPY` over the 12 most recent weeks up to and including the row's week:
```
TTW = Average( FirstN( Sort(<weeks>, Week_end, Descending), 12 ), Week_FPY )
```
See §10: the Welcome-screen display and the Save-button write compute this over **different row sets**.

**RTS_AVG (NCs per floor)**:
```
RTS_AVG = RTS_NCS / RTS_floors
```
Displayed `RoundUp(..., 1)` on the Welcome table and Records list, `RoundUp(..., 0)` in the Q matrix.

**Target / color rule.** The threshold is **90%** everywhere, and it is **hardcoded as a literal** in every conditional fill on both screens.

| State | Condition | Color |
|---|---|---|
| Above target | `rate >= 0.90` (or `>= 90` on the 0–100 scale) | green — `RGBA(97,207,108,1)` on Welcome tiles, `RGBA(38,187,26,0.5)` in tables |
| Below target | `rate < 0.90` | red — `RGBA(255,0,0,0.5)` / `RGBA(255,0,0,0.3)` |
| No data | expression errors or blank (wrapped in `IfError`) | gray — `RGBA(0,0,0,0.3)` or transparent |

The "no data → gray" behavior depends on `IfError` catching **divide-by-zero** when pass + fail is 0. Do not "fix" a division by pre-guarding it without preserving the gray outcome.

---

## 8. Design system

| Token | Value | Use |
|---|---|---|
| Header blue | `RGBA(15,108,189,1)` | Screen headers, Access accents |
| Theme primary | `PowerAppsTheme.Colors.Primary` | Records header, separators, selection bars |
| Dark red | `RGBA(89,0,0,1)` | Icons, selection bars, Add button on Access |
| Action blue | `RGBA(39,113,194,1)` | Cancel / secondary buttons |
| Danger red | `RGBA(189,49,51,1)` | Confirm-delete buttons |
| Screen fill | `RGBA(234,234,234,1)` / container `RGBA(245,245,245,1)` | Backgrounds |
| Card fill | `RGBA(255,255,255,1)` with `DropShadow.Regular` | Panels |
| Fonts | `Font.'Lato Black'` (titles), `Font.'Open Sans'` (body) | — |

Layout is **auto-layout `GroupContainer`s** at the outer levels (vertical screen → horizontal body → vertical panels) with **`ManualLayout` containers for the form cards and the Q matrix**, where absolute `X`/`Y` still rules.

Responsive behavior on the Records screen collapses the sidebar when `'Records screen'.Size = ScreenSize.Small` and any of `newMode / editMode / itemSelected` is true.

---

## 9. Conventions to follow when editing

- **Naming.** `col_*` collections, `var_*` / `varX` globals, `txt_*` text inputs and text displays, `drop_*` / `ddl_*` / `Dropdown*` dropdowns, `btn_*` buttons, `gal_*` / `acc_*` galleries, `cont_*` / `*Container` containers. New controls should follow this, not the `TextCanvas27` auto-names.
- **Week boundaries always come from `calendar_list`.** Never compute Sunday/Saturday inline for a saved record.
- **Every list write is a dual write** to `col_sys_test` (or `col_sys_access`) *and* the SharePoint list, in that order. If you touch one, touch the other, or refactor both together (see §11).
- **Wrap every division in `IfError`** so empty weeks stay gray instead of showing `#Error`.
- **Keep the 0–100 scale** for `Week_FPY` and `Week_TTW` in storage; convert only at the display edge.
- **`Title` is the composite natural key.** Any new write path must construct it identically: `PID & "." & Facility & "." & Text(weekEnd, "yyyymmdd")`.
- **Do not rename SharePoint columns.** `MMDD_field` / `MM_x002f_DD_field` already demonstrates how internal names leak into formulas.
- Test at both desktop width and `ScreenSize.Small`, and re-check the Q shape after any Welcome-screen layout change.

---

## 10. Known gaps, defects, and drift

Recorded so they are not mistaken for intentional design. **None of these should be changed without confirming with Rafa first.**

**Brief says / app does not:**

1. **24-hour edit lock.** The brief states *"You will be able to edit the data within 24 hours of entering it, after that it will be locked to prevent changes."* **No such lock exists anywhere in the source.** Any record remains editable and deletable indefinitely. This is the largest brief-vs-code gap.
2. **Title mismatch.** Brief and screenshot say **"Quality Test Reporting"**; the running header reads **"Quality KPI Reporting"**.
3. **Larger filtered cross-plant report** — described as the destination for this data; not built.
4. The Access List screen is not mentioned in the brief at all.

**Correctness / consistency:**

5. **TTW is computed two different ways.** The Welcome-screen cell (`txt_total_fpy_1`) averages the last 12 weeks **filtered to the row's `Facility_ID`** and computed from raw pass/fail. Both Save buttons compute `varTTW` as `Average(FirstN(Sort(col_sys_test, Week_end, Desc), 12), Week_FPY)` — **no facility filter, and not anchored to the edited week**. When a plant has multiple facilities loaded, the stored `Week_TTW` and the displayed TTW will disagree.
6. **`Goal_FPY` unit mismatch.** New records store `Goal_FPY: 0.9` while `Week_FPY` is stored 0–100. `LineChart1` plots `Goal_FPY` as `Series6` against a 0–100 axis, so the goal line renders effectively at zero.
7. **90% threshold is hardcoded** in ~10 places rather than read from `Goal_FPY` or a config list. Changing the target today means editing every conditional fill.
8. **`RTS_AVG` is computed by string division** — `Value(txt_numberofnonconformities.Text / txt_numberoffloors.Text)` — relying on implicit coercion, instead of `Value(a.Text) / Value(b.Text)`.
9. **Rounding is inconsistent** for NCs/floor: `RoundUp(...,1)` in the Welcome table and edit form, `RoundUp(...,0)` in the Q matrix and the new-record form.
10. **`var_lastweekgas` drives the Q tail color from Gas only**, for the *last calendar week of the month* — not from the combined FPY and not from the most recent week with data. It is computed once in `OnStart` and never refreshed when the facility or plant changes.
11. **New-record duplicate check ignores the collection.** It queries `system_testing_data` directly, which is correct, but the facility dropdown value is read as text and coerced — worth verifying when facility values are ever non-numeric.
12. **`itemSelected` is read in several visibility expressions but only ever set to `false`** (by the two back-chevron icons). It is never set to `true`, so the small-screen collapse logic is partly inert.

**Structural:**

13. **Dual-write to collection + list** doubles every write, can leave the mirror and the list out of sync if the second `Patch` fails, and has no error handling on any write.
14. **Delegation.** `Filter(system_testing_data, ...)` in `OnStart`, `LookUp(col_sys_test, ...)` inside gallery templates (many per row), `Distinct`, and `SortByColumns` over collections will silently cap at the delegation limit (default 500 / 2000) as history grows. The Q matrix performs roughly **8 `LookUp` calls per week-row** on every render.
15. **Duplicated form controls.** The edit and new forms are near-identical control trees distinguished only by a `_1` suffix, with the entire ~40-line patch expression duplicated. Any field added must be added in four places (two forms × collection + list).
16. **Hardcoded values:** the app GUID `8246a9fb-d4ea-4d9b-8032-9bcaec9648ea`; the facility dropdown `["All", 1, 2, 3]`; a `912`-pixel design width baked into expressions like `70/912*Parent.Width`; absolute X positions (`686`, `510`, `580`, `350`, …) that must move together if a column is inserted.
17. **Corporate access is UI-gated only.** Hiding the gear icon when `PID <> 0` does not prevent a determined user from reaching the Access screen or writing to `sys_test_access`; SharePoint list permissions are the only real control.
18. **Dead code:** `ButtonCanvas2` / `WeeklyRanges` on the Welcome screen; `LineChart1` binds nine series when only `Week_FPY` (and intentionally `Goal_FPY`) are meaningful — `ID`, `Period_day`, `Period_month`, and `Fiscal_year` are plotted as data.
19. **`Reset(txt_electricalfail)` appears twice** and `Reset(txt_plumbingpass)` twice in the edit save, while `txt_electricalpass`'s pair is fine — harmless, but a sign the block was hand-duplicated.
20. **No error handling on writes.** No `IfError` / `Errors()` check around any `Patch` or `Remove`; a failed SharePoint write is silent.

---

## 11. How v2 resolves each v1 defect

Every item in §10 is either fixed by the v2 design or explicitly deferred. Full detail in `QUALITY-KPI-V2-SPEC.md`.

| §10 item | Resolution in v2 | Spec § |
|---|---|---|
| 1. Missing 24-hour edit lock | `Entered_at` + `Locked` columns gate `DisplayMode`; Corporate override; countdown shown to the user | §5 |
| 2. Title mismatch | Single title agreed at build; brief and app reconciled | — |
| 3. No cross-plant report | Deferred — open question 6; likely Power BI, and it is the main argument for Dataverse | §9 |
| 4. Access screen undocumented | Documented; gains a `Role` column | §6.4 |
| 5. TTW computed two ways | **One definition**, facility-filtered and anchored to the row week, used by display and storage alike | §5 |
| 6. `Goal_FPY` unit mismatch | All rates stored 0–100; goal is a reference rule, not a plotted series | §5, §7 |
| 7. 90% hardcoded ~10 places | Single `varGoalFPY` read once from a new `qk_config` list | §5 |
| 8. String division in `RTS_AVG` | Explicit `Value()` on both operands | §5 |
| 9. Inconsistent rounding | NC per floor is 1 decimal everywhere | §5 |
| 10. `var_lastweekgas` quirks | Removed; the Q matrix reads live per-category status | §6.1 |
| 11. Duplicate-check coercion | Natural key `PID.FAC.yyyymmdd` enforced on write | §2.2 |
| 12. `itemSelected` inert | Responsive state rebuilt around the drawer pattern | §6.2 |
| 13. Dual-write drift | **Retired.** Write to SharePoint only, then re-hydrate via one load routine | §6.3 |
| 14. Delegation ceilings | The core of the redesign — `Week_key` Number filtering, indexed columns, ~312 rows at startup, defect log never bulk-loaded | §1, §4 |
| 15. Duplicated edit/new forms | Repeating rows generated from the taxonomy collections; a field is added once | §6.3 |
| 16. Hardcoded values | Facility and category lists come from data; layout rebuilt responsive | §6 |
| 17. UI-only Corporate gating | SharePoint list permissions on `qk_access` + a `Role` column | §6.4 |
| 18. Dead code / 9-series chart | Single-series trend with a goal reference rule | §7 |
| 19. Duplicated `Reset` calls | Gone with the rebuilt save path | §6.3 |
| 20. No write error handling | Every write wrapped in `IfError` with a user-visible `Notify` | §6.3 |

**Also new in v2, beyond fixing v1:** the four-level defect taxonomy in admin-editable config lists, a severity-weighted defect index, a defect Pareto, week-level drill-in, and a Taxonomy Admin screen.

> **One accessibility defect worth calling out separately.** v1 encodes pass/fail as pure red against green with no secondary channel — for a deuteranopic reader the Q matrix is close to unreadable. v2 pairs every status with a glyph and the numeric value. Do not carry v1's color-only pattern into any new screen.

---

## 12. Working agreements

- **v1 is production. Do not modify it while building v2** — see the banner at the top of this file.
- Ask before changing anything in §10 — several items look like bugs but may be intentional workarounds.
- When building v2, `QUALITY-KPI-V2-SPEC.md` is authoritative; this file is the record of what v1 does and why.
- Turn on delegation warnings in Power Apps Studio and treat every one as a build break. A surviving warning is a silently wrong number later.
- Preserve the three-state color language (green / red / gray) in every new visual.
- Preserve the Q-matrix metaphor; it is how plant users recognize the app.
- When adding a stored metric, update: both save paths (new + edit), both collection and list patches, the SharePoint column, the field table in §3, and the metric definition in §7.
- Update this file whenever behavior changes.
