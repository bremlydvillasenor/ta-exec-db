# TA Executive Dashboard — Project Specification

## 1. Project overview

This project builds the analytics-ready data layer for a **Talent Acquisition Executive Summary** report in Power BI.

The project is intentionally limited to the **Executive Summary page only**. It should provide a small, governed set of dimensions, facts, marts, metric definitions, and validation rules needed to answer the most important questions a TA executive asks about hiring delivery, speed, open-position risk, pipeline health, hiring quality, and expected future offer accepts.

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

The Executive Summary should allow a TA leader to answer six questions quickly:

1. **Are we filling the hiring demand the business needs?**
2. **Are we hiring fast enough?**
3. **Which open positions are most likely to miss their target offer acceptance date (TOAD)?**
4. **Is the active recruiting pipeline strong enough to meet future demand?**
5. **Where in the recruiting process are the main constraints or conversion problems?**
6. **Are we hiring quickly without creating poor early-tenure outcomes?**

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
- **60-Day Early Attrition as the primary hiring-quality metric**
- **speed-versus-quality trend using median Time to Fill and 60-Day Early Attrition**
- Power BI-ready dimensional and fact datasets
- executive-summary marts where useful
- metric definitions
- schema and relationship definitions
- data quality and reconciliation tests

### Out of scope

The following are explicitly excluded from this phase:

- a standalone Early Attrition page
- early attrition windows other than 60 days as primary Executive Summary metrics
- broader retention analysis
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

Any logic using concepts such as `today`, `current`, `open`, `days remaining`, `latest`, `at risk`, or `matured cohort` must use this configured as-of date rather than the computer system date.

This makes the project reproducible. Running the pipeline in the future must not change historical results unless the source data itself changes.

### Future demand

Requisitions may contain **Target Hire Dates through May 31, 2027**.

Future Target Hire Dates are valid planning data and must be preserved. However, no actual recruiting or employment event should occur after the reporting as-of date unless it is explicitly a planned or target date.

Examples of allowed future dates:

- Target Hire Date
- Target Offer Acceptance Date
- planned recruiting milestones

Examples of actual dates that must not be later than May 31, 2026:

- application date
- stage event date
- offer accepted date
- offer declined date
- offer rescinded date (post-acceptance employer rescind)
- candidate renege date (post-acceptance candidate withdrawal)
- candidate withdrawal date (pre-acceptance)
- candidate start date
- termination date

---

## 5. Core business concepts

### 5.1 Requisition and position

A **requisition** is the hiring request.

A **position** is one seat to be filled.

A requisition may contain multiple positions. Executive delivery metrics therefore use **positions** as the primary counting unit unless a metric explicitly states otherwise.

### 5.2 Target Hire Date (THD)

Target Hire Date is the date the business expects the person to start.

THD is the primary demand date for the Executive Summary. It determines which period a requisition's positions belong to for demand, Fill Rate, open-position reporting, and most delivery views.

### 5.3 Target Offer Acceptance Date (TOAD)

Target Offer Acceptance Date is the date by which an offer should be accepted for the position to remain on track for the Target Hire Date.

**TOAD is already defined in the requisition data and must be used as provided. Do not recalculate it from THD.**

TOAD is the date used to classify open-position risk.

### 5.4 Offer acceptance, filled position, and hire

These are three different things and must not be defined from one another.

**Offer acceptance is an immutable historical event. Current fill status is a separate current-state concept.**

**Offer Accepted** — a candidate accepted an offer on or before the reporting as-of date.
This is a historical Talent Acquisition fill event and it is permanent. If the employer
later rescinds the offer, or the candidate later reneges, the acceptance still happened:
`offer_accepted_date` is preserved and is never cleared or overwritten.

**Active Fill** — an accepted offer that has **not** subsequently been rescinded or
reneged. This is the current-state concept. A position is considered filled, for Fill Rate
and filled-position reporting, when it is held by an active fill. If an accepted offer is
lost after acceptance and the seat re-opens, the seat is restated as open.

**Hire** — a candidate who **actually started employment**. Narrower than an active fill:
an accepted offer waiting for its start date is a fill, not yet a hire. Only actual starts
are used for hiring-quality analysis.

Offer acceptance is used as the TA delivery event because it represents the end of the
recruiting process controlled by Talent Acquisition. Employee start is used for quality
because the observation window can only begin on a real first day of employment.

The model must therefore keep these separate, as distinct columns:

