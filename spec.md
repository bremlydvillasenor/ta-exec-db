# TA Executive Dashboard — Project Specification

## 1. Project overview

This project builds the analytics-ready data layer for a **Talent Acquisition Executive Summary** report in Power BI.

The project is intentionally limited to the **Executive Summary page only**. It should provide a small, governed set of dimensions, facts, marts, metric definitions, and validation rules needed to answer the most important questions a TA executive asks about hiring delivery, open-position risk, pipeline health, and expected future offer accepts.

The project is designed as a portfolio-quality analytics engineering example. The priority is not to reproduce every possible recruiting metric. The priority is to demonstrate a clear business problem, trustworthy metric logic, realistic data relationships, reproducible transformations, and Power BI-ready outputs.

### Audience

Primary users:

- VP / Head of Talent Acquisition
- TA leaders and recruiting operations leaders

Secondary users:

- Analytics engineers
- BI developers
- Reviewers evaluating the project as a portfolio artifact

---

## 2. Project objective

The Executive Summary should allow a TA leader to answer five questions quickly:

1. **Are we filling the hiring demand the business needs?**
2. **Are we hiring fast enough?**
3. **Which open positions are most likely to miss their target offer acceptance date (TOAD)?**
4. **Is the active recruiting pipeline strong enough to meet future demand?**
5. **Where in the recruiting process are the main constraints or conversion problems?**

The data layer must support these questions without requiring business logic to be recreated separately inside each Power BI visual.

---

## 3. Scope

### In scope

The project includes only data and business logic required for the **Executive Summary page**, including:

- hiring demand by Target Hire Date
- positions filled
- open positions
- Fill Rate
- median Time to Fill
- Target Offer Acceptance Date risk classification
- open positions at risk
- hiring constraints
- recruiting funnel volume
- stage-to-stage conversion
- time in recruiting stage where data is available
- active pipeline health
- forecasted Fill Rate using active pipeline yield
- Power BI-ready dimensional and fact datasets
- executive-summary marts where useful
- metric definitions
- schema and relationship definitions
- data quality and reconciliation tests

### Out of scope

The following are explicitly excluded from this phase:

- NPS
- recruiter scorecards
- recruiter productivity and capacity
- cost per hire
- source-of-hire analysis
- diversity analysis
- detailed Applications page
- detailed Hires page
- standalone Funnel Conversion page
- demand planning beyond what is required for the Executive Summary
- operational case-management workflows

These subjects may be added later as separate project phases.

---

## 4. Reporting period and reproducibility

### Historical data coverage

Actual recruiting activity should cover:

**January 1, 2024 through May 31, 2026.**

### Reporting as-of date

The fixed reporting as-of date is:

**May 31, 2026.**

Any logic using concepts such as `today`, `current`, `open`, `days remaining`, `latest`, or `at risk` must use this configured as-of date rather than the computer system date.

This makes the project reproducible. Running the pipeline in the future must not change historical results unless the source data itself changes.

### Future demand

Requisitions may contain **Target Hire Dates through May 31, 2027**.

Future Target Hire Dates are valid planning data and must be preserved. However, no actual recruiting event should occur after the reporting as-of date unless it is explicitly a planned or target date.

Examples of allowed future dates:

- Target Hire Date
- Target Offer Acceptance Date
- planned recruiting milestones

Examples of actual dates that must not be later than May 31, 2026:

- application date
- stage event date
- offer accepted date
- offer declined date
- candidate withdrawal date
- candidate start date

---

## 5. Core business concepts

### 5.1 Requisition and position

A **requisition** is the hiring request.

A **position** is one seat to be filled.

A requisition may contain multiple positions. Executive delivery metrics therefore use **positions** as the primary counting unit unless a metric explicitly states otherwise.

### 5.2 Target Hire Date (THD)

Target Hire Date is the date the business expects the person to start.

THD is the primary demand date for the Executive Summary. It determines which period a requisition's positions belong to for demand, Fill Rate, open-position reporting, and most executive views.

### 5.3 Target Offer Acceptance Date (TOAD)

Target Offer Acceptance Date is the date by which an offer should be accepted for the position to remain on track for the Target Hire Date.

**TOAD is already defined in the requisition data and must be used as provided. Do not recalculate it from THD.**

TOAD is the date used to classify open-position risk.

### 5.4 Filled position

A position is considered filled when a candidate has **accepted an offer on or before the reporting as-of date**.

