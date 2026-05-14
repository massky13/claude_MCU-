# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Power Apps Canvas App for the **OpEx Training Hub** at Champion Homes. It manages Lean Six Sigma certification exams (currently LSSGB — Green Belt). The app is authored as `.pa.yaml` files using the Canvas Authoring MCP server and synced to Power Apps Studio.

Two app folders exist:
- `lssgb-test/` — the active development version
- `lssgb-test-sync/` — a sync copy kept in step with the active version

## Authoring workflow

This project uses the Canvas Authoring MCP server. Use the `/canvas-app` skill to create or edit screens. The MCP server compiles and syncs changes to Power Apps. Do not hand-edit `.pa.yaml` files without the MCP server unless you are making trivial structural fixes.

## Screen navigation

```
screen_main  →  screen_lssgb_landing  →  screen_LSSGB  →  screen_results
                                                ↑
                              screen_lssgb_roster (admin roster management)
```

`App.OnStart` sets `varUserEmail = User().Email` and starts on `screen_main`.

## SharePoint data sources

All persistence is through SharePoint lists connected via the Power Apps SharePoint connector:

| List | Purpose | Key fields |
|---|---|---|
| `lss_questions` | Question bank | `Title` (ID), `exam`, `question`, `answerA`–`answerE`, `correct_answer` |
| `lss_sessions` | Exam sessions per user | `UserEmail`, `SessionID`, `ElapsedSeconds`, `IsCompleted`, `IsPaused`, `AttemptNumber`, `SessionStatus`, `CorrectAnswers`, `ScorePercent`, `Passed`, `CompletedDate` |
| `lss_responses` | Per-question answers | `Title` (composite key), `SessionID`, `UserEmail`, `QuestionID`, `QuestionNumber`, `SelectedAnswer`, `IsCorrect`, `AnswerTimestamp` |
| `lss_certificates` | Employee certification roster | `UserEmail`, `UserName`, `Number` (cert #), `type` (LSSGB/LSSYB/LSSWB), `role` (admin/user), `TrainingDone`, `TestDone`, `ProjectDone`, `award` |
| `Documents` | Question images library | `exam`, `QuestionID`, `Thumbnail.Large` |

### Composite key in `lss_responses`

`Title` = `varSessionID & "." & User().EntraObjectId & "." & varCurrentQ.Title`

This uniquely identifies a response row. All Patch/LookUp operations on `lss_responses` use this key.

## Connectors

- **Office365Outlook** — sends pass/fail email to candidate, massky@gmail.com, and ralfonzo@skylinehomes.com on exam submit or timeout
- **Office365Users** — employee photo (`UserPhotoV2`), user search (`SearchUserV2`), user info
- **SharePoint** — all four lists above

## Exam flow (`screen_LSSGB`)

1. `screen_lssgb_landing.OnVisible` loads `col_lssgb_questions`, counts completed attempts (`varCompletedAttempts`), and checks for an in-progress session (`varInProgressSession`).
2. Start button: if an in-progress session exists, resumes it (restores `varSessionID`, `varElapsedSeconds`, `varRemainingSeconds`); otherwise creates a new `lss_sessions` record and sets `varSessionID`.
3. `screen_LSSGB.OnVisible`: if no responses exist for this session, shuffles questions with `Rand()`, creates 100 `lss_responses` records in both `colResponses` (local) and SharePoint. Otherwise loads existing responses.
4. Navigates to first unanswered question (`IsBlank(SelectedAnswer)`).
5. **Timer1** counts down from `varRemainingSeconds * 1000`; on end it auto-submits, patches the session, and sends email.
6. **Timer2** ticks every 1 second (`Repeat=true`), increments `varElapsedSeconds`, and saves elapsed time to SharePoint every 300 seconds.
7. Answer selection (A–E) patches both `colResponses` and `lss_responses` immediately.
8. Submit: calculates score, patches `lss_sessions` and `lss_certificates.TestDone`, sends email, navigates to `screen_results`. Maximum 3 attempts enforced by `DisplayMode.Disabled` on the Start button.

## Key global variables

| Variable | Type | Meaning |
|---|---|---|
| `varUserEmail` | Text | `Lower(User().Email)` — always lowercase |
| `varSessionID` | Number | ID of current `lss_sessions` record |
| `varElapsedSeconds` | Number | Seconds elapsed in current session |
| `varRemainingSeconds` | Number | 7200 − elapsed (set on resume/start) |
| `varIsPaused` | Boolean | Controls both timers via `Start` property |
| `varIsCompleted` | Boolean | Locks exam after submit or timeout |
| `varCurrentQ` | Record | Current question from `col_lssgb_questions` |
| `var_question_number` | Number | 1–100, current position |
| `var_question_ID` | Text | `Title` of current question |
| `varSelectedAnswer` | Text | "A"–"E" or Blank() |
| `varCompletedAttempts` | Number | Count of `IsCompleted=true` sessions |
| `var_permission` | Text | "admin" or "user" from `lss_certificates.role` |

## Roster screen (`screen_lssgb_roster`)

- Reads `var_permission` from `lss_certificates` on `OnVisible`.
- Shows gallery of all employees filtered by cert type dropdown.
- Admins can toggle `TrainingDone`, `TestDone`, `ProjectDone` by clicking the check/cancel badge icons.
- "Add Person" panel uses `Office365Users.SearchUserV2` to find employees, assigns a random unique certificate number (0–9999), and writes to `lss_certificates`.

## Design conventions

- Primary brand color: `RGBA(2, 66, 109, 1)` (dark navy)
- Background: `RGBA(245, 245, 245, 1)` (light gray)
- All screens use a left-rail navigation sidebar (`Container3_*`) containing ModernText links
- AutoLayout containers are used throughout; ManualLayout only where pixel-exact placement is needed
- Theme: `CHB` (set in `App.Properties.Theme`)
