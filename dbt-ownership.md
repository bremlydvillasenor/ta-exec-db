# dbt ownership of business logic and transformations

Companion document to `spec.md`, `metric-def.yaml`, `wireframe.html` and `schemas/`.
It recommends which calculations should move into dbt, which should stay in Power BI,
and which should stay in Python.

Fixed reporting as-of date: **2026-05-31**. Nothing in any layer reads the system clock.

## Repository boundary

This document is a **contract for a downstream implementation**, not a description of code
in this repository. `ta-exec-db` holds the specification, the wireframe, the metric
definitions and the dataset contracts; no Python or dbt project files belong here.

| Repository | Owns |
|---|---|
| **This repository (`ta-exec-db`)** | Required grains, columns, metrics, business rules, validation tests, and the dbt architecture the implementation must follow |
| **Separate implementation repository** | Python synthetic-source generation, the dbt models, the executable tests, orchestration, and production of the CSV / Parquet outputs |
| **Power BI** | Consuming the validated outputs, and owning filter-responsive ratios, medians and presentation |

Every model name, project layout and test in this document describes what the
implementation repository must build. Read them as requirements, not as an inventory of
files.

---

## 1. Short answer

Time to Fill is a good example, but it is one of about forty derivations in this project
that belong in dbt. The pattern is bigger than a single metric: **almost every rule in
`spec.md` section 5, section 6, section 7 and section 12 is a row-level derivation, a
cross-table roll-up, or a validation rule — and all three are dbt work.**

What Power BI should keep is small and specific: dividing two additive columns under the
current filter context, taking a median over a governed row-level column, and everything
about presentation.

What Python should keep is also small: generating the synthetic source data that
simulates the ATS and HR systems. That is a source system, not a transformation.

The practical benefit is governance. Today the metric contract lives in YAML as prose. In
dbt it becomes executable: the derivation is a model, the rule is a test, and the pipeline
fails when the rule breaks. A TA leader asking "why did Fill Rate drop?" gets an answer
from a tested lineage instead of from someone re-reading a DAX measure.

---

## 2. The ownership rule

Use this decision test on any calculation before deciding where it lives.

| Question | If yes |
|---|---|
| Is it a row-level derivation (a flag, a date difference, a classification)? | **dbt** |
| Does it need the configured as-of date? | **dbt** |
| Does it need a grain the report model cannot reach (per application, per requisition)? | **dbt** |
| Would it require a fact-to-fact relationship in Power BI? | **dbt** |
| Is it non-additive, so it cannot be summed back up (a per-requisition cap, a deduplication)? | **dbt** |
| Is it a business rule that must be identical in every visual? | **dbt** |
| Is it dividing two additive columns in the current filter context? | **Power BI** |
| Is it a median over a governed row-level column? | **Power BI** |
| Is it formatting, sorting, conditional highlighting, or a target comparison label? | **Power BI** |
| Is it creating the raw source records? | **Python** |

**One definition, not one location.** Governed business logic must not be independently
reimplemented across layers — one definition, in one place, that the others consume.
Reference calculations may legitimately appear in more than one place: the marts store
row-grain rates and medians precisely so a build can prove the mart and the fact agree.
Those are validation values, hidden from the report. The figure the executive actually
reads is still calculated once, by the layer that owns it.

The three-layer split:

| Layer | Owns | Examples from this project |
|---|---|---|
| **Python** | Simulating the source systems | Generating requisitions, applications, stage histories, offer versions, HR start and termination events |
| **dbt** | Every transformation between source and mart, plus validation | `is_active_fill`, `days_to_toad`, cohort maturity, stage conversion, forecast yield, all marts, all reconciliation tests |
| **Power BI** | Aggregation under filter context and presentation | `DIVIDE(SUM(...), SUM(...))`, `MEDIAN(...)`, KPI cards, the "Watch" call-out, low-volume flagging at visual grain |

---

## 3. Where logic would leak today

Three risks are visible in the current design. Each one is the reason a specific
recommendation below exists.