- current application status
- offer accepted event and offer accepted date
- employer withdrawal (pre-acceptance)
- employer rescind (post-acceptance)
- candidate decline (pre-acceptance)
- candidate renege / post-acceptance withdrawal
- actual employee start

**Reserved offer-loss vocabulary.** An offer can be lost by either side, before or after
acceptance. These are four different events and each has one reserved word:

| | Before acceptance | After acceptance |
|---|---|---|
| Employer ends it | `offer_withdrawn` | `offer_rescinded` |
| Candidate ends it | `offer_declined` | `candidate_renege` |

Before acceptance, no acceptance event ever existed: the offer accepted date is null, the
seat was never filled, and nothing affects Fill Rate, Time to Fill or offer-stage
conversion. After acceptance, the acceptance event stands and is preserved: the seat was
filled and is restated as open, so Fill Rate falls while the historical measures are
unchanged. Only the after-acceptance pair are post-acceptance losses.

A candidate leaving with no offer on the table is `withdrawn`; one screened out by the
employer is `rejected`. Neither is an offer-loss term. `rescind` must never be used for a
pre-acceptance withdrawal.

No metric may be defined from an application status value. Statuses change; dated events do
not. In particular, `is_offer_accepted` must not be derived from `application_status = hired`.

**Scope limit — one acceptance per application.** One application contributes at most one
governed accepted-offer event. One accepted offer equals one seat, and every offer-based
figure in this specification is a count of applications, which is what keeps accepted offer
events, filled positions, and started positions reconcilable at requisition grain.

Multiple offer versions before final acceptance — a revised salary, a moved start date, a
re-issued offer letter — are offer *versions*, not separate acceptance events. The
resolution rule is:

1. **Collapse administrative revisions of the same accepted offer.** A corrected salary, a
   moved start date or a re-issued letter for the offer the candidate accepted are versions
   of one event and must not produce a second acceptance.
2. **Preserve the earliest valid acceptance event for the accepted offer cycle.** The
   governed acceptance date is the moment the candidate committed, not the date of the last
   piece of paperwork.
3. **Quarantine ambiguous multiple-acceptance cases for review.** An application carrying
   more than one distinct acceptance cycle that cannot be resolved as revisions of a single
   offer is held for review — never silently collapsed, and never silently dropped.
4. **Audit the source, not only the output — and do not fail on legitimate revisions.**
   Multiple accepted offer versions on one source application are *expected*: rule 1 exists
   because they occur. Their presence must never fail the build on its own. The
   implementation must materialise an **audit model** recording every source application
   that arrived with more than one accepted offer version, together with how it was
   resolved — `administrative_revision` or `quarantined` — and why. A uniqueness test on the
   resolved output only proves that the resolution ran; the audit model is what shows
   whether a real second acceptance was discarded.
5. **Fail hard only on the two cases that mean something is wrong.** The build must fail
   when a multi-version application is neither resolved as administrative revisions nor
   quarantined — nobody can then say what happened to it — and when an application marked
   quarantined reaches the final application fact. Quarantined applications are reviewed by
   a person before anything counts them.
6. **Use a separate offer-event fact if genuine re-offer cycles are supported later.**

The resolution rule must be documented where it is applied.

A genuine re-offer cycle, where the same candidate accepts, is lost to a rescind or renege,
and is later re-offered and accepts again for the same requisition, is out of scope for the
current design; the source is expected to produce a new application for the second attempt.
If multiple acceptance or re-offer cycles per application become a requirement, a separate
offer-event fact must be introduced rather than adding further offer columns to the
application fact.

### 5.5 Open position

Open positions are represented by `openings_position` in the requisition data.

For an open requisition:

```text
requested_positions = filled_positions + openings_position
```

where `filled_positions` counts **active fills**. When an accepted offer is rescinded or
reneged and the seat re-opens, the seat moves from `filled_positions` to
`openings_position`; `requested_positions` is unchanged and the identity still holds. If
the business decides not to re-open the seat, it moves to `cancelled_positions` instead.
A requisition returning from `filled` to `open` after a post-acceptance loss is a valid
transition, not a data error.

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

### 5.7 Hire and start cohort

A **hire** for the quality metric is a person who has actually started employment on or before the reporting as-of date.

A **start cohort** groups hires by their employee start month. Start month, not THD or offer acceptance month, is the date basis for 60-Day Early Attrition because the 60-day observation period begins only when employment starts.

### 5.8 Fully matured 60-day cohort

