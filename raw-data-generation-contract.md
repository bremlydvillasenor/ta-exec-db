# Raw ATS and HR data generation contract

Contract release: **1.3**. Follow the authority order in `README.md`: the spec,
metric definitions and analytics dataset YAML take precedence. The wireframe is
last. This contract lives here; generator code lives in the separate raw-data repo.

## Purpose and boundary

Build a small **uv + Python + Polars** project generating realistic synthetic source
CSVs for the Executive Summary. Python simulates ATS/HR records and validates source
consistency. dbt produces dimensions, facts, marts, metric flags, durations, risk bands
and forecast yields. Include HR events only for actual starts and early attrition.

Python may simulate source statuses, requisition quantities and operational dates.
It must not export `is_active_fill`, maturity flags, Time to Fill, risk bands,
forecast probabilities, KPI totals, or `dim_`, `fct_`, `mart_` datasets. Checking source
consistency is permitted; it does not make Python the owner of analytics derivations.

## Coverage and reproducibility

- Actual events: **2024-01-01 through 2026-05-31**, inclusive.
- Fixed as-of date: **2026-05-31**. Never use the system clock for business dates.
- THD may extend through **2027-05-31**. Future planned starts are separate from actual starts.
- Configure the random seed; use deterministic IDs and stable row order. Identical
  configuration and seed reproduce identical CSV content.
- Write complete extracts to `data/raw/`. Re-runs replace outputs rather than append
  duplicates. No streaming or orchestration platform is required.
- Do not export future actual outcomes even if simulated internally.

## Required source files

These are minimum logical contracts. New implementations use these filenames and
columns. An existing generator may retain equivalent names with a complete documented
mapping into dbt staging. Extra columns must support a required behavior.

| File | Grain / unique key | Minimum columns |
|---|---|---|
| `requisition_snapshots.csv` | Requisition at snapshot date; `(requisition_id, snapshot_date)` | `requisition_id`, `snapshot_date`, `requisition_status`, `approval_date`, `target_hire_date`, `target_offer_acceptance_date`, `requested_positions`, `openings_position`, `cancelled_positions`, `business_unit_code`, `job_family_code`, `job_level_code`, `hiring_constraint_code`, `updated_at`, `extracted_at` |
| `applications.csv` | Application attempt; `application_id` | `application_id`, `candidate_id`, `requisition_id`, `application_date`, `application_status_current`, `current_stage_code`, `rejected_date`, `withdrawal_date`, `disposition_reason`, `updated_at`, `extracted_at` |
| `offers.csv` | Current offer per application with an issued offer; `application_id` | `application_id`, `requisition_id`, `offer_status_current`, `offer_extended_date`, `offer_accepted_date`, `offer_declined_date`, `offer_withdrawn_date`, `offer_rescinded_date`, `candidate_renege_date`, `planned_start_date`, `updated_at`, `extracted_at` |
| `stage_history.csv` | Stage visit; `stage_event_id` | `stage_event_id`, `application_id`, `stage_sequence_number`, `stage_code`, `stage_entry_date`, `stage_exit_date`, `exit_reason`, `updated_at`, `extracted_at` |
| `worker_events.csv` | Actual HR event; `worker_event_id` | `worker_event_id`, `worker_id`, `application_id`, `event_type`, `event_date`, `termination_reason`, `updated_at`, `extracted_at` |

Provide small `business_units.csv`, `job_families.csv` and `job_levels.csv` lookup
files, each with its corresponding `*_code`, `*_name`, `updated_at` and `extracted_at`. Codes are stable, unique
and resolve every requisition. Use the recruiting-stage and hiring-constraint codes
from the analytics schema seeds; do not create competing categories.

## Field conventions

- UTF-8 CSV, headers, snake_case columns. IDs/codes are strings; quantities and
  sequences are integers; dates use `YYYY-MM-DD`; empty cells mean null. Real names
  and contact details are unnecessary.
