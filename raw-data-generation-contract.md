# Raw ATS and HR data generation contract

Contract release: **1.2**. Follow the authority order in `README.md`: the spec,
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
| `requisition_snapshots.csv` | Requisition at snapshot date; `(requisition_id, snapshot_date)` | `requisition_id`, `snapshot_date`, `requisition_status`, `approval_date`, `target_hire_date`, `target_offer_acceptance_date`, `requested_positions`, `openings_position`, `cancelled_positions`, `business_unit_code`, `job_family_code`, `job_level_code`, `hiring_constraint_code` |
| `applications.csv` | Application attempt; `application_id` | `application_id`, `candidate_id`, `requisition_id`, `application_date`, `application_status_current`, `current_stage_code`, `last_updated_date`, `rejected_date`, `withdrawal_date`, `disposition_reason` |
| `offer_versions.csv` | Offer version; `(application_id, offer_cycle_id, offer_version)` | `application_id`, `offer_cycle_id`, `offer_version`, `version_date`, `offer_extended_date`, `offer_accepted_date`, `offer_declined_date`, `offer_withdrawn_date`, `offer_rescinded_date`, `candidate_renege_date`, `planned_start_date` |
| `stage_history.csv` | Stage visit; `stage_event_id` | `stage_event_id`, `application_id`, `stage_sequence_number`, `stage_code`, `stage_entry_date`, `stage_exit_date`, `exit_reason` |
| `worker_events.csv` | Actual HR event; `worker_event_id` | `worker_event_id`, `worker_id`, `application_id`, `event_type`, `event_date`, `termination_reason` |

Provide small `business_units.csv`, `job_families.csv` and `job_levels.csv` lookup
files, each with its corresponding `*_code` and `*_name`. Codes are stable, unique
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
- Offer cycle/version identifiers, `version_date` and `offer_extended_date` are
  required. Planned start is optional. Repeated accepted dates on administrative
  versions describe one acceptance event.
- `last_updated_date` is the latest actual ATS activity date (stage, offer or
  disposition), between application date and the as-of date. HR events are separate.

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
5. **Offer versions.** Include single offers and administrative revisions within
   the same cycle, preserving its earliest valid acceptance. Each application has
   at most one genuine accepted cycle in the normal dataset. Reapplication after
   a loss uses a new `application_id`; candidate/requisition pairs may repeat but
   cannot hold simultaneous active fills for the same candidate. Ambiguous multiple
   cycles belong in a separate quarantine test fixture, not normal exports.
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

Check unique keys, foreign keys, date chronology, as-of limits, source quantities
and lifecycle consistency before marking a run successful. Failures identify IDs.
Keep deliberately invalid cases in a small separate fixture directory.

Deliver the generator, dependency lockfile, raw CSVs, concise README, and a
`manifest.json` with contract release/commit, generator commit, random seed,
effective configuration, file row counts/checksums and validation status. Document
one command to reproduce the dataset. dbt must consume these files without reading
generator internals. No new data platform or elaborate framework is required.