Offer acceptance is used because it represents the end of the recruiting process controlled by Talent Acquisition.

### 5.5 Open position

Open positions are represented by `openings_position` in the requisition data.

For an open requisition:

```text
requested_positions = filled_positions + openings_position
```

Total Open Positions is therefore:

```text
SUM(openings_position)
```

for qualifying open requisitions.

`openings_position` is a requisition-level quantity. It must not be multiplied by the number of candidates or stage events joined to the requisition.

### 5.6 Hiring constraint

Hiring constraint is recorded at the **requisition level**.

Examples may include:

- insufficient qualified candidates
- compensation / offer competitiveness
- hiring manager delay
- assessment or interview capacity
- niche skill availability
- candidate availability / notice period
- no material constraint

Because the constraint belongs to the requisition, every open position on that requisition inherits the same current primary constraint for executive reporting.

---

## 6. Executive Summary metric contract

### EXEC-01 — Fill Rate

**Business question:** Of the positions the business expects to fill in the selected demand period, how many have been filled?

```text
Fill Rate = Positions Filled / Requested Positions
```

**Date basis:** Target Hire Date.

**Denominator:** Sum of requested positions on non-cancelled requisitions with THD in the selected period.

**Numerator:** Filled positions associated with the same requisitions and demand period.

**Filled event:** accepted offer on or before the as-of date.

**Primary comparison:** configured Fill Rate target.

Fill Rate is a position-based demand attainment measure, not the percentage of requisitions closed.

---

### EXEC-02 — Positions Filled

**Business question:** How many required positions has TA successfully filled?

```text
Positions Filled = SUM(filled_positions)
```

**Date basis:** requisition Target Hire Date.

Offers that were extended but not accepted do not count as fills.

---

### EXEC-03 — Demand / Requested Positions

**Business question:** How many positions does the business expect TA to fill in the selected period?

```text
Demand = SUM(requested_positions)
```

**Date basis:** Target Hire Date.

Cancelled requisitions or cancelled demand must be excluded.

---

### EXEC-04 — Total Open Positions

**Business question:** How many positions remain unfilled?

```text
Total Open Positions = SUM(openings_position)
```

for open, non-cancelled requisitions within the selected THD demand period.

Expected reconciliation:

```text
Requested Positions = Filled Positions + Open Positions
```

This reconciliation should hold at total level and under supported business filters.

---

### EXEC-05 — Median Time to Fill

**Business question:** How long does it typically take TA to secure an accepted offer?

```text
Time to Fill = Offer Accepted Date - Requisition Approval Date
```

The Executive Summary reports the **median**, not the average, because recruiting cycle times often contain long-tail outliers.

**Population:** accepted offers only.

**Date basis for Executive Summary attribution:** Target Hire Date of the associated requisition.

The calculation must not use future accepted offers after the reporting as-of date.

---

### EXEC-06 — Days to TOAD

For every open requisition:

```text
Days to TOAD = Target Offer Acceptance Date - As-of Date
```

The project uses the fixed as-of date of May 31, 2026.

Risk is based only on TOAD and current open positions. Pipeline strength does not change the risk classification.

---

### EXEC-07 — Open Position Risk Band

Every open requisition is classified from `days_to_toad`.

| Risk band | Rule |
|---|---|
| **Missed** | `days_to_toad < 0` |
| **High Risk** | `0 <= days_to_toad <= 7` |
| **Medium Risk** | `8 <= days_to_toad <= 14` |
| **On Track** | `days_to_toad >= 15` |

The number reported in each risk band is the **sum of `openings_position`**, not the number of requisitions.

Example:

A requisition has 8 open positions and `days_to_toad = 5`.

Result:

```text
High-Risk Requisitions = 1
High-Risk Open Positions = 8
```

The Executive Summary KPI uses **open positions**, while requisition count may be retained as supporting context where useful.

---

### EXEC-08 — At-Risk Open Positions

**Business question:** How many currently open positions require immediate management attention because their TOAD is near or already missed?

```text
At-Risk Open Positions =
    Missed Open Positions
  + High-Risk Open Positions
  + Medium-Risk Open Positions
```

Equivalent condition:

```text
days_to_toad <= 14
```

for open positions on qualifying requisitions.

The page should still show Missed, High Risk, and Medium Risk separately so the executive can distinguish overdue demand from approaching risk.

---