- IDs, foreign keys, sequences, status/stage codes and requisition dates/quantities
  are required. Optional event dates are null when no event occurred. Disposition
  reason is required for rejected/withdrawn applications.
- Worker `event_type` is `start` or `termination`. The same application and worker
  link both events for one employment spell. Only termination requires a reason.
- An issued offer requires `offer_extended_date` and `offer_status_current`.
  Allowed offer statuses: `pending`, `accepted`, `offer_declined`, `offer_withdrawn`,
  `offer_rescinded`, `candidate_renege`. `accepted` includes pending starts and actual
  starts; actual start remains an HR event. Application status is tracked separately.
  An application with no offer has no `offers.csv` row. Planned start is optional.
  Loss dates and acceptance nullability must match the documented lifecycle.

## Extraction and update timestamps

| Column | Meaning | Requirement |
|---|---|---|
| `updated_at` | Last modification of this source record, including status/date corrections | Generator populates it on every raw file. For real exports it is nullable only when the source does not supply a reliable change timestamp; use a full comparison/reload in that case. |
| `extracted_at` | When the complete source extract was produced | Required on every row in every raw file, including lookups and events. |

Use UTC ISO 8601 timestamps, for example `2026-05-31T23:59:59Z`. Business dates stay
`YYYY-MM-DD`. `updated_at <= extracted_at`; never change `updated_at` solely because
an unchanged row was exported again. A source edit advances its update timestamp;
acceptance/start dates keep their own business meaning. Stage rows update when an
exit is recorded; HR event rows use their creation/correction time; lookup rows use
the last source label/code change. The source must expose changes to each exported
row; a change on another table does not automatically update this row's timestamp.

For deterministic synthetic output, configure `extracted_at` explicitly (default
`2026-05-31T23:59:59Z`) and record it in the manifest. All files in the generated
batch use that value. Same seed/configuration, including extraction time, yields
the same CSV content. Configure the synthetic business cutoff at end of the as-of
day; generated source changes must be known by that cutoff. For earlier requisition
snapshot rows, their source state/update must be known by their snapshot date.

An actual export may be produced after the business as-of date. Extraction metadata
is allowed to be later; it must not advance maturity, risk or event eligibility.
A current report cannot reconstruct an earlier as-of state unless that state was
retained. Do not label today's changed source state as a historical snapshot merely
by filtering its updated_at. Use a retained historical extract for historical replay.

The default is a **complete extract**, followed by validation and replacement of
current staging. To demonstrate changes across runs, optionally retain dated folders
with a manifest per complete batch. Reading two folders does not mean two offers:
select the intended batch before loading current state.

Incremental loading is optional. Match on the source keys above; insert unseen keys
and replace existing rows only for newer source updates. Use a short lookback on
per-source watermarks, and periodically compare the full extract to catch changes
outside that lookback. Equal key/update time with different business values is a
validation error; extraction time alone does not make a stale record newer. Replaying
the same extract is idempotent. For requisitions, resolve the intended snapshot before
producing one current row per requisition. Derived facts/marts can be rebuilt fully.

`offers.csv` must include **all issued offers**, not only currently accepted ones.
Keep declined, withdrawn, rescinded and reneged rows with explicit dates. If a real
Workday report omits them, a supplementary outcome export must supply them. Compare
previously delivered offer IDs with complete current coverage; missing IDs without
explicit outcomes are a coverage failure, not evidence of a particular loss. Do not
silently remove their historical acceptance or keep counting them as confirmed active
fills. Resolve coverage before publishing affected delivery/forecast outputs. An
upsert and updated_at alone cannot detect deletions or supply missing event history.

## Required simulation behavior

