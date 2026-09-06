# dbt ownership and implementation contract

Contract release: **1.2**. Follow the authority order in `README.md`. This document
implements the spec, metric definitions and dataset contracts; it cannot override
them. Code belongs in the separate dbt repository, not here or in the generator.

## Responsibilities

| Layer | Owns |
|---|---|
| Python generator | Synthetic ATS/HR CSVs and source consistency checks, under `raw-data-generation-contract.md` |
| dbt | Source resolution, event/state flags, reporting eligibility, durations, maturity, risk bands, yield lookup, requisition roll-ups, marts and business tests |
| Power BI | Ratios of sums, medians over governed fact columns, filters and presentation |

Python may simulate ATS quantities/statuses, but dbt independently derives analytics
fields from the supplied source events. Keep raw IDs/dates; do not substitute status
text for dated acceptance/start/loss events. A validated as-of status can define
current active pipeline. That is a current-state measure, not a historical event.

Use dbt Core with local DuckDB for this portfolio. Keep a small staging/intermediate/
fact/mart layout. No Semantic Layer, extra platform or generalized framework is
required. Store mart rates/medians only as hidden validation references; displayed
rates are ratios of sums and displayed medians come from facts.

## Build dependencies

These are build inputs, not merely logical relationships. Final-model comparisons
belong in tests after the referenced models exist; they must not become upstream
model dependencies. dbt derives execution order from `ref()`.

| Model | Build inputs and responsibility |
|---|---|
| Governed seeds | `dim_recruiting_stage`, `dim_hiring_constraint`, `ref_risk_band` from their YAML seed rows |
| `ref_reporting_config` | One-row model selecting `dbt_project.yml` vars; not a second seed |
| Dimensions | Date/cohort spines from config; BU/JF/JL from source lookups |
| Staging models | Normalize raw CSV names, types and source codes; preserve source events |
| `int_requisition__resolved_snapshot` | Latest snapshot on/before as-of, one row per requisition; fail conflicting same-date snapshots |
| `int_offer__resolved_acceptance` | Resolve versions into one governed cycle per application; retain all applicable loss dates |
| `audit_offer__multi_accepted_version` | Audit every source application with multiple accepted versions before resolution |
| `int_application__events` | Staged applications, resolved offers/requisitions and resolved HR events; derive flags, eligibility and Time to Fill, without yield |
| `int_stage_event__sequenced` | Stage history, application events and stage seed; sequence visits and derive completed intervals/conversion |
| `fct_application_stage_event` | Sequenced stage events; does not read final application/requisition facts |
| `mart_stage_yield` | Stage-event fact plus `int_application__events`; does not read final `fct_application` |
| `fct_application` | Application events plus stage-yield lookup |
| `int_requisition__pipeline_rollup` | Final application fact grouped to requisition; counts and uncapped expected fills |
| `fct_requisition` | Resolved requisitions, pipeline roll-up and risk seed; classify risk and cap forecast |
| `fct_hire_outcome` | Started applications plus resolved HR events and cohort dimension |
| Four `mart_exec_*` models | Corresponding facts; demand mart also stores forecast at THD-month grain |

## Core implementation rules

- **Offer resolution:** collapse administrative revisions of one cycle and preserve
  its earliest valid acceptance. Quarantine multiple genuine cycles. The audit
  records `application_id`, `accepted_version_count`, `resolution` and notes.
  Allowed resolutions are `administrative_revision` and `quarantined`. Fail if a
  multi-version application is missing from the audit, has an invalid resolution,
  or reaches the final fact while quarantined. Multiple legitimate revisions alone
  are not a failure. Inspect quarantined records rather than silently dropping
  their impact on reconciliation. Repeat candidate/requisition attempts use distinct
  application IDs; only multiple acceptances within one application require quarantine.
- **Reporting eligibility:** derive `is_delivery_eligible = NOT is_cancelled` from
  the resolved requisition and carry it onto applications, stage events and hires.
  Delivery, pipeline and offer-context measures use it; quality retains all actual
  hires regardless of later requisition cancellation. Keep dates/events on excluded
  rows for audit. Do not delete history to make mart totals agree.
- **Historical attribution:** THD and segment keys reflect the latest snapshot at
  the as-of date. Reports can restate when those attributes or eligibility change.
  Preservation tests compare event IDs/dates and successful acceptance exits for
  the same applications. They do not demand unchanged totals under moving filters.
- **Maturity:** `dim_start_cohort` owns full-month maturity and latest/prior 12-month
  flags. The latest mature month is March 2026 at the configured as-of date. Quality
  facts have no active THD relationship. KPI and footer use the same latest-12 window.
- **Risk:** range-join days-to-TOAD to `ref_risk_band`; the seed alone owns boundaries
  and `is_at_risk`. Test contiguous, non-overlapping ranges and exactly one match.
  TOAD is source data and must not be overwritten.
- **Forecast:** baseline yield is trained on non-active application outcomes known
  at the as-of date, using `is_active_fill` as the observed label. Pending starts
  still count as active fills: this is a provisional as-of label, not a fully
  observed eventual outcome. Use `is_yield_training_eligible` explicitly so this
  training choice is separate from current pipeline state. No maturity correction
  or machine-learning model is required in this phase.
- **Fallback:** retain stage at every level: `bu_jf_jl -> jf_jl -> jf -> all`.
  Choose the first sufficiently supported level. Global stage yield can be used
  below the threshold only when it has at least one observation; flag low support.
  If an active candidate's stage has zero global observations, fail forecast
  validation with the affected stage/segment. Never invent a zero/100% yield.
  Sum applied candidate yields per requisition, then cap at source openings.
  This is a capped planning estimate, not an on-time forecast or an exact
  probabilistic expectation. Existing pending fills are assumed to remain filled;
  unknown future applicants are not included.

## Configuration and export

Declare as-of date, coverage, targets and thresholds once in `dbt_project.yml` vars.
Macros read vars; `ref_reporting_config` exposes the same values to Power BI. Risk
boundaries remain in their seed. Business calculations must not read the system clock.

Record contract release/commit and the input generator manifest in the dbt run
summary. Export validated analytics CSVs or Parquet matching the schemas. Publish
exports only after required tests pass; a failed build must not look like a successful
Power BI refresh. Re-running unchanged input/configuration reproduces results.

## Tests and portfolio evidence

Dataset YAML is a custom contract, not directly executable dbt YAML. Map columns to
dbt `name`/`data_type`, and configure supported model contracts. Implement uniqueness,
nullability, accepted values and relationships as explicit tests; contract enforcement
alone does not prove metric correctness or every database constraint.

Use small SQL tests for business identities and the acceptance examples in spec
section 15. Store offending rows for reconciliation failures. Test event preservation
with paired before/after fixtures, changing only the loss event and corresponding
source seat state. Broader source changes may legitimately restate reporting totals.

Minimum evidence: output-grain checks, requested = filled + open, accepted = active
fills + losses, starts <= fills, risk reconciliation, forecast cap/fallback, as-of
limits, and quality numerator/denominator alignment. Run these in the dbt repository's
CI; do not build a testing framework in this contracts repository. Add one exposure
for the Executive Summary and link a successful run, lineage image and report
screenshot when available. The Python and dbt repositories each need one reproducible
run command, not additional infrastructure.