A hire is 60-day matured when at least 60 calendar days have elapsed between the employee start date and the configured as-of date.

For monthly reporting, a start month is considered **fully matured** only when every possible start date in that month has had the full 60-day observation window by the as-of date. This avoids presenting a partially observed month as if its attrition rate were complete.

For the May 31, 2026 as-of date, the latest fully matured calendar start cohort is determined programmatically from this rule rather than hard-coded into visuals.

### 5.9 Quality metric date behavior

Demand-oriented visuals use THD as their reporting date. The 60-Day Early Attrition KPI and speed-versus-quality trend use **employee start cohort month** instead.

The THD slicer must therefore **not** shorten or distort the 60-day quality observation window. Business Unit, Job Family, and Job Level filters may apply to both delivery and quality metrics where conformed keys are available.

---

## 6. Executive Summary metric contract

### EXEC-01 — Fill Rate

**Business question:** Of the positions the business expects to fill in the selected demand period, how many are filled right now?

```text
Fill Rate = Positions Filled / Requested Positions
```

**Date basis:** Target Hire Date.

**Denominator:** Sum of requested positions on non-cancelled requisitions with THD in the selected period.

**Numerator:** Filled positions associated with the same requisitions and demand period.

**Filled event:** an **active fill** — an offer accepted on or before the as-of date that has
not subsequently been rescinded or reneged.

**Primary comparison:** configured Fill Rate target.

Fill Rate is a position-based demand attainment measure, not the percentage of requisitions
closed, and not a count of historical accepted offers.

Fill Rate reports **current** state. If an accepted offer is later rescinded or reneged and
the seat re-opens, Fill Rate falls. The historical accepted-offer count is reported
separately and never falls, so the drop is explainable rather than mysterious. Show
post-acceptance losses alongside Fill Rate whenever it moves for this reason.

---

### EXEC-02 — Positions Filled (Active Fills)

**Business question:** How many required positions are filled right now?

```text
Positions Filled = SUM(filled_positions)      -- active fills
```

**Date basis:** requisition Target Hire Date.

Offers that were extended but not accepted do not count as fills. Accepted offers that were
later rescinded by the employer, or reneged by the candidate, are no longer active fills and
do not count here either — their seats are restated as open.

Two supporting counts must be available alongside this metric:

- **Accepted Offer Events** — every accepted offer on or before the as-of date, including
  those later lost. Historical, never decreases.
- **Post-Acceptance Losses** — accepted offers lost to employer rescind or candidate renege,
  reported separately for the two causes.

```text
Accepted Offer Events = Positions Filled + Post-Acceptance Losses
```

An accepted offer that has not started yet is still a fill. It becomes a **hire** only when
the person starts; started hires are counted separately and are the only delivery figure
that may be labelled "hires".

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

**Population:** every accepted-offer event, on or before the as-of date.

The original accepted offer **stays in the population even if the offer was later rescinded
or the candidate reneged**. Time to Fill measures the recruiting cycle Talent Acquisition
actually completed; removing it after the fact would rewrite history and bias the cycle
time. Offers the candidate declined (`offer_declined`), and offers the employer withdrew
before acceptance (`offer_withdrawn`), are excluded — there was no acceptance event.

**Default date basis for delivery reporting:** Target Hire Date of the associated requisition.

For the speed-versus-quality visual defined in EXEC-14, Time to Fill is re-cohorted by employee start month so it describes the same hires used in the attrition line.

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

For the Offer stage, the successful exit is the **offer-acceptance event**. An accepted
offer remains a successful conversion even if the offer was later rescinded, the candidate
reneged, or the person never started. Historical conversion describes what the recruiting
process achieved at that point in time; current fill status is a separate concept reported
by EXEC-01.

Offer-stage conversion will therefore normally sit above the current fill picture. The
difference is exactly the post-acceptance losses, and it is reported, not hidden.

The exact stage mapping must be governed in configuration or YAML rather than embedded repeatedly in transformation code.

Candidate records still actively in a stage must not be incorrectly treated as failures.

Conversion outcomes must not be recalculated from current application status. They are
derived from dated stage events and the offer-acceptance event.

---

### EXEC-12 — Median Days in Stage

**Business question:** Where is the recruiting process slowing down?

```text
Days in Stage = Stage Exit Date - Stage Entry Date
```

The Executive Summary uses median days in stage for completed stage intervals.

For active candidates still in their current stage, current age may be calculated against the as-of date for operational pipeline-health analysis, but it must be clearly distinguished from completed historical duration.