1. **Requisitions.** Status is `open`, `filled` or `cancelled`. Emit at least one
   snapshot per requisition on the as-of date, with earlier snapshots for a
   small subset to exercise resolution. Each snapshot reflects simulated state at
   that date. Requested positions are net demand, excluding cancelled seats. On
   non-cancelled requisitions, requested seats equal active accepted seats plus
   openings. Filled requisitions have zero openings. Fully cancelled requisitions
   have zero requested/open seats and no active accepted offers. A termination
   after start does not reopen the original seat; replacement demand is a new req.
   Where some seats already started, cancel only remaining unfilled seats and keep
   fulfilled demand rather than marking the entire requisition cancelled.
2. **Targets and constraints.** Generate THD and TOAD as source fields, with TOAD
   no later than THD and realistic notice periods. dbt consumes TOAD unchanged.
   Primary constraints belong to requisitions and apply to all their open seats.
3. **Applications.** Use `active`, `offer_accepted`, `started`, `rejected`,
   `withdrawn`, `offer_declined`, `offer_withdrawn`, `offer_rescinded` and
   `candidate_renege`, consistent with dated events. Active applications belong
   only to open requisitions and have one open stage interval. When a requisition
   closes, disposition remaining active applications with dated reasons. Document
   a simple inactivity policy; do not leave old unattended applications active
   indefinitely or infer staleness from application year alone.
4. **Stages.** Use governed stage order. The default scenario progresses sequentially
   without skipped/repeated stages. Visits are ordered and non-overlapping;
   sequence number resolves same-day ties. Successful progression has null
   `exit_reason`; unsuccessful exits use the schema's pre-acceptance loss terms.
   Acceptance closes Offer successfully. Later rescind/renege events never rewrite
   that stage exit. dbt derives conversion flags.
5. **Current offers.** Emit one current row per application with an issued offer.
   Change planned start or offer status on the same row and advance `updated_at`;
   preserve original acceptance after a loss. Do not generate offer version/cycle
   identifiers. Each application supports at most one acceptance. Reapplication
   after a loss uses a new application ID; candidate/requisition pairs can repeat
   without overlapping active fills. Include pending, accepted and lost offers.
6. **Losses and replacement.** Before acceptance: employer `offer_withdrawn` or
   candidate `offer_declined`, with no accepted date. After acceptance and before
   start: employer `offer_rescinded` or `candidate_renege`, preserving acceptance.
   Reopen or cancel the lost seat in the matching requisition snapshot. Include
   some replacement applications. One seat can accumulate multiple historical
   acceptances, but has at most one active fill.
7. **Employment.** Only actual starts by the as-of date produce HR start events;
   pending starts have none. Terminations follow starts and never become reneges.
   Include exits within 60 days, later exits, retained hires and immature cohorts.
   Include hires across the 12 matured months through March 2026 and the previous
   12 months to demonstrate both quality windows.

## Story and scale

Choose manageable volumes sufficient to demonstrate differences across BU, Job
Family and Job Level. Document the scale and scenario assumptions; exact KPI totals
are not prescribed. Include constrained demand, a stage bottleneck, differing
pipeline strength and early-attrition variation. Include open demand across the
risk boundaries and future THD months, plus well-supported and sparse forecast
segments. Generate coherent events, dates and probabilities; do not force a perfect
speed/attrition correlation or independently tune CSVs to match wireframe numbers.

## Validation and handoff

Check unique keys within each extract, foreign keys, date chronology, as-of limits, timestamps, source quantities
and lifecycle consistency before marking a run successful. Failures identify IDs.
Keep deliberately invalid cases in a small separate fixture directory.

Deliver the generator, dependency lockfile, raw CSVs, concise README, and a
`manifest.json` with contract release/commit, generator commit, random seed,
effective configuration (including extracted_at and timestamp availability), file row counts/checksums and validation status. Document
one command to reproduce the dataset. dbt must consume these files without reading
generator internals. No new data platform or elaborate framework is required.

For an optional incremental demonstration, provide two small dated full extracts:
keep an unchanged offer, update another offer's planned start, record a renege while
preserving acceptance, and add a new offer. Expected results: changed records have
newer updated_at, every second-batch row has the new extracted_at, and a second load
of that batch changes no counts. No historical version-resolution model is needed.