**Risk 1 — the vocabulary collapses back into one word.** The contract separates
*accepted offer*, *active fill* and *hire*. If those three flags are derived in Python but
not tested, or re-derived in DAX per visual, one report page will eventually count
accepted offers as hires again. That is exactly the defect contract v1.1 was written to
remove.

**Risk 2 — Power BI is asked to do something the model forbids.** `filled_positions`,
`accepted_offer_events`, `started_positions` and `active_pipeline_applications` are counts
of applications stored at requisition grain. Producing them in Power BI would need a
relationship from `fct_requisition` to `fct_application`, which `schemas/README.md`
explicitly forbids because it duplicates `openings_position`. So these roll-ups have no
valid home except upstream.

**Risk 3 — non-additive results get averaged.** The forecast cap
(`LEAST(expected_pipeline_fills_uncapped, openings_position)`) and the offer-version
resolution are non-additive. Applied at the wrong grain they give a different answer at
every level of a report hierarchy, silently.

---

## 4. Recommended dbt scope

Ranked by how much damage the calculation does if it is left in Python or Power BI. The
first three are equally foundational and none of them is optional: cohort maturity decides
who is even eligible for the quality KPI, offer resolution decides the grain every
offer-based count depends on, and the event and state flags decide what each count means.
A weakness in any one of them invalidates the others.

### 4.1 Cohort maturity and the rolling-12 window (`dim_start_cohort`)

**What:** the full month spine from January 2024 to the as-of month, plus
`cohort_maturity_date` (month end + 60 days), `is_fully_matured_start_month`,
`matured_cohort_sequence`, `kpi_cohort_window` (`latest_12` / `prior_12` / `outside`) and
`is_latest_fully_matured_cohort`.

**Why dbt:** this is the single highest-value model in the project. It is where the
statement "the latest fully matured cohort is March 2026" stops being a number someone
typed and becomes a result the pipeline derives from the as-of date. It also makes the
"quality visuals ignore the Target Hire Date slicer" behaviour *structural*: because the
dimension has no relationship to `dim_date`, no measure has to remember to call
`REMOVEFILTERS`.

**Risk otherwise:** if maturity is computed in DAX, an executive filtering the page to
"Jan–May 2026" quietly shortens the 60-day observation window, and the 60-Day Early
Attrition KPI reports a rate on hires who have not been observed long enough. The number
looks fine and is wrong — the worst kind of reporting error.

**dbt approach:** `dbt_utils.date_spine` at month grain, then window functions for the
sequence and window flags. Tests: exactly 12 rows with `kpi_cohort_window = 'latest_12'`,
exactly one row with `is_latest_fully_matured_cohort`.

---

### 4.2 Offer-version resolution to one governed acceptance

**What:** collapsing multiple source offer versions for one application (revised salary,
moved start date, re-issued letter) into the single governed acceptance — normally the
earliest acceptance date of the offer the candidate actually accepted.

**Why dbt:** `spec.md` 9.4 and `fct_application.yaml` both say this must happen
"upstream", but upstream is currently undefined. Make it a named model,
`int_offer__resolved_acceptance`, so the rule has one home, one owner and one test. The
whole one-acceptance-per-application assumption — and every count-based identity that
depends on it — rests on this model.

**Risk otherwise:** two accepted offer versions on one application double-count a seat.
`accepted_offer_events = filled_positions + lost_after_acceptance_positions` breaks, and
Fill Rate can exceed 100% for a business unit.

**The resolution rule.** Do not simply take the earliest accepted version. The model must
first establish what the source versions actually mean, then apply the rule in this order:

1. **Preserve the earliest valid acceptance event for the offer cycle that was ultimately
   accepted.** The governed acceptance date is the moment the candidate committed, not the
   date of the last paperwork.
2. **Collapse administrative revisions of that same accepted offer** — a corrected salary,
   a moved start date, a re-issued letter. These are versions of one event and must not
   create a second acceptance.