### EXEC-09 — Hiring Constraint Mix

**Business question:** What is preventing the current open demand from being filled?

The measure distributes open positions by the current primary hiring constraint recorded on each requisition.

```text
Open Positions by Constraint = SUM(openings_position)
```

The Executive Summary should use open positions as the weight so that a constraint affecting a 20-position requisition has more executive impact than a constraint affecting a single-position requisition.

---

### EXEC-10 — Funnel Volume by Stage

**Business question:** Where are active candidates currently concentrated in the recruiting process?

The measure counts active candidate applications by current recruiting stage for open requisitions.

Only candidates whose current status is active should contribute to the active-pipeline view.

Active candidates must not remain attached to closed or cancelled requisitions.

---

### EXEC-11 — Stage-to-Stage Conversion

**Business question:** At which recruiting stages are candidates most likely to fall out?

For completed historical stage movements:

```text
Stage Conversion = Candidates progressing to next stage / Candidates completing current stage
```

The exact stage mapping must be governed in configuration or YAML rather than embedded repeatedly in transformation code.

Candidate records still actively in a stage must not be incorrectly treated as failures.

---

### EXEC-12 — Median Days in Stage

**Business question:** Where is the recruiting process slowing down?

```text
Days in Stage = Stage Exit Date - Stage Entry Date
```

The Executive Summary uses median days in stage for completed stage intervals.

For active candidates still in their current stage, current age may be calculated against the as-of date for operational pipeline-health analysis, but it must be clearly distinguished from completed historical duration.

---

## 7. Forecast Fill Rate

### 7.1 Business purpose

The forecast answers:

**Based on the active candidate pipeline as of May 31, 2026, how much of future hiring demand is likely to be filled if historical conversion behavior continues?**

The forecast is a planning indicator, not a guarantee.

### 7.2 Forecast segmentation

Historical pipeline yield should be calculated as specifically as the data supports using:

- Business Unit
- Job Family
- Job Level
- Recruiting Stage

The preferred yield grain is:

```text
Business Unit + Job Family + Job Level + Current Stage
```

If a segment does not have enough historical observations to produce a stable rate, the model should use a documented fallback hierarchy to a broader segment rather than return an unstable or misleading conversion probability.

### 7.3 Stage-to-acceptance yield

For each active pipeline candidate, estimate the probability of eventually reaching accepted offer from the candidate's current stage using historical candidates from the appropriate segment.

```text
Expected Pipeline Fills = SUM(active candidate stage-to-acceptance probability)
```

Expected fills must then be capped at the number of remaining open positions on the requisition.

A requisition with 3 open positions cannot contribute more than 3 forecast fills regardless of how many candidates are in its pipeline.

### 7.4 Forecast filled positions

```text
Forecast Filled Positions =
    Actual Filled Positions
  + Expected Pipeline Fills
```

### 7.5 Forecast Fill Rate

```text
Forecast Fill Rate = Forecast Filled Positions / Demand
```

The forecast should support grouping by THD month so Power BI can show actual Fill Rate and projected Fill Rate on the same executive trend.

### 7.6 Leakage prevention

Historical yield calculations must use only recruiting outcomes known on or before May 31, 2026.

Future outcomes must never be used to calculate a candidate's historical conversion probability.

---

## 8. Executive Summary Power BI contract

The data model must support the following page components.

### KPI cards

1. **Fill Rate**
2. **Median Time to Fill**
3. **At-Risk Open Positions**

Supporting context such as requested positions, filled positions, or total open positions may be included as secondary labels or tooltips rather than additional primary KPI cards.

### Core visuals

The Executive Summary should be able to support:

- actual Fill Rate versus forecast Fill Rate over the THD timeline
- open positions by risk band
- open positions by primary hiring constraint
- recruiting funnel / stage volume with conversion context
- stage conversion and/or stage-time bottleneck indicators
- offer or pipeline outcome context where retained in the final wireframe

The purpose of every visual should be to explain one of the executive questions in Section 2. Avoid visuals that provide detail without supporting an executive decision.

### Page filters

Required slicers / filters:

- Target Hire Date range
- Business Unit
- Job Family
- Job Level, if retained in the wireframe

THD is the primary date filter for demand-oriented Executive Summary visuals.

The as-of date is a project configuration value and is not a user slicer.

---

## 9. Data model requirements

The implementation may use normalized facts plus executive marts, but the business logic must remain consistent regardless of physical design.