---

### EXEC-13 — 60-Day Early Attrition

**Business question:** Of the hires with a complete 60-day observation window, what percentage left the organization within their first 60 days?

The Executive Summary uses 60-Day Early Attrition as its primary **hiring-quality outcome metric**.

```text
60-Day Early Attrition Rate =
    Hires who terminated within 60 days of start
    /
    Hires in fully matured 60-day start cohorts
```

A termination qualifies when:

```text
0 <= Termination Date - Employee Start Date <= 60 days
```

**Date basis:** Employee start month.

**KPI period:** rolling 12 months of fully matured start cohorts, ending with the latest fully matured start month available as of the configured reporting date.

**Denominator:** hires in those fully matured start cohorts.

**Numerator:** those same hires who terminated within 60 days of their employee start date.

Hires who have not yet completed the full 60-day observation window must not enter the denominator.

This project intentionally treats early attrition as a **cohort quality measure** for TA. It should not be mixed with a broader enterprise turnover metric that may use average headcount as its denominator.

The KPI should support comparison with a configured quality target and, where useful, the immediately preceding matured 12-month period.

---

### EXEC-14 — Speed vs 60-Day Early Attrition

**Business question:** When hiring becomes faster or slower, does early-tenure quality appear to move with it?

This visual compares hiring speed and 60-day quality over time using the **same monthly start cohorts**.

For each fully matured employee start month:

```text
Speed = Median Time to Fill for hires in the start cohort
Quality = 60-Day Early Attrition Rate for the same start cohort
```

**Visual:** dual-axis line chart.

- X-axis: employee start cohort month
- Left Y-axis: median Time to Fill in days
- Right Y-axis: 60-Day Early Attrition rate (%)
- Population: fully matured 60-day start cohorts only
- Recommended window: latest 12 fully matured monthly cohorts

The Time to Fill line must use the hires represented in the same start cohort rather than the normal THD-based Executive Summary attribution. This keeps the comparison aligned.

The visual is diagnostic, not causal. A relationship between faster hiring and higher or lower attrition may justify investigation, but the chart must not state that hiring speed caused the attrition outcome.

Where the sample size for a monthly cohort is too small for a stable attrition rate, the model should expose cohort hire count so Power BI can flag or suppress low-volume points using a configured minimum cohort-size rule.

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

### 7.3 Stage-to-active-fill yield

For each active pipeline candidate, estimate the probability of eventually reaching an
**active fill** — an accepted offer that is not subsequently rescinded or reneged — from the
candidate's current stage, using historical candidates from the appropriate segment.

The training label is the active fill rather than the raw acceptance event, so that forecast
filled positions stay on the same definition as actual filled positions and Forecast Fill
Rate stays comparable with Fill Rate. An offer accepted and then lost trains as a failure
here, even though it remains a successful historical offer-stage conversion in EXEC-11.

```text
Expected Pipeline Fills = SUM(active candidate stage-to-active-fill probability)
```

Expected fills must then be capped at the number of remaining open positions on the requisition.

A requisition with 3 open positions cannot contribute more than 3 forecast fills regardless of how many candidates are in its pipeline.

### 7.4 Forecast filled positions

```text
Forecast Filled Positions =
    Actual Filled Positions        -- active fills
  + Expected Pipeline Fills        -- expected active fills
```

Both terms use the active-fill definition. A seat re-opened by a rescind or a renege returns
to open positions, so the pipeline is given a fresh chance to fill it rather than the seat
being counted as filled twice.

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

1. **Fill Rate** — delivery
2. **Median Time to Fill** — speed
3. **At-Risk Open Positions** — execution risk
4. **60-Day Early Attrition** — quality

Supporting context such as requested positions, filled positions, total open positions, matured hire count, or comparison to target may be included as secondary labels or tooltips rather than additional primary KPI cards.

### Core visuals

The Executive Summary should be able to support:

- actual Fill Rate versus forecast Fill Rate over the THD timeline
- open positions by risk band
- open positions by primary hiring constraint
- recruiting funnel / stage volume with conversion context
- stage conversion and/or stage-time bottleneck indicators
- **Speed vs 60-Day Early Attrition dual-axis line chart using fully matured monthly start cohorts**
- offer or pipeline outcome context where retained in the final wireframe

The purpose of every visual should be to explain one of the executive questions in Section 2. Avoid visuals that provide detail without supporting an executive decision.

### Page filters

Required slicers / filters:

- Target Hire Date range
- Business Unit
- Job Family
- Job Level, if retained in the wireframe

THD is the primary date filter for demand-oriented Executive Summary visuals.

The 60-Day Early Attrition KPI and speed-versus-quality chart are **cohort-maturity visuals** and should not be truncated by the THD slicer. Their time window is controlled by the latest fully matured start cohort relative to the configured as-of date.

Business Unit, Job Family, and Job Level filters should apply to the quality visuals where valid conformed attributes are available for the hire.

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
- filled_positions (active fills: accepted and not subsequently rescinded or reneged)
- accepted_offer_events (historical accepted offers, including those later lost)
- lost_after_acceptance_positions (employer rescind + candidate renege)
- started_positions (seats where the person actually started)
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
- application_status_current (current state; never used to define the acceptance event)
- application date
- stage entry date
- stage exit date
- disposition / withdrawal reason where available

The application fact must keep current state and historical events in separate columns. The
status value `hired` must not be used, because it conflates the Talent Acquisition fill
event with the actual employee start; use `offer_accepted` and `started` instead.

The application fact assumes at most one governed accepted-offer event per application; see
section 9.4.

### 9.4 Offer data

Offer information may be stored in a separate fact or integrated into the application fact,
but the model must reliably and **separately** identify:

- `is_offer_accepted_event` — the immutable historical acceptance event
- `offer_accepted_date` — preserved even after a rescind or renege
- `offer_withdrawn_date` — employer withdrawal **before** acceptance (no acceptance event)
- `is_offer_rescinded` / `offer_rescinded_date` — employer rescind **after** acceptance
- `is_candidate_renege` / `candidate_renege_date` — candidate withdrawal **after** acceptance
- `is_started` / `employee_start_date` — the actual employment start
- `post_acceptance_outcome` — one of `pending_start`, `started`, `candidate_renege`, `employer_rescind`
- `offer_declined_date` — candidate decline **before** acceptance (no acceptance event)

The four offer-loss terms are reserved as defined in section 5.4 and must not be
interchanged: `offer_withdrawn` and `offer_declined` are pre-acceptance and never affect a
fill or delivery metric; `offer_rescinded` and `candidate_renege` are post-acceptance and
reduce current fill while leaving history intact.

`is_offer_accepted_event` must be derived from `offer_accepted_date`, never from a status
value. A post-acceptance rescind or renege must not clear `offer_accepted_date`, must not
clear `time_to_fill_days`, and must not change any historical conversion outcome.

An **active fill** is then an accepted offer with no subsequent rescind or renege. Accepted
offer events are required for Time to Fill and offer-stage conversion; active fills are
required for positions filled, Fill Rate and forecast training labels; actual starts are
required for hiring-quality analysis.

**Cardinality assumption.** The columns above describe a single acceptance event, so one
application must contribute at most one governed accepted-offer event. Where the source
holds several offer versions for one application before final acceptance, the transformation
must collapse administrative revisions of the accepted offer and preserve the earliest valid
acceptance event of that cycle, as set out in section 5.4. An application carrying more than
one distinct acceptance cycle that cannot be resolved this way must be quarantined for
review, never loaded as-is and never silently collapsed. Multiple accepted offer versions in
the source are expected and must not fail the build by themselves; instead, an audit model
must record every application that arrived with more than one accepted version and how it
was resolved. The hard failures are narrower: a multi-version application left neither
resolved nor quarantined, or a quarantined application reaching the application fact. The
resolution rule must be documented where it is applied.

If future requirements need multiple acceptance or re-offer cycles for a single application,
introduce a **separate offer-event fact** — one row per offer event, with an offer sequence
number and its own accepted, rescinded and reneged dates — and keep the application fact at
one row per application holding the resolved current state. Do not overload the application
fact with a second set of offer columns or a repeated group; that would break the application
grain and every count-based reconciliation in section 12.

### 9.5 Hire and termination data

The project must include the minimum employee outcome data required to calculate 60-Day Early Attrition.

Suggested table: `fct_hire_outcome`

**Preferred grain:** one row per started hire — a person who **actually started employment**.

Accepted offers that never became a start (still pending, rescinded by the employer, or
reneged by the candidate) have no row here. They remain fully visible in the application and
requisition facts, so a post-acceptance loss is out of scope for quality analysis without
being deleted from the data.

Minimum fields:

- hire_key
- candidate_key and/or worker_key
- requisition_key
- employee_start_date
- termination_date, nullable
- tenure_days_at_termination, nullable
- is_60_day_matured
- is_60_day_early_attrition
- business_unit_key
- job_family_key
- job_level_key
- offer_accepted_date
- requisition_approval_date or a reliable link back to it
- time_to_fill_days

A hire should appear once at this grain. Multiple termination or worker-event source rows must be resolved before the executive quality mart is produced.

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
- filled_positions (active fills)
- accepted_offer_events
- lost_after_acceptance_positions
- started_positions
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
- stage_to_active_fill_yield
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

### `mart_exec_quality`

Purpose: Executive Summary hiring-quality KPI and speed-versus-quality trend.

Suggested grain:

```text
Employee Start Month + Business Unit + Job Family + Job Level
```

Suggested measures and fields:

- matured_hires_60d
- early_attrition_60d_count
- early_attrition_60d_rate
- median_time_to_fill_same_cohort
- is_fully_matured_start_month
- cohort_start_month
- cohort_sample_size

Only fully matured monthly cohorts should feed the Executive Summary trend. The KPI should aggregate the latest rolling 12 fully matured cohorts.

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
- meaningful variation in 60-Day Early Attrition between matured hire cohorts or business segments
- periods where faster or slower Time to Fill can be compared with early attrition without forcing a perfect relationship
- future demand extending beyond the as-of date without impossible future recruiting outcomes

The speed-versus-quality data story should be realistic. The generated data must not artificially force a strong correlation between Time to Fill and Early Attrition merely to make the chart look interesting.

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
9. `accepted_offer_events = filled_positions + lost_after_acceptance_positions`.
10. `started_positions <= filled_positions <= accepted_offer_events`.
11. `accepted_offer_events` must never decrease for a period that is already in the past.
12. A requisition moving from `filled` back to `open` after a post-acceptance loss is a valid transition and must not be flagged as an error.

### Date rules

1. Actual recruiting and employee events must not occur after the as-of date.
2. Target dates may occur after the as-of date.
3. Offer accepted date must not precede application chronology where source logic makes that impossible.
4. Stage exit date must not precede stage entry date.
5. Time to Fill must not be negative.
6. Termination date must not precede employee start date.
7. `offer_rescinded_date` and `candidate_renege_date` must not precede `offer_accepted_date`.
8. `employee_start_date` must not precede `offer_accepted_date`.

### Offer event rules

1. `offer_accepted_date` is immutable. It **must be preserved** after an employer rescind or a candidate renege. Any rule that requires it to become null after a post-acceptance loss is a defect and must be removed.
2. `is_offer_accepted_event` must be derived from `offer_accepted_date`, never from `application_status_current` and never from the removed status value `hired`.
3. `time_to_fill_days` must be preserved for accepted offers later rescinded or reneged.
4. `is_offer_rescinded` and `is_candidate_renege` are mutually exclusive on one application, and both imply `is_offer_accepted_event = true`.
5. A loss that happened **before** acceptance is not a post-acceptance event and leaves `post_acceptance_outcome` null. Use `offer_withdrawn` (employer) or `offer_declined` (candidate); both imply no acceptance event and a null offer accepted date.
6. `offer_rescinded` is reserved for an employer rescind **after** acceptance and always implies an acceptance event. The value `offer_rescinded` must not appear as a stage exit reason, because a post-acceptance loss is not a stage exit.
7. `is_active_fill = is_offer_accepted_event AND NOT is_offer_rescinded AND NOT is_candidate_renege`.
8. `is_started` implies an accepted-offer event and no post-acceptance loss. A person who started and then left is a termination, not a renege.
9. Historical conversion outcomes must be reproducible: re-running the pipeline on an unchanged as-of date must return the same offer-stage conversion, even after post-acceptance losses have been loaded.
10. One application must carry at most one governed accepted-offer event. Administrative revisions of the accepted offer are collapsed and the earliest valid acceptance event of that cycle is preserved. An application with more than one distinct acceptance cycle is quarantined for review, not loaded as-is and not silently collapsed.
11. An audit model must record every source application arriving with more than one accepted offer version, with its resolution (`administrative_revision` or `quarantined`). Multiple accepted versions are expected and are not a failure. The uniqueness test on the resolved output proves only that the resolution ran, not that it was correct.
12. Two hard tests must fail the build: any multi-version application whose resolution is neither `administrative_revision` nor `quarantined`, and any quarantined application appearing in the application fact.

### Pipeline rules