3. **Quarantine ambiguous cases**, where an application carries more than one distinct
   acceptance cycle that cannot be resolved as revisions of a single offer. Quarantined
   rows are held for review, not silently collapsed and not silently dropped.
4. **If genuine re-offer cycles are later introduced** — a candidate accepts, the offer is
   lost to a rescind or a renege, and the same candidate accepts again for the same
   requisition — build a separate offer-event fact with one row per offer event and an
   offer sequence number. Do not extend `fct_application` with a second set of offer
   columns.

**dbt approach:** deduplicate with `qualify row_number() over (partition by application_id
order by offer_accepted_date, offer_version)` after the revision-collapsing logic, not
instead of it. Two tests are needed, not one:

- a `unique` test on `application_id` in the resolved output, which proves the grain holds;
- an **audit test on the source** counting applications with more than one distinct
  accepted offer version, materialised with `store_failures: true`.

The second test is the one that matters. A unique test on the output only proves the
deduplication ran — it cannot tell you whether it discarded a real second acceptance.

---

### 4.3 The event and state flags on `fct_application`

**What:** `is_offer_accepted_event`, `is_active_fill`, `is_started`,
`post_acceptance_outcome`, `is_active_pipeline`, `has_final_outcome`,
`time_to_fill_days`, `active_stage_age_days`.

**Why dbt:** these flags *are* the governed vocabulary. Every metric in `metric-def.yaml`
names one of them. Deriving them once in dbt, with the contract's own rules as tests, is
what makes "no metric may be defined from an application status value" enforceable rather
than aspirational.

Time to Fill belongs here for a subtle reason worth stating: `time_to_fill_days` must stay
populated when an accepted offer is later rescinded or the candidate reneges. That is a
deliberate historical rule, and it is easy for a later "cleanup" in Python or a
`FILTER(is_active_fill)` in DAX to break it. As a dbt column with the test
`is_offer_accepted_event implies time_to_fill_days IS NOT NULL`, the rule defends itself.

**Risk otherwise:** the classic recruiting reporting failure — TA reports 953 "hires" that
are actually accepted offers, 40 of which have not started and 12 of which were lost. The
business plans headcount on a number that overstates reality.

---

### 4.4 Stage-event windowing and conversion (`fct_application_stage_event`)

**What:** `stage_sequence_number`, `stage_exit_date`, `is_completed`, `is_current_stage`,
`days_in_stage`, `active_stage_age_days`, `exceeded_sla`, `exit_reason` and above all
`advanced_to_next_stage`.

**Why dbt:** `advanced_to_next_stage` is the hardest derivation in the project. It needs
the next stage from `dim_recruiting_stage`, a look-ahead to the following stage-event row,
and a special rule for the Offer stage where the successful exit is the acceptance event —
which stays a success even after a rescind or a renege. That is a window function plus two
joins. Complex DAX could reproduce it, but it would be fragile, expensive to evaluate at
every visual, and wrong for this model — the contracts deliberately give Power BI no
relationship between the stage-event fact and the application fact. Doing it in Python
means re-implementing window logic that SQL already has.

**Risk otherwise:** the wireframe already shows the trap. Interview conversion is 43%, and
the wireframe notes it is "deliberately not calculated as 41 active offers ÷ 95 active
interviews". Those are two different populations — a current snapshot versus completed
historical movements. Dividing one by the other is the most common funnel error in TA
reporting, and it is only avoidable if the conversion numerator and denominator are
produced upstream as separate, additive counts.

**dbt approach:** `lead()` / `row_number()` over application and entry date; join
`dim_recruiting_stage.next_stage_code` from the seed. Test: offer-stage rows of
applications with `is_offer_rescinded` or `is_candidate_renege` still have
`advanced_to_next_stage = true`.

---

### 4.5 Requisition roll-ups (`fct_requisition`)

**What:** `filled_positions`, `accepted_offer_events`, `lost_after_acceptance_positions`,
`started_positions`, `active_pipeline_applications`.