### 9.1 Required dimensions

At minimum:

- `dim_date`
- `dim_business_unit`
- `dim_job_family`
- `dim_job_level`
- `dim_recruiting_stage`

Additional dimensions may be added only where they improve model clarity or avoid repeated attributes.

### 9.2 Requisition fact

Suggested table: `fct_requisition`

**Grain:** one row per requisition.

Minimum fields required for Executive Summary logic:

- requisition_key
- requisition_id
- requisition_status
- approval_date
- target_hire_date
- target_offer_acceptance_date
- requested_positions
- filled_positions
- openings_position
- business_unit_key
- job_family_key
- job_level_key
- primary_hiring_constraint

If the source provides multiple requisition snapshots, transformation logic must explicitly resolve the snapshot needed for the as-of reporting state rather than accidentally summing multiple versions of the same requisition.

### 9.3 Application / candidate pipeline fact

The project must support both current active pipeline and historical stage conversion.

Preferred design:

- a current application-level fact for active pipeline state
- a stage-event fact for historical movement and stage duration

Suggested tables:

- `fct_application`
- `fct_application_stage_event`

Important keys and attributes include:

- application_key
- candidate_key
- requisition_key
- recruiting_stage_key
- current recruiting status
- application date
- stage entry date
- stage exit date
- disposition / withdrawal reason where available

### 9.4 Offer data

Offer information may be stored in a separate fact or integrated into the application fact, but the model must reliably identify:

- offer accepted
- offer declined
- offer rescinded where applicable
- candidate withdrawal / renege where applicable
- offer accepted date

Accepted offers are required for positions filled, Time to Fill, conversion outcomes, and forecast training labels.

---

## 10. Executive marts

The exact physical mart design may be refined during implementation. The following outputs are recommended because they keep Power BI logic simple.

### `mart_exec_demand`

Purpose: KPI and THD-based delivery analysis.

Suggested grain:

```text
THD month + Business Unit + Job Family + Job Level
```

Suggested measures:

- requested_positions
- filled_positions
- open_positions
- fill_rate
- median_time_to_fill

### `mart_exec_risk`

Purpose: open-position risk and hiring-constraint visuals.

Suggested grain:

```text
requisition
```

Suggested fields:

- target_hire_date
- target_offer_acceptance_date
- days_to_toad
- risk_band
- openings_position
- primary_hiring_constraint
- Business Unit
- Job Family
- Job Level

### `mart_exec_pipeline`

Purpose: active candidate pipeline health.

Suggested grain:

```text
Business Unit + Job Family + Job Level + Recruiting Stage
```

Suggested measures:

- active_candidates
- historical_stage_conversion
- stage_to_acceptance_yield
- median_completed_days_in_stage
- median_active_stage_age

### `mart_exec_forecast`

Purpose: actual versus projected Fill Rate.

Suggested grain:

```text
THD month + Business Unit + Job Family + Job Level
```

Suggested measures:

- demand
- actual_filled_positions
- expected_pipeline_fills
- forecast_filled_positions
- actual_fill_rate
- forecast_fill_rate

Power BI may calculate final presentation measures in DAX, but complex row-level business logic should be produced upstream where practical and documented clearly.

---

## 11. Data story requirements

The generated or curated portfolio data should tell a realistic and internally consistent business story rather than behave like unrelated random records.

The Executive Summary should allow a reviewer to observe relationships such as:

- differences in Fill Rate between business segments
- open-position risk concentrated in specific Job Families or Job Levels
- a visible connection between hiring constraints and risk exposure
- a funnel bottleneck that helps explain weaker delivery
- stronger or weaker active pipelines producing appropriately different forecast Fill Rates
- future demand extending beyond the as-of date without impossible future recruiting outcomes

The story must emerge from consistent underlying records. Dashboard values must not be independently hard-coded to produce a desired picture.

---

## 12. Data quality and validation rules

The pipeline must fail or clearly flag records that violate core business rules.

### Requisition rules

1. `requested_positions >= 0`
2. `filled_positions >= 0`
3. `openings_position >= 0`
4. `filled_positions <= requested_positions`
5. `openings_position <= requested_positions`
6. For active demand where the fields represent current state:

```text
requested_positions = filled_positions + openings_position
```

7. Cancelled requisitions must not contribute to active demand or open-position KPIs.
8. TOAD must be sourced from the requisition field and not silently recomputed.

