# Backfill Engine for Zoho CRM (Deluge)

A backfill engine that applies a fixed rule to the records that already
exist in a Zoho CRM module, in resumable batches, with a dry run, an
approval step, and a ledger of every batch.

This is a prototype built and verified in the author's own Zoho CRM org
(Ultimate trial, JP data center) using synthetic data only. It is not
client work. See "Verification" and "Limitations" below for exactly
what was and was not proven.

## Problem

Workflow rules in Zoho CRM fire on record events such as create and
edit. When a rule is added or changed, the records that already exist
are not re-evaluated, and there is no built-in "run this rule over
existing records" action. The official alternatives each stop short of
that:

| Option | Official limit | Why it does not cover a backfill |
|---|---|---|
| Mass Update API, scheduler type (custom view / territory) | Up to 50,000 records; automation rules fire only when the record count is less than 1,000 | Above 1,000 records no workflow fires, and every record gets the same value |
| Mass Update API, non-scheduler type (record ids) | Up to 500 records per call; automation fires | Every record gets the same constant value; no per-record evaluation |
| Update Records API | Up to 100 records per call | A building block, not a job: no paging, checkpoint, or resume |
| Bulk Write API | Up to 25,000 records per CSV file | Values must be computed outside CRM first; no rule evaluation, dry run, or per-record ledger |
| COQL (Query API) | Up to 2,000 records per call | Read only |

Sources (Zoho CRM API v8 and Functions documentation):

- Mass Update: https://www.zoho.com/crm/developer/docs/api/v8/mass-update-records.html
- Update Records: https://www.zoho.com/crm/developer/docs/api/v8/update-records.html
- Bulk Write limitations: https://www.zoho.com/crm/developer/docs/api/v8/bulk-write/limitations.html
- COQL limitations: https://www.zoho.com/crm/developer/docs/api/v8/COQL-Limitations.html
- Function limits (Schedule functions time out at 15 minutes): https://www.zoho.com/crm/developer/docs/functions/limits-quotas.html

The workflow-rule behaviour is described in the Zoho CRM help pages on
workflow rules.

The case this engine targets is narrower than "update many records":
compute a different value for each record from its current state, be
able to stop and resume without double-applying or skipping, and leave
a ledger that shows where every record landed. A constant value for
every record is a job for Mass Update, not for this engine.

## Design

**Flow.** Dry run -> fingerprinted approval -> checkpointed execution ->
Retry Run for failures only.

1. **Dry run.** The Runner pages through the target module, evaluates
   every record, and writes nothing to it. Each page goes to
   `Backfill_Logs` (evaluated ids and proposed changes). At the end the
   Run gets `DryRun_Evaluated`, `DryRun_Changed`, `DryRun_At`, and a
   `Config_Fingerprint`, and moves to `DryRunDone`.
2. **Approval.** `bf_approveRun` moves a `DryRunDone` Run to `Queued`
   only if there is no lock, the code's rule version equals
   `Rule_Version`, the fingerprint recomputes to the stored value, and
   the dry run is at most 7 days old.
3. **Execution.** The Runner evaluates every record again (it does not
   replay the dry run), and writes only the records whose values
   change. The fingerprint and rule version are checked again at the
   start; a mismatch writes nothing to the target module and sets
   `Paused_Failure`.
4. **Retry Run.** `bf_makeRetryRun` collects the record ids of the
   `Failure` log rows of a finished Run and creates a new Run that
   inherits the frozen configuration and processes only those ids.

**Id cursor and frozen boundary.** Paging never uses OFFSET. Each page
is `id > cursor and id <= boundary order by id limit 200`. The boundary
(`Boundary_Max_ID`) is the largest matching id, taken once at dry run
start and reused for execution. The scope of a Run is therefore defined
as a set of ids, not as "records created before the dry run": Zoho does
not assign ids in time order, so a record created after the dry run can
still have an id below the boundary and be included (this happened
twice during testing, see Verification), and a record with a larger id
is excluded.