**Why dbt:** these are application-grain counts stored at requisition grain. Power BI
cannot produce them without the forbidden fact-to-fact relationship. This is not a
preference — it is the only valid place for them.

**Risk otherwise:** if someone does create that relationship to "just get the counts",
`openings_position` is multiplied by the number of candidates joined to the requisition. A
requisition with 8 open seats and 30 active candidates reports 240 open positions. Every
risk and constraint visual on the page becomes wrong at once.

**dbt approach:** aggregate from `fct_application`, then test the identities:
`accepted_offer_events = filled_positions + lost_after_acceptance_positions`,
`started_positions <= filled_positions <= accepted_offer_events`, and
`requested_positions = filled_positions + openings_position` for non-cancelled rows.

---

### 4.6 Requisition snapshot resolution

**What:** picking exactly one source row per requisition — the latest snapshot on or
before the as-of date.

**Why dbt:** `spec.md` 9.2 warns that summing several snapshots of the same requisition
silently inflates demand. This is a deduplication, so it is non-additive and must happen
before anything aggregates.

**dbt approach:** `qualify row_number() over (partition by requisition_id order by
snapshot_date desc)` filtered to `snapshot_date <= as_of_date`, with a `unique` test on
`requisition_id`. If the source is a live table rather than a snapshot history, use a dbt
**snapshot** to build the history first.

---

### 4.7 Risk banding from a seed, not a CASE expression

**What:** `days_to_toad`, `risk_band_code`, `is_at_risk`, `at_risk_open_positions`.

**Why dbt:** `ref_risk_band` already defines the bands as data — `min_days_to_toad` and
`max_days_to_toad` per band. Load it as a dbt **seed** and assign the band by a **range
join** to that seed, not by a `CASE WHEN days_to_toad < 0 ...` written in SQL, Python and
DAX. Changing the bands then means editing the seed, and only the seed. Moving the at-risk
boundary from 14 to 10 days touches two band rows — `medium_risk` gains a new upper bound
and `on_track` a new lower bound — because the bands must stay contiguous. That is still one
governed file, reviewed as one change, and every downstream model and every visual picks it
up on the next run.

**The seed must be the only authority.** An earlier draft of this document claimed that a
seed edit keeps everything synchronised. It does not, because the project had a second
definition of the same rule: `at_risk_max_days_to_toad` in `ref_reporting_config`, with
`ref_risk_band` carrying the band boundaries and `is_at_risk` per band. Two definitions of
one threshold can only ever be *validated* against each other — a test detects the
disagreement after it happens, it does not prevent it.

The fix is to derive `is_at_risk` entirely from `ref_risk_band.is_at_risk` by joining
`days_to_toad` to the band ranges, and to remove the duplicate threshold from the reporting
configuration. **This change has been applied to the contracts**: the column and its seed
value are gone from `ref_reporting_config.yaml` (now v1.1), the `fct_requisition` test now
derives `is_at_risk` from the seed by range join, and the cross-check in `ref_risk_band.yaml`
that compared the two copies has been replaced by a shape test on the bands themselves.

**Business value:** a TA leader who wants to tighten the escalation threshold from 14 to 10
days gets a change confined to `ref_risk_band`. It may span more than one row there, since
adjusting a boundary means adjusting the band on each side of it, but it is one governed
seed file and one review. The band label, the bar chart, the At-Risk KPI and every
downstream model then move together, because there is only one place that says what "at
risk" means.

**Risk otherwise:** the boundaries drift. The bar chart says "High Risk 0–7" while the KPI
counts 0–10, and nobody notices for a quarter.

---

### 4.8 Forecast yield, fallback and the per-requisition cap

**What:** `mart_stage_yield` (`entered_stage_count`, `reached_active_fill_count`,
`reached_acceptance_event_count`, `observed_yield`, `applied_yield`,
`yield_segment_level`, `applied_observations`), then applying the yield per active
candidate, summing per requisition, and capping at `openings_position`.