### Date rules

1. Actual recruiting events must not occur after the as-of date.
2. Target dates may occur after the as-of date.
3. Offer accepted date must not precede application chronology where source logic makes that impossible.
4. Stage exit date must not precede stage entry date.
5. Time to Fill must not be negative.

### Pipeline rules

1. Active candidates must belong to open, valid requisitions.
2. A candidate application should have one current stage at the reporting as-of date.
3. Historical stage events must not create duplicate candidate-stage transitions unless the recruiting process genuinely allows a return to a prior stage.
4. Candidates still in process must not automatically be treated as failed conversions.

### Risk rules

1. Risk classification applies only to open requisitions / open positions.
2. Risk counts must use `openings_position`.
3. Risk band totals must reconcile to Total Open Positions for the same filter context.
4. Requisition-level joins must not duplicate `openings_position` when candidate-level data is joined.

### Forecast rules

1. Historical yields must not use future outcomes.
2. Stage yields must remain between 0 and 1.
3. Expected pipeline fills must not exceed remaining open positions per requisition.
4. Forecast filled positions must not exceed demand.
5. Forecast logic and fallback hierarchy must be documented and deterministic.

---

## 13. Required project outputs

The completed analytics project should provide:

### Data outputs

- dimension datasets
- requisition fact
- application / pipeline fact
- application stage-event fact
- offer fields or offer fact as required
- Executive Summary marts
- CSV or Parquet outputs suitable for Power BI

### Documentation

- `README.md` — project purpose, setup, run instructions, and output summary
- `architecture.md` — pipeline layers, table relationships, grain, and data flow
- metric definition YAML or Markdown — governed Executive Summary metrics
- dataset schema YAML — columns, data types, keys, relationships, and descriptions

### Engineering requirements

- Python project managed with `uv`
- deterministic configuration for the as-of date
- modular transformations rather than one monolithic script
- clear source / intermediate / fact / mart separation
- logging
- validation tests
- repeatable execution

The implementation should favor simplicity and readability over unnecessary framework complexity.

---

## 14. Power BI readiness

The final outputs must be designed so that Power BI can use a straightforward star-schema model.

Requirements:

- dimensions have unique keys
- facts have documented grain
- many-to-many relationships should be avoided unless there is a clear business reason
- position quantities must not be duplicated through candidate-level joins
- THD should connect cleanly to the date dimension for demand views
- metric logic must produce the same result whether calculated from the governed fact tables or validated against the executive marts
- numeric measures should remain numeric; presentation formatting belongs in Power BI

Where a calculation is highly reusable and business-critical, prefer creating a governed field or mart measure upstream rather than embedding equivalent logic in several visuals.

---

## 15. Acceptance criteria

The Executive Summary data project is complete when all of the following are true:

1. The project covers only the Executive Summary scope defined here.
2. Data is reproducible using an as-of date of May 31, 2026.
3. Historical actual data covers January 2024 through May 31, 2026.
4. Requisition THDs through May 31, 2027 are preserved for future-demand analysis.
5. Fill Rate reconciles from requested, filled, and open position quantities.
6. Median Time to Fill uses approval-to-offer-acceptance duration.
7. Open-position risk is based on source TOAD and the configured as-of date.
8. High Risk is 0–7 days to TOAD; Medium Risk is 8–14 days; Missed is below 0.
9. At-Risk Open Positions counts open seats, not merely requisitions.
10. Hiring constraints are treated as requisition-level attributes.
11. Funnel metrics distinguish active pipeline from completed historical conversion.
12. Forecast Fill Rate uses active pipeline stage-to-acceptance yield segmented by Business Unit, Job Family, and Job Level where data supports it.
13. Forecast expected fills are capped by remaining open positions.
14. No future actual recruiting outcomes leak into historical metrics or forecast training data.
15. Required dimensions, facts, marts, documentation, and schema definitions are produced.
16. Power BI can build the Executive Summary page without reconstructing core business logic from raw source files.
17. Early Attrition and all other non-Executive Summary report pages remain outside the project scope.

---

## 16. Design principle

The project should remain intentionally focused.

A strong Executive Summary does not contain every recruiting metric. It gives leadership a concise view of:

**demand → delivery → risk → pipeline → expected outcome.**

Every dataset, metric, and transformation included in this phase should support that story. If an element does not materially help explain one of those five areas, it should be excluded from the current project.