**Intent ledger.** For each execution batch of up to 100 records the
Runner (1) creates a `Backfill_Logs` row with `State = Intent`, holding
the evaluated ids and the proposed `old -> new` values, (2) re-reads
the batch by COQL and skips records whose `Modified_Time` changed since
the page read (`STALE`) or that no longer exist (`MISSING`), (3) sends
one PUT with `"trigger": []`, (4) checks every element of the response
`data[]` and writes a `Failure` row for each non-SUCCESS code, and
(5) updates the same log row to `State = Outcome` with the per-id
result. The cursor moves only after step 5. A row left in `Intent`
marks a batch whose result was never recorded.

**Per-phase cursors and counters.** `DryRun_Cursor` and `Exec_Cursor`
are separate, and the dry run totals (`DryRun_Evaluated`,
`DryRun_Changed`) are kept apart from the execution totals, so the dry
run record is not overwritten by execution.

**Frozen evaluation date.** `Effective_Date` (the "today" used for day
counts) is fixed at dry run time and part of the fingerprint
(`Rule_ID|Rule_Version|Boundary_Max_ID|Effective_Date`). Execution and
Retry Runs evaluate against the same date even on a later day.

**Budget and stop conditions.**

- One Runner invocation processes at most 2 COQL pages of 200 records
  (400 records), or up to 10 chunks of 100 ids in a Retry Run. If the
  range is not finished, the Run is saved as `Paused_Budget` and the
  next invocation resumes from the cursor.
- A missing or malformed COQL / PUT response stops the invocation
  without advancing the cursor for that batch; the Run becomes
  `Paused_Manual` with the reason in `Error_Summary`.
- 200 failures in one Run stop it.
- The Run status is re-read before every page after the first and
  before the final save. `Cancelled` or a terminal status means nothing
  more is saved; `Paused_Manual` (an operator stop) keeps the operator's
  status and saves progress.
- A non-empty `Lock_Token` at start makes the Runner skip the Run and
  append a `LOCK DETECTED` line to `Error_Summary`.

**Idempotence.** A record whose computed values equal its current
values is not written and is counted as `Skipped_NoChange`. Re-running
over records that are already correct produces no PUT at all.

**Writes go to dedicated `BF_` fields only.** No rule writes to an
existing business field.

## Rules

Rules are fixed in code. There is no free-form formula or WHERE clause
input. All boundaries follow one convention: over the threshold is in
scope, exactly on it is not.

| Rule | Module | Inputs | Writes | Logic |
|---|---|---|---|---|
| L1 | Leads | `BF_Registered_Date`, else `Created_Time` | `BF_Days_Since_Created`, `BF_Age_Bucket` | Calendar days from the start date to `Effective_Date` (negative = 0). Buckets `0-30` / `31-90` / `91-365` / `Over365` |
| L2 | Leads | `BF_Registered_Date`, `Lead_Status` | `BF_Stale_Flag`, `BF_Stale_Reason` | Stale when more than 90 days have passed and `Lead_Status` is not in the terminal list. Exactly 90 days is not stale |
| D1 | Deals | `Closing_Date`, `Stage` | `BF_Overdue_Flag`, `BF_Overdue_Days` | Overdue when `Closing_Date` is before `Effective_Date` and `Stage` is not Closed. The same day is not overdue. An empty `Closing_Date` is not overdue |

- L2 and D1 re-flag in both directions: a record that is no longer
  stale / overdue is cleared (flag false, reason empty / days 0), and
  that counts as a change.
- A record that cannot be evaluated (for example no registered date for
  L2, or an unparseable date) is marked `EXCLUDED` in the ledger and
  not written.
- **Why L2 does not use `Last_Activity_Time`.** It was the first
  candidate. Measured in the author's org: a PUT that wrote only `BF_`
  fields, with triggers suppressed, moved `Last_Activity_Time` to the
  write time (it matched the new `Modified_Time`). A rule that reads
  `Last_Activity_Time` would therefore flip its own result after its
  first write. L2 uses a business date field instead. `Modified_Time`
  is not used as a rule input for the same reason.