**Why dbt:** people reach for Python here because "forecast" sounds like machine learning.
It is not. It is a grouped ratio with a deterministic fallback hierarchy — pure SQL, and
much safer in dbt for three reasons:

1. **Leakage prevention is testable.** The training population is applications with
   `has_final_outcome = true` and outcomes on or before the as-of date. As a dbt test, the
   spec's rule "historical yields must not use future outcomes" runs on every build.
2. **The fallback hierarchy is visible.** `bu_jf_jl → jf_jl → jf → all` becomes a
   coalesce over four aggregation levels, with `yield_segment_level` and
   `applied_observations` stored so a reviewer can see which segment supplied each
   probability. In a Python script this is a nested if-statement nobody audits.
3. **The cap must be upstream.** `LEAST(sum_of_candidate_yields, openings_position)` is
   applied per requisition and is non-additive. Power BI would need `SUMX` over
   requisitions with a candidate-level relationship that does not exist. dbt applies the
   cap once, at the right grain, and Power BI only sums the capped column.

**Business example:** a niche engineering requisition with 3 open seats and 40 active
candidates. Without the cap, the forecast says 7 seats will be filled on a requisition
that only has 3. Forecast Fill Rate exceeds 100% for that job family and the executive
loses trust in the whole chart.

**Naming note (fixed):** the wireframe footnote said "stage-to-acceptance yield", the
pre-v1.1 name. Contract v1.1 renamed it to **stage-to-active-fill yield** because the
training label is `is_active_fill`, not the raw acceptance event. `wireframe.html` now uses
the contract's word.

---

### 4.9 Hire outcome and the 60-day quality flags (`fct_hire_outcome`)

**What:** deduplicating HR worker and termination events to one row per started hire
(earliest termination after start), then `tenure_days_at_termination`, `days_observed`,
`is_60_day_matured`, `is_60_day_early_attrition`, the cohort flags joined from
`dim_start_cohort`, and `time_to_fill_days` carried onto the hire.

**Why dbt:** two things must be true at once and only dbt can guarantee both. First, the
population is *started hires only* — accepted offers pending a start, employer rescinds and
candidate reneges have no row here. Second, the speed line and the quality line on the
Speed vs Quality chart must describe **the same people**, which is why Time to Fill is
carried onto the hire row rather than looked up from the application at report time.

**Risk otherwise:** duplicate termination rows inflate the numerator, or a rescinded offer
is counted as an early exit. Both push the 60-Day Early Attrition rate up and make TA look
worse than it is on its primary quality metric.

---

### 4.10 The executive marts

`mart_exec_demand`, `mart_exec_risk`, `mart_exec_pipeline`, `mart_exec_quality` are
straightforward dbt aggregations of the facts. Three rules to keep:

- Store **additive** numerators and denominators, always. Power BI divides them.
- Store rates and medians as **row-grain reference values only**, hidden in the model, and
  test them against the fact. They exist to prove the mart and the fact agree, not to be
  aggregated.
- Reconcile every mart to its governed fact under total and per-segment filters, as each
  mart contract already specifies.

---

### 4.11 Configuration and reproducibility

**What:** `as_of_date`, `history_start_date`, `future_thd_end_date`, `fill_rate_target`,
`early_attrition_60d_target`, `attrition_window_days`, `quality_rolling_cohort_count`,
`min_cohort_size`, `forecast_min_segment_observations`.

Note what is **not** in this list: the At-Risk threshold. Risk classification is governed
entirely by the `ref_risk_band` seed (see 4.7). Configuration holds values that have no
other home; it does not hold a second copy of a rule a seed already defines.

**Recommended pattern:** put these in `dbt_project.yml` under `vars`, so macros and models
can read them at parse time, and build `ref_reporting_config` as a **one-row model** that
selects those vars. One source of truth, and Power BI still gets the disconnected config
table it expects.