1. Active candidates must belong to open, valid requisitions.
2. A candidate application should have one current stage at the reporting as-of date.
3. Historical stage events must not create duplicate candidate-stage transitions unless the recruiting process genuinely allows a return to a prior stage.
4. Candidates still in process must not automatically be treated as failed conversions.
5. An accepted offer later lost must not be back-written as a failed offer-stage conversion.

### Risk rules

1. Risk classification applies only to open requisitions / open positions.
2. Risk counts must use `openings_position`.
3. Risk band totals must reconcile to Total Open Positions for the same filter context.
4. Requisition-level joins must not duplicate `openings_position` when candidate-level data is joined.

### Quality rules

1. Every hire used in 60-Day Early Attrition must have a valid employee start date. Only candidates who actually started are hires; accepted offers that never started are excluded from both the numerator and the denominator.
2. A hire must not enter the 60-day denominator until the full 60-day observation window has elapsed.
3. Monthly trend points must use fully matured start months only.
4. `is_60_day_early_attrition = true` only when termination occurs from day 0 through day 60 after start.
5. 60-Day Early Attrition must remain between 0 and 1 before presentation formatting.
6. The rolling KPI must use the latest 12 fully matured start cohorts, not the latest 12 calendar months regardless of maturity.
7. The speed-versus-quality visual must calculate median Time to Fill from the same hire cohort used for the attrition point.
8. Duplicate hire or termination rows must not inflate either the numerator or denominator.
9. Cohort sample size must be available for quality checks and tooltip context.

### Forecast rules

1. Historical yields must not use future outcomes.
2. Stage yields must remain between 0 and 1.
3. The yield training label is the active fill, not the raw acceptance event and not the employee start.
4. Expected pipeline fills must not exceed remaining open positions per requisition.
5. Forecast filled positions must not exceed demand.
6. Forecast logic and fallback hierarchy must be documented and deterministic.

---

## 13. Required project outputs

The completed analytics project should provide:

### Data outputs

- dimension datasets
- requisition fact
- application / pipeline fact
- application stage-event fact
- offer fields or offer fact as required
- hire / early-outcome fact
- Executive Summary marts including quality
- CSV or Parquet outputs suitable for Power BI

### Documentation

- `README.md` — project purpose, setup, run instructions, and output summary
- `architecture.md` — pipeline layers, table relationships, grain, and data flow
- metric definition YAML or Markdown — governed Executive Summary metrics
- dataset schema YAML — columns, data types, keys, relationships, and descriptions

### Engineering requirements

#### Repository boundary

This repository is the **contract**. It defines the required grains, columns, metrics,
business rules, validation tests and the downstream dbt architecture. It contains no
executable pipeline code.

| Repository | Owns |
|---|---|
| **This repository (`ta-exec-db`)** | The dashboard specification, the wireframe, the governed metric definitions, the analytics dataset contracts, and the dbt architecture the implementation must follow |
| **Separate implementation repository** | Python synthetic-source generation, the dbt models, the executable tests, orchestration, and production of the CSV / Parquet outputs |
| **Power BI** | Consuming the validated outputs, and owning filter-responsive ratios, medians and presentation |

The dbt architecture described in this specification and in `dbt-ownership.md` is a
**required downstream implementation contract**. It is not a set of files that exist here.

#### Layer ownership

Each layer owns one kind of work. Governed business logic must not be independently
reimplemented across layers: one definition, in one place, that the others consume.
Reference calculations may legitimately appear in more than one place — marts store
row-grain rates and medians so a build can prove the mart and the fact agree — but those
are validation values, and the figure the report presents is still calculated once, by the
layer that owns it.

| Layer | Owns | Must not do |
|---|---|---|
| **Python** | Source generation: creating the synthetic ATS and HR records that simulate the source systems | Decide what a record means. Python may invent an offer acceptance date; it may not decide whether that acceptance is still an active fill |
| **dbt** | Every transformation between source and mart: grain resolution, row-level derivations, classifications, roll-ups, the forecast, and all validation tests | Store final rates or medians as the values the report presents |
| **Power BI** | Semantic aggregation under the user's filter context, and presentation | Re-implement any business rule, or recreate a count that requires a fact-to-fact relationship |

The decisive test for a calculation: if it needs the configured as-of date, a grain the
report model cannot reach, or a rule that must be identical in every visual, it belongs in
dbt. If it is dividing two additive columns or taking a median over a governed row-level
column in the current filter context, it belongs in Power BI.

`dbt-ownership.md` holds the full review and the model-by-model recommendation.