- The terminal list for L2 is `失注` / `無効` / `精査済み（対象外）`
  and the Closed list for D1 is `Closed Won` / `Closed Lost` /
  `Closed Lost to Competitor`. These are the picklist values of the
  author's org (its Lead Status values are Japanese). Replace them with
  your org's values before use.
- L1 is evaluated inside the Runner. L2 and D1 are separate pure
  functions (`bf_evaluateL2`, `bf_evaluateD1`) with string arguments
  and no API calls.

## Data model

### Backfill_Runs (custom module)

One record per job: definition, checkpoints, totals, and the approval
record. The primary `Name` field plus 28 fields were created in the
first build; `Retry_Of` was in the field dictionary from the start but
was missing in the org until the Retry Run test found it, and was added
then.

| Field (API name) | Type | Purpose |
|---|---|---|
| Name | Single line | Run name |
| Rule_ID | Picklist | `L1` / `L2` / `D1` |
| Target_Module | Picklist | `Leads` / `Deals` (must match the rule) |
| Param_Created_From / Param_Created_To | Date | Optional `Created_Time` range (JST) |
| Run_Type | Picklist | `Normal` / `Retry` |
| Retry_Of | Lookup (Backfill_Runs) | Original Run of a Retry Run |
| Status | Picklist | `Draft` / `DryRunning` / `DryRunDone` / `Queued` / `Running` / `Paused_Budget` / `Paused_RateLimit` / `Paused_Manual` / `Paused_Failure` / `Failed` / `Completed` / `CompletedWithErrors` / `Cancelled` |
| Boundary_Max_ID | Single line | Frozen upper id of the range |
| Config_Fingerprint | Single line | `Rule_ID\|Rule_Version\|Boundary_Max_ID\|Effective_Date` |
| Rule_Version | Single line | Must equal the code constant (`L1-v1` / `L2-v1` / `D1-v1`) |
| Effective_Date | Date | Frozen evaluation date |
| Time_Offset_Minutes | Number | Clock correction used for the dry run |
| DryRun_Cursor / Exec_Cursor | Single line | Per-phase checkpoints (last processed id) |
| Pending_IDs | Multi-line (large) | Comma-separated ids for a Retry Run (max 1,000) |
| Lock_Token | Single line | Set during an invocation; used to detect overlap |
| Evaluated_Count / Success_Count / Failure_Count | Number | Execution totals |
| Skipped_NoChange_Count / Skipped_Stale_Count / Missing_Count | Number | Execution totals |
| DryRun_Evaluated / DryRun_Changed | Number | Dry run totals |
| DryRun_At | DateTime | Dry run completion time |
| DryRun_Summary | Multi-line (large) | Dry run summary text |
| Approved_By | User | Approver (set in the UI; `bf_approveRun` does not set it) |
| Approved_At | DateTime | Set by `bf_approveRun` |
| Error_Summary | Multi-line (small) | Pause reasons and audit lines (newest 1,900 characters kept) |

The execution totals satisfy
`Success + Failure + NoChange + Stale + Missing = Evaluated`, which
`bf_validateRun` checks. `EXCLUDED` records are counted as NoChange.

### Backfill_Logs (custom module)

Batch rows (the intent ledger) and one row per failed record. There is
no row per successful record.

| Field (API name) | Type | Purpose |
|---|---|---|
| Name | Single line | Unique key built from the Run, phase, and batch |
| Run | Lookup (Backfill_Runs) | Parent Run |
| Log_Type | Picklist | `Batch` / `Failure` |
| Phase | Picklist | `DryRun` / `Execute` / `Retry` |
| Batch_No | Number | Sequence within the invocation |
| State | Picklist | `Intent` / `Outcome` (execution batches) |
| Evaluated_IDs | Multi-line (large) | Ids evaluated in the batch |
| Proposed_Detail | Multi-line (large) | `id:old->new` for changed records (cut at 30,000 characters) |
| Outcome_Detail | Multi-line (large) | `id:SUCCESS` / `FAIL:<code>` / `STALE` / `MISSING` / `NOCHANGE` / `EXCLUDED` |
| Last_ID_In_Batch | Single line | Last id of the batch |
| Batch_Evaluated / Batch_Success / Batch_Failure / Batch_Skipped | Number | Batch totals |
| Target_Identifier | Single line | Failure rows: the record id (`Record_ID` is a reserved name and cannot be created) |
| Error_Code | Single line | Failure rows: response code |
| Error_Message | Multi-line (small) | Failure rows: response message |