Add a macro `{{ as_of_date() }}` and a CI check that fails if `current_date`, `getdate` or
`now()` appears anywhere in `models/`. That is what makes "running the pipeline in the
future must not change historical results" a guarantee instead of a promise.

---

### 4.12 Governed vocabulary as seeds

Move these to `seeds/`, and have models `ref()` them instead of hard-coding values:

| Seed | Governs | Used by |
|---|---|---|
| `dim_recruiting_stage` | Stage order, `next_stage_code`, `sla_days` | Stage conversion, days-in-stage SLA, pipeline mart, yield mart |
| `ref_risk_band` | Band boundaries, `is_at_risk`, sort order | Risk banding, At-Risk KPI, risk bars |
| `dim_hiring_constraint` | Constraint labels and display order | Constraint mix bars |

`spec.md` EXEC-11 already requires this: "the exact stage mapping must be governed in
configuration or YAML rather than embedded repeatedly in transformation code." A dbt seed
is that configuration, and it is the version that actually runs.

---

### 4.13 Validation rules as dbt tests

This is the recommendation with the best effort-to-value ratio. `spec.md` section 12 lists
roughly sixty business rules, and the schema contracts add more. Every one of them can be
made executable, though not all through generic tests:

| Rule type in the contracts | dbt test |
|---|---|
| `not_null`, `unique`, `accepted_values`, ranges | Built-in tests and `dbt_utils.accepted_range` |
| `{test: expression, ...}` | `dbt_utils.expression_is_true` |
| Reconciliation between two models | Singular tests in `tests/`, or `dbt_utils.equality` |
| Referential integrity | `relationships` |
| `severity: error` / `severity: warn` | dbt's own `severity` config |

Expect the mapping to be partial. Generic tests cover the column-level rules well, but a
large share of section 12 will need **custom singular tests** written as SQL in `tests/`.
The reconciliation rules (`filled_positions` equals the count of active-fill applications
per requisition, in total and per segment), the temporal rules (no actual event after the
as-of date, across five different date columns on four models), and the historical-immutability
rules (offer-stage conversion must be unchanged for a prior period after a post-acceptance
loss) all compare across models or across time. None of them is expressible as a generic
test. Budget for that work rather than assuming `dbt_utils` covers section 12.

Two suggestions on top:

- Set `store_failures: true` on reconciliation tests. When Fill Rate and the risk mart
  disagree, you want the offending requisitions in a table, not just a red line in the log.
- Use **model contracts** with `contract: {enforced: true}`. The `schemas/*.yaml` files
  already specify every column name, type and nullability. As dbt contracts, a model that
  drops or renames a column fails at build time rather than silently breaking a Power BI
  refresh.

---

## 5. What should stay in Power BI

Moving too much upstream is also a mistake. These belong in DAX and should not be
pre-computed as final values.

| Calculation | Why it stays |
|---|---|
| Fill Rate, Forecast Fill Rate, conversion rates, At-Risk share, 60-Day Early Attrition | Every one is a ratio of two sums under the current filter context. A stored rate cannot be re-aggregated: averaging monthly Fill Rates is not the Fill Rate. dbt stores the numerator and denominator; DAX divides them. |
| Median Time to Fill, Median Completed Days in Stage, Median Time to Fill (Start Cohort) | Medians never re-aggregate. dbt owns the row-level `time_to_fill_days` and `days_in_stage`; `MEDIAN()` runs over the fact so it honours every slicer combination. |
| Target comparison and status labels ("Below target") | Presentation. The target value comes from `ref_reporting_config`, never from a mart column. |
| The "Watch" call-out on the pipeline table | It highlights whichever stage is currently worst under the user's filters. That is a visual-grain decision. |
| Low-volume cohort flagging | `is_low_volume_cohort` is stored at row grain, but Power BI must re-evaluate it at visual grain using `SUM(matured_hires_60d)`, because a cohort can be large in total and small inside one business unit. |
| Formatting, sorting, conditional colours | Presentation. Keep numeric columns numeric in the marts. |

---