#### Requirements on the implementation

These are requirements this specification places on the separate implementation
repository.

- Python project managed with `uv` for synthetic-source generation
- dbt project for all transformations, contracts and tests, following the dependency flow
  in `schemas/README.md`
- deterministic configuration for the as-of date, read from `ref_reporting_config`; no layer reads the system clock
- governed vocabulary held as seeds (recruiting stages, risk bands, hiring constraints) and referenced by models, never hard-coded in transformation code
- clear source / staging / intermediate / fact / mart separation, with the dependency order derived from the model graph rather than maintained by hand
- the business rules in section 12 implemented as executable tests, including custom tests for the reconciliation and temporal rules that generic tests cannot express
- CSV or Parquet outputs produced for Power BI, matching the schema contracts in `schemas/`
- logging
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
- employee start date should support a separate cohort-date role for quality views
- quality visuals must be able to ignore the THD slicer while retaining valid Business Unit, Job Family, and Job Level filters
- metric logic must produce the same result whether calculated from the governed fact tables or validated against the executive marts
- numeric measures should remain numeric; presentation formatting belongs in Power BI

Where a calculation is highly reusable and business-critical, prefer creating a governed field or mart measure upstream rather than embedding equivalent logic in several visuals.

---

## 15. Acceptance criteria

The Executive Summary data project is complete when all of the following are true:

1. The project covers only the Executive Summary page scope defined here.
2. Data is reproducible using an as-of date of May 31, 2026.
3. Historical actual data covers January 2024 through May 31, 2026.
4. Requisition THDs through May 31, 2027 are preserved for future-demand analysis.
5. Fill Rate reconciles from requested, filled, and open position quantities, where filled means active fills.
6. Offer acceptance, active fill, and hire are modelled as three separate concepts; no metric is defined from an application status value.
7. `offer_accepted_date` is preserved after an employer rescind or a candidate renege, and a seat lost after acceptance is restated as open.
8. One application contributes at most one governed accepted-offer event: administrative revisions are collapsed, the earliest valid acceptance of the accepted cycle is preserved, ambiguous multiple-acceptance cases are quarantined for review, an audit model records every source application with multiple accepted versions and how it was resolved, the build fails only on an unclassified multi-version application or a quarantined application reaching the fact, and the limitation plus its remedy (a separate offer-event fact) are documented.
9. Median Time to Fill uses approval-to-offer-acceptance duration, over every accepted-offer event including those later rescinded or reneged.
10. Open-position risk is based on source TOAD and the configured as-of date.
11. High Risk is 0–7 days to TOAD; Medium Risk is 8–14 days; Missed is below 0.
12. At-Risk Open Positions counts open seats, not merely requisitions.
13. Hiring constraints are treated as requisition-level attributes.
14. Funnel metrics distinguish active pipeline from completed historical conversion, and an accepted offer remains a successful offer-stage conversion even if it is later lost before the start.
15. Forecast Fill Rate uses active pipeline stage-to-active-fill yield segmented by Business Unit, Job Family, and Job Level where data supports it.
16. Forecast expected fills are capped by remaining open positions.
17. No future actual recruiting outcomes leak into historical metrics or forecast training data.
18. 60-Day Early Attrition is included as the Executive Summary quality KPI and counts only candidates who actually started employment.
19. The 60-Day Early Attrition KPI uses a rolling 12-month window of fully matured start cohorts.
20. Immature hires and incomplete start months do not enter the quality denominator or trend.
21. The speed-versus-quality visual compares median Time to Fill and 60-Day Early Attrition for the same monthly matured hire cohorts.
22. The speed-versus-quality visual is treated as diagnostic association, not evidence of causation.
23. Required dimensions, facts, marts, documentation, and schema definitions are produced.
24. Power BI can build the Executive Summary page without reconstructing core business logic from raw source files.
25. Standalone Early Attrition and all other non-Executive Summary report pages remain outside the project scope.

---

## 16. Design principle

The project should remain intentionally focused.

A strong Executive Summary does not contain every recruiting metric. It gives leadership a concise view of:

**demand → delivery → speed → risk → pipeline → quality → expected outcome.**

Quality is represented by one governed outcome metric: **60-Day Early Attrition**. The purpose is to ensure the Executive Summary does not optimize only for filling roles quickly, but also shows whether those hires remain through the critical first 60 days.

Every dataset, metric, and transformation included in this phase should support that story. If an element does not materially help explain one of these executive areas, it should be excluded from the current project.