### Fields on the target modules

| Module | Field (API name) | Type |
|---|---|---|
| Leads | BF_Days_Since_Created | Number |
| Leads | BF_Age_Bucket | Picklist: `0-30` / `31-90` / `91-365` / `Over365` |
| Leads | BF_Registered_Date | Date (business date; used by the fixtures) |
| Leads | BF_Stale_Flag | Checkbox |
| Leads | BF_Stale_Reason | Single line (255) |
| Deals | BF_Overdue_Flag | Checkbox |
| Deals | BF_Overdue_Days | Number |

## Files

| File | Declaration | What it is |
|---|---|---|
| `bfRunnerCore.ds` | `string standalone.bf_runnerCore(String)` | The Runner (v2.0): dry run, execution, Retry Run mode, L1 inline |
| `bfEvaluateL2.ds` | `string standalone.bf_evaluateL2(String x5)` | L2 evaluator (pure, returns a Map as text) |
| `bfEvaluateD1.ds` | `string standalone.bf_evaluateD1(String x5)` | D1 evaluator (pure, returns a Map as text) |
| `bfValidateRun.ds` | `string standalone.bf_validateRun(String)` | Read-only check of a Run; ends with `VERDICT: OK` or `NG` |
| `bfApproveRun.ds` | `string standalone.bf_approveRun(String)` | `DryRunDone` -> `Queued` after re-checks |
| `bfControlRun.ds` | `string standalone.bf_controlRun(String, String)` | `stop` / `clearlock` / `requeue` / `cancel` |
| `bfRetryRun.ds` | `string standalone.bf_makeRetryRun(String)` | Creates a Retry Run from the Failure rows of a Run |
| `bfRunnerSchedule.ds` | `string standalone.bf_runnerSchedule()` | Schedule wrapper, standalone version for manual runs and tests |
| `bfRunnerScheduleSched.ds` | `void schedule.bf_runnerScheduleSched()` | Schedule wrapper, Schedule category version (same body, no return value) |
| `bfSeedPhase2.ds` | `string standalone.bf_seedPhase2()` | Fixtures: 5 L2 boundary leads, 1 account, 6 D1 deals |
| `bfSeedMoreLeads.ds` | `string standalone.bf_seedMoreLeads()` | Fixtures: 2,000 synthetic leads (`BFTEST2-0001` to `-2000`) for the scale test |

Header comments are working notes written during the build. A few of
them describe something as not yet measured that was measured later
(for example COQL on the custom modules, which works); the
Verification section below is the current state.

## Setup