## 6. What should stay in Python

Generating the synthetic source data. That is simulating an ATS and an HR system, not
transforming it. It belongs in the implementation repository as a `uv`-managed project that
writes raw source files (CSV or Parquet) for dbt to read as sources — not in this
repository, which holds no generation code.

The dividing line: **Python may invent a record; it may not decide what a record means.**
Python creates an offer acceptance date. dbt decides whether that acceptance is still an
active fill.

One consequence for `spec.md`: section 13 said "Python project managed with `uv`" and
"modular transformations rather than one monolithic script" as the engineering requirement,
which would leave the spec and the implementation disagreeing. **This has been applied**:
section 13 now opens with a repository-boundary table and a layer-ownership table, stating
Python as synthetic-source generation, dbt as transformation and testing, and Power BI as
semantic aggregation and presentation — and marking the whole engineering section as
requirements on the separate implementation repository.

---

## 7. Required dbt project shape

This is the structure the **implementation repository** must build. None of these files
belongs in `ta-exec-db`.

```
dbt_project.yml            # vars: as_of_date, targets, thresholds
seeds/
  dim_recruiting_stage.csv
  dim_hiring_constraint.csv
  ref_risk_band.csv
models/
  reference/
    ref_reporting_config.sql
  staging/
    stg_ats__requisition_snapshot.sql
    stg_ats__application.sql
    stg_ats__offer_version.sql
    stg_ats__stage_history.sql
    stg_hr__worker_event.sql
  intermediate/
    int_requisition__resolved_snapshot.sql   # one row per requisition
    int_offer__resolved_acceptance.sql       # one governed acceptance per application
    int_application__events.sql              # event and state flags, Time to Fill
    int_stage_event__sequenced.sql           # windowing, conversion, days in stage
    int_requisition__pipeline_rollup.sql     # application counts back to requisition grain
  dimensions/
    dim_date.sql
    dim_start_cohort.sql
    dim_business_unit.sql
    dim_job_family.sql
    dim_job_level.sql
  facts/
    fct_application_stage_event.sql
    fct_application.sql
    fct_requisition.sql
    fct_hire_outcome.sql
  marts/
    mart_stage_yield.sql
    mart_exec_demand.sql
    mart_exec_risk.sql
    mart_exec_pipeline.sql
    mart_exec_quality.sql
macros/
  as_of_date.sql
  days_between.sql
  apply_risk_band.sql        # range join to the ref_risk_band seed
  yield_with_fallback.sql    # bu_jf_jl -> jf_jl -> jf -> all
tests/
  assert_requested_equals_filled_plus_open.sql
  assert_accepted_equals_filled_plus_losses.sql
  assert_risk_bands_reconcile_to_open_positions.sql
  assert_no_actual_event_after_as_of_date.sql
  assert_offer_conversion_survives_post_acceptance_loss.sql
  assert_forecast_not_greater_than_demand.sql
```

Each `schemas/*.yaml` file in this repository becomes the matching dbt `_models.yml` entry
over there: same columns, same descriptions, same tests, plus `contract: {enforced: true}`.
That is the mechanism by which these contracts stop being documentation about a pipeline
and start governing one — the contract is authored here and enforced at build time there.

**Warehouse suggestion:** dbt-duckdb fits this project well. It runs locally with no
infrastructure, reads the Python-generated CSV or Parquet sources directly, and can
materialise the marts as Parquet files for Power BI — which is exactly what `spec.md`
section 13 asks for.

---

## 8. Resolved finding: the build order was circular

`schemas/README.md` used to describe the build order as a numbered list, in which:

- Step 3 built `fct_requisition` "base columns", noting that "pipeline/forecast columns are
  filled in step 6"
- Step 6 applied the yield to `fct_application`, then computed
  `fct_requisition.active_pipeline_applications` and `expected_pipeline_fills`