1. **Connection.** Every `invokeurl` uses the literal connection name
   `"crm_module"`. Deluge requires a literal here, so either create a
   Zoho OAuth connection with that link name or replace the literal in
   every file. The functions call the record and COQL endpoints, so the
   connection needs `ZohoCRM.modules.ALL` and `ZohoCRM.coql.READ` (the
   connection in the author's org also had settings scopes).
2. **Data center.** URLs use `https://www.zohoapis.jp/crm/v8`, the
   endpoint for the JP data center. In another data center, replace
   the domain (for example `zohoapis.com`, `zohoapis.eu`,
   `zohoapis.in`, `zohoapis.com.au`); see
   https://www.zoho.com/crm/developer/docs/api/v8/multi-dc.html.
3. **Modules and fields.** Create `Backfill_Runs` and `Backfill_Logs`
   and the fields in the tables above, plus the `BF_` fields on Leads
   and Deals. The code addresses modules by API name. In the author's
   org the API names came from the display names (`Backfill_Runs`);
   the internal `CustomModuleNN` value shown in the UI is not an API
   name and is rejected with `INVALID_MODULE`. Check the API names in
   your org before saving the functions.
4. **Org-specific values.** Replace the L2 terminal list, the D1 Closed
   list, and the fixture values (Lead Status values, the Deals stage
   and `Pipeline` value) with your org's picklist values. If you change
   a rule, change its version constant in `bfRunnerCore`,
   `bfValidateRun`, and `bfApproveRun` together.
5. **Clock correction.** In the author's org, `zoho.currenttime`
   returned a wall clock 17 hours behind JST (16 hours during US
   daylight saving time). Every function corrects for this with a
   fixed +17h / +16h rule and writes ISO times with `+09:00`. Check
   what `zoho.currenttime` returns in your org before trusting any date
   computed by this code; do not copy the offset blindly.
6. **Function order.** Save `bf_evaluateL2` and `bf_evaluateD1` first
   (the Runner calls them), then `bf_runnerCore`, then the helpers
   (`bf_validateRun`, `bf_approveRun`, `bf_controlRun`,
   `bf_makeRetryRun`), then the schedule wrappers. The seeders are
   only for testing.
7. **Schedule.** Only Schedule category functions can be chosen on the
   Schedules screen; a standalone function is not offered. Assign
   `bf_runnerScheduleSched`. The shortest interval offered was
   Hourly, so a Run that needs several invocations advances at most
   400 records per hour per Run through the schedule. Each firing
   picks up to 2 Runs in `Queued` or `Paused_Budget`, oldest first, and
   runs them one after the other. Schedule names cannot contain `(`.

## Operations

All functions take the Run record id as text and can be executed from
the function editor.

1. **Create a Run** in `Backfill_Runs`: `Rule_ID`, `Target_Module`,
   `Rule_Version` equal to the code constant (for example `L2-v1`),
   `Run_Type = Normal`, optional `Param_Created_From/To`, and
   `Status = DryRunning`. Leave `Effective_Date` empty to use today, or
   set it to evaluate as of a fixed date.
2. **Dry run.** Execute `bf_runnerCore(<run id>)` until the Run reaches
   `DryRunDone` (each call covers up to 400 records). The schedule
   wrapper does not pick up `DryRunning` Runs. Review `DryRun_Changed`
   and the `Phase = DryRun` rows in `Backfill_Logs`.
3. **Validate.** `bf_validateRun(<run id>)` is read-only. It checks the
   status value, rule version, fingerprint, the count identity,
   dry run age, and the lock, and ends with `VERDICT: OK` or
   `VERDICT: NG issues=<n>`.
4. **Approve.** `bf_approveRun(<run id>)` sets `Approved_At` and
   `Status = Queued`, or returns `REFUSED` with the reasons and writes
   nothing. Record `Approved_By` in the UI if you need it.
5. **Execute.** The schedule wrapper picks the Run up, or execute
   `bf_runnerCore(<run id>)` manually. The Run goes `Running` ->
   `Paused_Budget` between invocations -> `Completed`, or
   `CompletedWithErrors` if any record failed.
6. **Control.** `bf_controlRun(<run id>, <action>)`:
   - `stop`: `Running` / `Queued` / `Paused_Budget` /
     `Paused_RateLimit` -> `Paused_Manual`, lock cleared. A running
     invocation honours it at its next page boundary.
   - `clearlock`: clears a lock left by an invocation that died.
   - `requeue`: any `Paused_*` -> `Queued` (refused while a lock is
     held).
   - `cancel`: any non-terminal status -> `Cancelled`. Records already
     written stay as written; there is no rollback. The old values are
     in `Proposed_Detail`.
   Each applied action appends an audit line to `Error_Summary`.
7. **Retry.** After a Run has stopped with failures, fix the cause and
   execute `bf_makeRetryRun(<run id>)`. It creates `<name>-RETRY` as
   `Draft`, writes and reads back every inherited field, and only then
   sets `Queued`. The Runner processes it in `Pending_IDs` mode with
   `Phase = Retry` in the ledger. Records that are already correct
   come back as NoChange.

## Verification

All numbers below were measured in the author's org on 2026-11-10
(Phase 1) and 2026-11-12 (Phase 2) with synthetic records.

**Trigger suppression (positive control).** A temporary workflow rule
was set up to write a marker on edit. A PUT without `trigger` fired it
(1 criteria match, 1 update); the same kind of PUT with
`"trigger": []` wrote the values and did not fire it. The rule was
deleted afterwards. This covers the invokeurl PUT path used here.

**Phase 1: rule L1, Runner v1.2 / v1.3.**

- Dry run over 311 leads (250 synthetic fixtures plus 61 other test
  leads already in the org; only the new `BF_` fields are written):
  311 evaluated, 311 with changes (the `BF_` fields were empty), no
  PUT to Leads in the execution log, 4.79 s.
- Execution: 312 records written, 0 failures. The extra record was a
  lead created in the same org between the dry run and the execution
  whose id fell below the boundary; it was evaluated and written as
  part of the range. A test lead created with an id above the boundary
  stayed untouched.
- Day counts and buckets were checked on boundary fixtures at 0, 30,
  31, 90, 91, 365, and 366 days; all buckets were correct.
- Re-running the same Run from cursor 0: all 312 records came back
  `Skipped_NoChange`, with no PUT to Leads (10.60 s).
- A batch-selection bug in the first execution (v1.2) made the second
  batch of each page re-select the first 100 ids. The stale check
  rejected all 200 re-selected records, so nothing was written twice;
  the bug was fixed in v1.3 and the Run resumed and completed.

**Phase 2: rules L2 and D1, scale, and failure paths, Runner v2.0.**

- **L2:** dry run 326 evaluated / 195 changed; execution 326
  evaluated, 195 succeeded, 0 failed, 131 unchanged. Predicted and
  actual changes matched exactly. Boundary leads: 89 and 90 days not
  flagged (and not modified), 91 and 120 days flagged with a reason,
  120 days with a terminal status not flagged.
- **D1:** dry run 129 evaluated / 2 changed; execution 129 evaluated,
  2 succeeded, 0 failed, 127 unchanged. Deals 30 days and 1 day past
  `Closing_Date` were flagged with 30 and 1 days; today, +7, +30, and a
  closed deal 10 days past were not.
- **Scale:** 2,000 synthetic leads were added (20 POSTs of 100, 0
  errors). An independent count over the fixture formula predicted
  1,395 stale leads.
  - L2 dry run over 2,326 leads: 1,395 changes, equal to the
    independent count. 6 successful invocations, 53.77 s in total.
  - L2 execution over 2,327 leads: 6 invocations (17.77 / 16.46 /
    14.07 / 15.22 / 13.99 / 14.18 s, 91.69 s in total), passing through
    `Paused_Budget` between them. Result: 1,396 succeeded (the 1,395
    plus one test lead created after the dry run whose id was below
    the boundary), 0 failed, 931 unchanged. The last batch ended at
    the boundary id with no larger id evaluated.
- **API credits:** the CRM API usage dashboard, trailing 24 hours at
  2026-11-12 12:14 JST, which covers that day's Phase 2 work including
  the scale dry run and execution: `bf_runnerCore` 298,
  `bf_controlRun` 20, `bf_approveRun` 16, `bf_makeRetryRun` 16,
  `bf_runnerSchedule` 10, `bf_validateRun` 2.

| Test | Result |
|---|---|
| Version mismatch (Run at `L2-v9`) | Refused with `REFUSED version mismatch`, 0 writes, `Paused_Failure`; `bf_validateRun` returned NG |
| Overlapping start (lock set by hand) | `SKIP: locked`, `LOCK DETECTED` line recorded, cleared with `clearlock` |
| Platform error mid-run | One scale dry run invocation ended with a Zoho "internal error". Nothing was saved, the cursor stayed at the previous checkpoint (800 evaluated), no lock was left, and the next invocation resumed and completed |
| Mixed failure | `BF_Stale_Reason` max length lowered to 20: the PUT was rejected with HTTP 400 (not truncated), the Run ended `CompletedWithErrors` with a Failure row, the target lead was unchanged. Length restored to 255 |
| Frozen evaluation date | The same lead was evaluated as 80 days (not stale, flag cleared) with `Effective_Date` 2026-11-01 and as 91 days (stale, flag set) with 2026-11-12 |
| Retry Run | Built from the Failure rows, `CREATED queued=true`, executed through the schedule wrapper (standalone version) to `Completed`; a record that was already correct came back NoChange. The first attempt stopped in `Draft` because `Retry_Of` did not exist yet; the read-back caught it |
| Helpers | `bf_validateRun` returned OK on completed Runs; `bf_approveRun` approved the scale, mixed-failure, and frozen-date Runs; all four `bf_controlRun` actions were exercised |

## Limitations

- **400 records per invocation.** The planned budget was 1,000
  evaluations per invocation; the binding limit in practice is the
  2-page cap (400 records). With an hourly schedule, 2,300 records take
  about 6 hours per phase; the scale test above was driven by running
  the Runner by hand.
- **Crash during a write was not reproduced.** The order "Intent row ->
  PUT -> Outcome row -> cursor" was confirmed in every execution log,
  but killing an invocation between the PUT and the Outcome row could
  not be done from the editor. The design is confirmed, the behaviour
  is not. Cursor and totals are saved once at the end of each
  invocation, so a dead invocation is re-evaluated from the previous
  checkpoint; records it already wrote would then come back as
  NoChange, and its `Intent` rows remain in the ledger. Reconciling those rows is a
  manual step; the code does not do it automatically.
- **Timed schedule firing was not verified.** The Schedule category
  function saved, calling the Runner from it saved, and it can be
  selected on the Schedules screen. No schedule was created because
  the org was shared with other projects.
- **Write race (TOCTOU).** Between the stale re-read and the PUT there
  is a short window in which a person could edit the record, and the
  PUT does not use a conditional header. This was accepted as a
  residual risk because the engine writes only its own `BF_` fields and
  was run on synthetic data. A version that writes real business
  fields must close this window (for example with a conditional
  update) and needs its own design review.
- **Three fixed rules.** L1, L2, and D1 only. No arbitrary rules, no
  free-form conditions, no actions other than field writes.
- **Simplified safety rules.** Only the 200-failures-per-Run stop is
  implemented (no failure-ratio stop). Rate limits and other platform
  errors pause as `Paused_Manual`; the `Paused_RateLimit` status
  exists but is not set by the Runner. The lock is detection only,
  with no expiry. The fingerprint does not include the `Created_Time`
  range parameters.
- **Clock correction is fixed, not derived.** See Setup step 5.
- **Rule L1 was verified on Runner v1.3.** v2.0 keeps the L1 path
  unchanged, but the Phase 1 L1 runs were not repeated on v2.0.
- **Scale.** Verified up to 2,327 records per Run. Larger volumes were
  not tested.

## Deluge gotchas (measured)

- **Standalone functions return string only.** Declaring a Map return
  was refused on save with `Invalid return Type Map. Category
  standalone returns string`, and `void` with the same message for
  void. The evaluators return `map.toString()` and the Runner calls
  `.toMap()`; boolean values survived the round trip.
- **`zoho.crm.updateRecord` can silently drop a bundled update.** One
  call with 11 fields returned 202 and applied nothing; the same values
  sent in small groups all applied. Every important save here is split
  into small groups and read back.
- **Pipeline takes the `actual_value`.** Sending `"Standard"` failed
  with `MAPPING_MISMATCH`; the value that works is
  `"Standard (Standard)"`, which is also what COQL returns. The
  settings endpoint is `GET /settings/pipeline?layout_id=...`.
- **`Last_Activity_Time` moves on a `BF_`-only write**, with triggers
  suppressed. Its initial value also depends on how the lead was
  created (creation time for a UI-created lead, null for an
  API-created one).
- **Ids are not assigned in time order.** A record created later can
  have a smaller id. Range by id, and describe the range as an id set.
- **Shortening a field's max length gives no warning** even when
  existing values are longer, and a PUT with a longer value is rejected
  with HTTP 400 rather than truncated.
- COQL works on custom modules by API name; no rows comes back as
  HTTP 204 with an empty body.
- `invokeurl` needs the connection name as a literal.

## Status

Working prototype built as a portfolio evidence pack. Not client work,
and not a packaged product. License: TBD by the author.