That works in Python, where a script can add columns to a DataFrame later. It does **not**
work in dbt, where a model is one immutable `SELECT`. As written, `fct_requisition`
depended on `fct_application`, which depended on `mart_stage_yield`, which depended on
`fct_application`, which depended on `fct_requisition`.

**This has been fixed.** `schemas/README.md` now carries a conceptual *dependency flow*
instead of a numbered build order, splitting the two facts into an intermediate stage and a
final stage to give a clean acyclic graph:

```
int_requisition__resolved_snapshot     (source columns, status, quantities, TOAD)
        v
int_offer__resolved_acceptance         (one acceptance per application)
        v
int_application__events                (flags, Time to Fill - no yield yet)
        v
int_stage_event__sequenced             (windowing, conversion)
        v
mart_stage_yield                       (training, fallback)
        v
fct_application                        (events + stage_to_active_fill_yield)
        v
int_requisition__pipeline_rollup       (counts and uncapped expected fills)
        v
fct_requisition                        (roll-ups, risk band, capped forecast)
        v
fct_hire_outcome  ->  the four mart_exec_* models
```

This is worth doing regardless of dbt, but dbt makes it unavoidable — which is a benefit.
It is also why `schemas/README.md` now states the flow conceptually and leaves the ordering
to the implementation: dbt derives the order from `ref()` dependencies and keeps it correct
as the project grows, so no maintainer has to keep a numbered list accurate by hand.

---

## 9. Suggested sequence

You do not need to build all of this at once. This order gets governance value early, and
it follows the dependency graph in section 8 rather than cutting across it.

The sequence must respect one constraint: **the final `fct_application` and
`fct_requisition` models cannot be built before `mart_stage_yield`**, because
`fct_application.stage_to_active_fill_yield` comes from the yield mart and
`fct_requisition.expected_pipeline_fills` comes from the application fact. The intermediate
models come early; the two final facts close after the yield.

| Phase | Build | Why here |
|---|---|---|
| 1 | Seeds, `ref_reporting_config`, `dim_date`, `dim_start_cohort`, the `as_of_date` macro | Reproducibility and cohort maturity are the foundation everything else assumes |
| 2 | Staging models, `int_requisition__resolved_snapshot`, `int_offer__resolved_acceptance` with its source audit test | Fixes the grain before anything counts it |
| 3 | `int_application__events`, `int_stage_event__sequenced`, `fct_application_stage_event`, with the event/state and conversion tests | The vocabulary guarantees and stage conversion become executable. This phase builds the **intermediate application model** and the stage-event fact; the requisition side is still only the resolved snapshot from phase 2, and neither final fact is built yet |
| 4 | `mart_stage_yield` | Training needs a tested `is_active_fill` flag and sequenced stage events, and nothing else |
| 5 | `fct_application` with the applied yield, `int_requisition__pipeline_rollup`, then `fct_requisition` with the roll-ups, risk band and capped forecast | The point where the two final facts can close, because the yield now exists |
| 6 | `fct_hire_outcome`, `mart_exec_quality` | The quality KPI is the metric most exposed to a maturity mistake |
| 7 | `mart_exec_demand`, `mart_exec_risk`, `mart_exec_pipeline`, reconciliation tests, model contracts, exposures | Closes the loop between the marts and the page |

Adding a dbt **exposure** for the Executive Summary page in phase 7 is a small step with
good returns: lineage then shows which models feed which visual, and the wireframe coverage
table in `schemas/README.md` gains a machine-readable counterpart.

---

## 10. Optional: the dbt Semantic Layer

`metric-def.yaml` is close in structure to dbt semantic models and metrics — it already has
business questions, numerators, denominators, date bases, filters and validation rules.
Porting it to MetricFlow is technically possible and would give one metric definition for
every consumer.

The honest recommendation is **not yet**. Power BI consumes the marts directly today, and
the DAX guidance in the contracts is precise enough that duplication is manageable. Revisit
this if a second consumer appears — a notebook, an API, or an AI assistant answering
questions about hiring delivery. At that point one governed metric definition serving all
of them becomes worth the migration.
