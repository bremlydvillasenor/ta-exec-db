# Metric Definitions — TA Analytics

**Scope:** Executive Summary and Early Attrition pages.
**Audience:** VP / Head of Talent Acquisition, business unit leaders, HR business partners, and the analytics team that maintains the model.

**Purpose of this document.** This is the governed business dictionary for the dashboard. For each metric it defines what the number means, how it is calculated, what it deliberately excludes, and what decision it supports.

**This document defines metrics. It does not report values.** No observed result appears here. Figures shown on the dashboard change every time the data is refreshed; the definitions do not. Keeping them apart means this document stays correct without maintenance, and it can be handed to a new analyst, a reviewer or a model as a standalone contract.

All configured numbers — targets, thresholds, risk bands, observation windows, service levels — are collected in section 8 and sourced from configuration, not from this text.

**Status: authoritative.** This document is the single source of truth for metric definitions. Where it conflicts with the project specification, the dashboard wireframe, the semantic model or any DAX measure, this document wins and the other artefact is corrected. Definitions change here first, then propagate.

**Reproducibility.** All logic that depends on "today", "current", "latest" or "open" is evaluated against a fixed reporting as-of date held in configuration. System time is never used. The same pipeline run in any future month returns the same result.

---

## 1. How to read each metric entry

Every metric below uses the same structure:

| Section | What it tells you |
|---|---|
| **Business question** | The question a leader is actually asking |
| **Plain-English definition** | One or two sentences, no formulas |
| **Formula** | Numerator ÷ denominator, or the calculation logic |
| **Date basis** | Which date decides whether a record belongs to the period |
| **Population** | Which records are counted, and at what grain |
| **Excluded** | What is deliberately left out, and why |
| **Comparison** | What the metric is measured against |
| **How to read it** | What good and bad look like |
| **Common misreadings** | The mistakes this metric invites |
| **Where to go next** | The drill path for investigation |

---

## 2. Shared concepts

These concepts apply across several metrics. Read this section once and the rest of the document becomes much easier.

**Requisition vs position.** A requisition is the hiring request. A position is one seat to be filled. One requisition can hold several positions, for example one requisition to hire a group of contact centre agents. Almost every metric on the Executive Summary Page counts **positions**, not requisitions, because leaders are accountable for positions filled. Where a metric counts requisitions, it says so explicitly.

**Demand.** The set of positions the business expects to be filled inside the selected period. Demand is anchored on **Target Hire Date**.

**Target Hire Date (THD).** The date the business needs the person to start. THD is the demand clock. It answers "when is the position needed?"

**Target Offer Acceptance Date (TOAD).** The date an offer must be accepted for the hire to start on time. TOAD is the internal delivery clock for TA. It answers "are we going to be late?" TOAD is the only anchor used for risk classification.

**Filled position.** A position where a candidate has **accepted** the offer on or before the as-of date. This is a deliberate governance choice: the recruitment process that TA controls ends at acceptance. Whether the person then starts on the target hire date depends on notice periods, onboarding and start-date scheduling, which sit outside recruitment. An acceptance can still fall through before the target hire date, so acceptances that do not convert to a start are tracked separately rather than being removed from this measure.

**Started position.** A filled position where the new hire has actually begun work. Starts are tracked separately from fills, because they arrive later and are governed by notice periods and onboarding rather than by recruitment. Starts are the population used for all Early Attrition reporting.

**Open position.** A seat on a requisition that has not been filled yet.

This is not worked out on the fly. It is stored as `openings_position`, a column on `fct_requisition` holding the number of seats still to be filled on that requisition. Total Open Positions is the sum of that column across requisitions with status Open.

The column decreases by one when a candidate accepts an offer, which is the same event that adds one to Positions Filled. Both measures therefore move at the same moment and always reconcile back to the size of the requisition:

```
requested_positions = filled_positions + openings_position
```

Example. A requisition for three contact centre agents starts with `openings_position` equal to three and `filled_positions` equal to zero. When the first candidate accepts, the values become two and one. That candidate's actual start date, whenever it happens, changes neither value.

**The measurement seam between the two pages.** The Executive Summary counts fills at **offer acceptance**. Early Attrition counts hires at **start date**. This is intentional and unavoidable: retention cannot be measured for someone who has not started. It means the two pages report on overlapping but different populations, and the Early Attrition population always lags the Executive Summary population by the gap between acceptance and start. Never expect the hire counts on the two pages to reconcile.

**Cohort.** A group of new hires who started in the same selected cohort period. Cohorts are used for early attrition reporting.

**Cohort maturity.** A hire only becomes eligible for the early attrition metric once the full observation window has passed since their start date. A cohort is treated as **fully matured** only when *every* hire in that start month has completed the window. This is the single most important rule on the Early Attrition page.

**pp (percentage points).** The difference between two percentages. A move from one percentage to another is expressed in percentage points, not as a percentage, to avoid confusion with relative change.

**PYTD.** Prior year to date. The same calendar window one year earlier, used for like-for-like comparison.

**Target vs threshold vs watch level.** A *target* is the performance commitment for the fiscal year. A *threshold* is a maximum tolerance, used for risk exposure. A *watch level* is an earlier warning line that triggers investigation before the target is formally breached. All three are configured values, listed in section 8.

**How the Date Range slicer behaves.** Every visual on the Executive Summary is filtered by **Target Hire Date**, with one exception.

*Filtered by THD* — Fill Rate, Time to Fill, the funnel, open positions, the risk bands, At-Risk Rate and the hiring constraint mix. Changing the range changes which positions are counted. Select a prior year and the page reports on that year's demand.

*Not filtered* — the early attrition card. It always shows the rolling window of fully matured cohorts, because that window is set by cohort maturity against the as-of date rather than by user selection. A shortened range would include cohorts that have not completed the observation window and would report a falsely low rate.

Business Unit and Job Family filters apply to everything.

One demand window across the page keeps the visuals reconciled with each other. Because a position is open precisely when it has no accepted offer, the open-position count equals Unfilled Demand, and the risk bands are a breakdown of the Fill Rate shortfall rather than a separate book of work.

---

## 3. Metric index

| ID | Metric | Page | Type | Date basis | Comparison |
|---|---|---|---|---|---|
| EXEC-01 | Fill Rate (Positions Filled vs Demand) | Executive Summary | Ratio | Target Hire Date | FY target |
| EXEC-02 | Positions Filled | Executive Summary | Sum | Target Hire Date | — |
| EXEC-03 | Demand (Positions Requested) | Executive Summary | Sum | Target Hire Date | — |
| EXEC-04 | Pending Starts | Executive Summary | Sum | THD > as-of-date | — |
| EXEC-05 | Fill Rate Trend (cumulative YTD) | Executive Summary | Trend | Target Hire Date | FY target |
| EXEC-06 | Forecast Fill Rate | Executive Summary | Forecast | Target Hire Date | FY target |
| EXEC-07 | Total Open Positions | Executive Summary | Sum | Target Hire Date | — |
| EXEC-08 | Days to TOAD and Risk Band | Executive Summary | Classification | TOAD vs as-of date | Configured bands |
| EXEC-09 | Open Positions at Risk / At-Risk Rate | Executive Summary | Count + ratio | Target Hire Date; TOAD vs as-of date | Threshold, PYTD |
| EXEC-10 | Primary Hiring Constraint | Executive Summary | Mix | Target Hire Date; latest weekly status | — |
| EXEC-11 | Time to Fill (median) | Executive Summary | Duration | Target Hire Date | FY target, PYTD |
| EXEC-12 | Funnel Volume by Stage | Executive Summary | Count | Target Hire Date | — |
| EXEC-13 | Stage-to-Stage Conversion | Executive Summary | Ratio | Target Hire Date | Per-stage target |
| EXEC-14 | Cumulative Conversion | Executive Summary | Ratio | Target Hire Date | — |
| EXEC-15 | Median Days in Stage | Executive Summary | Duration | Stage entry | Per-stage SLA |
| EXEC-16 | Bottleneck Stage | Executive Summary | Derived flag | Target Hire Date | Per-stage target and SLA |
| EXEC-17 | Early Attrition (rolling matured cohorts) | Early Attrition | Ratio | Employee start date | FY target, prior period |


---

## 4. Executive Summary metrics

### EXEC-01 — Fill Rate (Positions Filled vs Demand)

**Business question.** Of the positions the business needed filled in this period, how many did we actually fill?

**Plain-English definition.** Fill Rate compares hiring delivery against hiring demand. Demand is fixed by when the business needed people, not by when recruitment happened to finish. A position counts as filled once the candidate has accepted the offer, because that is the point at which the recruitment process is complete.

**Formula.**
```
Fill Rate = Positions Filled ÷ Positions with a Target Hire Date in the period
```

**Date basis.** Target Hire Date. Both sides of the ratio are anchored to the same demand window, so the metric answers "did we deliver what was due?" rather than "how busy were we?"

**Population.** All positions on non-cancelled requisitions with a THD inside the selected period.

**Excluded.** Cancelled demand, because the business withdrew the requirement. Offers extended but not accepted, because the candidate has not committed.

**Comparison.** FY Fill Rate target.

**How to read it.** The shortfall is the share of needed positions with no accepted offer by the time they were due. The number is a delivery statement.

Because the metric stops at acceptance, it measures what TA controls. It does **not** confirm that the business received the capacity. For that, read it together with pending starts in EXEC-04.

**Common misreadings.**
- Reading the percentage as seats occupied. It is seats with a signed acceptance. Some of those people will not have started yet.
- Reading the cumulative year-to-date figure as current performance. See EXEC-05.

**Where to go next.** EXEC-04 (where the gap sits), EXEC-09 (which open positions are late), EXEC-13 (where candidates are lost).

---

### EXEC-02 — Positions Filled

**Business question.** How many positions did recruitment successfully close?

**Plain-English definition.** The sum of positions filled in requisition.

**Formula.** `Sum of filled positions in requisition.`

**Date basis.** Target Hire Date of its requisition for period attribution.

**Excluded.** Offers extended but not accepted. Declined offers. Cancelled requisition.

**How to read it.** This is the numerator of Fill Rate and the measure of TA output. It closes at the last event recruitment controls.

**Common misreadings.**
- Reading this as headcount on the payroll. It is not. Finance and operations recognise starts, and the two figures always differ by the pending-start population.
- Confusing this with the number of requisitions closed. A requisition normally holds more than one position.

---

### EXEC-03 — Demand (Requested Positions)

**Business question.** How many approved positions did the business commit to filling in this period?

**Plain-English definition.** The total number of positions with a Target Hire Date inside the selected period, across all non-cancelled requisitions.

**Formula.** `Sum of positions on requisitions where Target Hire Date is in the selected period.`

**Date basis.** Target Hire Date.

**Excluded.** Cancelled requisitions and cancelled positions.

**How to read it.** Demand is the denominator that makes Fill Rate fair. It is set by workforce planning and hiring managers, not by TA.

**Common misreadings.** Treating Demand as a TA-controlled number. TA influences delivery, not the plan.

---

### EXEC-04 — Unfilled Demand and Pending Starts

**Business question.** How many of the demand has been filled in advance?

**Plain-English definition.** Pending Starts is the accepted positions where the person has not yet begun work.

**Formula.**
```
Unfilled Demand   = Demand − Positions Filled            (no accepted offer)
Pending Starts    = Positions Filled − Started Positions  (accepted, not yet begun)
```

**Date basis.** Target Hire Date for scope.

**Pending Starts** Recruitment finished in advance. It represents filled positions that has been secured but has not yet arrived, because target hire date is the future.

---

### EXEC-05 — Fill Rate Trend (Cumulative YTD)

**Business question.** Is delivery against the annual commitment improving as the year progresses?

**Plain-English definition.** A running total of positions filled divided by a running total of positions due, from the start of the fiscal year. Each month adds to both sides of the ratio rather than replacing them.

**Formula.**
```
Cumulative Fill Rate (month M) = Σ Positions Filled (FY start .. M)
                                 ÷ Σ Positions Due by THD (FY start .. M)
```

**Date basis.** Target Hire Date.

**How to read it.** Cumulative measures are correct for **attainment** reporting against an annual target. They are slow to move because early months stay in the calculation all year. A rising cumulative line means recent months are performing better than the year-to-date average, but it does not by itself prove recovery.

**Common misreadings.** Using the rising cumulative line as evidence that the problem is solved. For directional performance, use the monthly THD-cohort Fill Rate, which shows each month on its own. This is the difference between "the year is catching up" and "this month was good".

---

### EXEC-06 — Forecast Fill Rate

**Business question.** Based on the candidates we have right now, where will we land at year end?

**Plain-English definition.** A projection that adds expected future fills from the live candidate pipeline to the fills already achieved. Each active candidate is weighted by how often candidates at that stage, in that Business Unit, Job Family and Job Level, have historically converted to an accepted offer.

**Formula.**
```
Expected Pipeline Fills   = Σ (active candidates × historical stage-to-acceptance yield
                                for their segment)
                            capped at remaining open positions per requisition
Forecast Filled Positions = Actual Filled Positions + Expected Pipeline Fills
Forecast Fill Rate        = Forecast Filled Positions ÷ Demand
```

**Date basis.** Target Hire Date, consistent with EXEC-01.

**Excluded.** Any event after the as-of date is excluded from the historical yield calculation, so the forecast cannot borrow information from the future. Sparse segments fall back to a broader grouping so that small populations do not produce unstable projections. The fallback hierarchy is configured, not improvised.

**How to read it.** The dashed forecast line is a planning aid, not a commitment. It says whether the current pipeline is sufficient to reach target if conversion behaves as it has historically.

The yield being applied is stage-to-**acceptance**, not stage-to-start. This makes the forecast shorter-horizon and more reliable, because it no longer has to predict start-date scheduling that sits outside recruitment.

**Common misreadings.**
- Treating the forecast as a promise. It assumes historical conversion holds.
- Assuming pipeline strength changes risk classification. It does not. Risk is set by TOAD alone (EXEC-08).

---

### EXEC-07 — Total Open Positions

**Business question.** How large is the open hiring book right now?

**Plain-English definition.** The number of unfilled seats on open requisitions whose Target Hire Date falls in the selected period.

**Formula.** `Sum of openings_position across requisitions with status = Open and a Target Hire Date in the selected period.`

**Date basis.** Target Hire Date, the same basis as Demand and Fill Rate.

**Reconciliation.** Because a position is open precisely when it has no accepted offer, this measure and Unfilled Demand in EXEC-04 describe the same population:

```
Total Open Positions (period) = Demand − Positions Filled = Unfilled Demand
```

This identity should hold on every filter combination and is a useful validation test.

**How to read it.** This is the denominator for the risk module, and it is also the Fill Rate shortfall. The risk bands in EXEC-08 are therefore a breakdown of positions the page has already reported as missed, which is what makes the Executive Summary read as a single chain rather than as separate scorecards.

**Common misreadings.**
- Reading this as the size of the whole open hiring book. It is not. Open positions with a Target Hire Date beyond the selected period are out of scope on this page. For the full portfolio, use the At Risk Requisitions detail page.
- Expecting it to differ from Unfilled Demand. It should not.

---

### EXEC-08 — Days to TOAD and Risk Band

**Business question.** Which open positions are going to be late, and how late?

**Plain-English definition.** For every open position, count the days between the observation date and the date by which an offer must be accepted. That number places the position in one of four bands.

**Formula.**
```
Days to TOAD = Target Offer Acceptance Date − as-of date

Overdue    Days to TOAD < 0                                TOAD already passed
High       0 ≤ Days to TOAD ≤ high_risk_max
Medium     high_risk_max < Days to TOAD ≤ medium_risk_max
On Track   Days to TOAD > medium_risk_max
```

Band boundaries are configured (section 8), not hardcoded in the report.

**Date basis.** Two dates do two different jobs. Target Hire Date decides whether a position is in scope for the selected period. Target Offer Acceptance Date, compared against the fixed as-of date, decides which band it lands in. The band is evaluated once per requisition, and all remaining openings on that requisition inherit it.

**Expected band distribution.** TOAD precedes THD by the expected notice period, so positions whose THD falls inside a period ending on or before the as-of date will mostly classify as Overdue. This is correct behaviour, not a data problem. The High, Medium and On Track bands only populate meaningfully when the selected range extends into future months.

**Excluded from the logic.** Recruitment stage, stage ageing, pipeline strength and projected remaining duration. These are deliberately kept out. Risk is a **schedule** measure, and mixing schedule with pipeline judgement makes the classification impossible to audit or compare across teams. Those fields are still available as diagnostics (EXEC-10).

**How to read it.** The bands are a triage queue. Overdue positions have already missed the point at which an on-time start was possible. High and Medium are still recoverable with intervention.

**Common misreadings.** Assuming a position with a strong pipeline should be downgraded out of the risk bands. It should not. The band describes time remaining, and the pipeline diagnostic sits beside it to explain why.

---

### EXEC-09 — Open Positions at Risk / At-Risk Rate

**Business question.** How much of the open hiring book is exposed to missing its target start date?

**Plain-English definition.** The count and share of open positions that are Overdue, High or Medium risk against their Target Offer Acceptance Date.

**Formula.**
```
Open Positions at Risk = Overdue + High + Medium
At-Risk Rate           = Open Positions at Risk ÷ Total Open Positions
```

**Date basis.** Target Hire Date for scope; TOAD against the fixed as-of date for classification. The card answers "of the demand due in this period, how much is still unfilled and how late is it?"

**Comparison.** A configured maximum threshold, plus the same period one year earlier.

**What the PYTD comparison means here.** Because scope is filtered by Target Hire Date but lateness is always measured against the current as-of date, the prior-year figure reads as "of last year's demand, how much remains unfilled and overdue today". It is not a reconstruction of how exposed the book looked a year ago. That is a legitimate and useful comparison, but it is a different question, and anyone presenting the movement should say which one they mean.

**How to read it.** This module explains the Fill Rate shortfall rather than adding a separate measure. The positions counted here are the same positions Fill Rate reported as missed, sorted by how overdue they are.

Read the count of affected **requisitions** alongside the count of positions. Risk is usually concentrated on relatively few requisitions, which means intervention can be targeted at a small number of conversations rather than spread thinly.

When the selected range extends into future months, the metric also becomes forward-looking, because positions not yet due appear in the High, Medium and On Track bands. For a period that has already closed, treat it as a lateness breakdown of a known shortfall.

**Common misreadings.**
- Treating this as the whole open hiring book. It is only the part with a Target Hire Date in the selected period.
- Being surprised that almost everything is Overdue on a closed period. See the expected band distribution note in EXEC-08.
- Reading the rate as a failure rate. Some of these positions will still be recovered, and the denominator is already the shortfall population.

**Where to go next.** EXEC-10 for the reason behind each at-risk position.

---

### EXEC-10 — Primary Hiring Constraint

**Business question.** What is actually blocking the positions that are at risk?

**Plain-English definition.** Each open requisition carries one primary blocker, recorded by the TA leadership team in a weekly status review. Every remaining opening on that requisition inherits that blocker, so the chart can be read in positions.

**Formula.** `Count of at-risk positions grouped by the primary_hiring_constraint on their requisition, taken from the latest valid weekly status record on or before the as-of date.`

**Date basis.** Target Hire Date for scope, inherited from the at-risk population; latest valid weekly status record on or before the as-of date for the constraint value.

**Constraint values and ownership.** The value list is governed. Ownership is what makes the breakdown actionable.

| Constraint | Owner |
|---|---|
| Hiring manager delays | Hiring managers |
| Low candidate pipeline | TA sourcing |
| Hard-to-fill / niche skills | TA sourcing and market conditions |
| Interview scheduling delays | Hiring managers and coordination |
| Compensation constraints | Reward / finance |
| High candidate fallout | TA and candidate experience |
| Offer approval delays | Approvers |
| Background check delays | Vendor |

**How to read it.** This is a diagnostic layer, not a classifier. It never changes the risk band. Its value is accountability: a material share of at-risk positions is normally blocked by causes owned outside TA. Presenting risk without this breakdown invites the assumption that all delay is a recruitment failure.

**Common misreadings.**
- Treating constraint counts as a second risk measure. They are a decomposition of the same at-risk population.
- Expecting more than one constraint per requisition. One primary constraint is recorded by design, to keep the totals additive.

---

### EXEC-11 — Time to Fill (Median)

**Business question.** How long does recruitment take from approval to a signed acceptance?

**Plain-English definition.** The median number of calendar days between a requisition being approved and a candidate accepting the offer, across fills completed in the period.

**Formula.**
```
Time to Fill (per fill) = Offer Accepted Date − Requisition Approval Date
Reported value          = Median across all completed fills in the period
```

**Date basis.** Offer accepted date determines which period a fill falls into.

**Population.** Completed fills only.

**Excluded.** Requisitions still open, because including them would understate duration. This is survivorship by design and is the reason Time to Fill can look stable while At Risk deteriorates: the slowest requisitions are still open and have not yet entered the calculation.

**Comparison.** FY target and PYTD.

**Definitional coherence.** Time to Fill ends at offer acceptance, and a position is counted as filled at offer acceptance. The two definitions stop at the same event, so Time to Fill measures exactly the duration that Fill Rate counts as complete.

**Why median rather than average.** A small number of very hard roles can add weeks to an average and make a normal month look like a crisis. The median describes the typical requisition, which is the more useful management number.

**How to read it.** Read alongside EXEC-09. Speed improving while risk exposure rises is a meaningful signal that the difficult work is accumulating in the still-open book rather than in completed fills.

**Common misreadings.**
- Adding up the stage medians in the funnel to reconstruct Time to Fill. They will not reconcile. See EXEC-15.
- Reading a fall in Time to Fill as unambiguously good. Fast hiring with weak screening is exactly the pattern that appears two months later on the Early Attrition page.

---

### EXEC-12 — Funnel Volume by Stage

**Business question.** How many candidates reached each stage of the recruitment process?

**Plain-English definition.** The count of candidates who entered each stage, from application to offer acceptance, for applications received in the period.

**Stage sequence.** Stage order, conversion target and service level are held in `dim_stage` and configuration, not in the report.

| Order | Stage |
|---|---|
| 1 | Application Received |
| 2 | Recruiter Screen |
| 3 | Assessment |
| 4 | Interview |
| 5 | Reference Check |
| 6 | Offer Extended |
| 7 | Offer Accepted |

**Date basis.** Application date.

**How to read it.** The funnel ends at Offer Accepted, which is also the point at which a position counts as filled. The funnel therefore reconciles directly to the Fill Rate numerator: acceptances equal positions filled. Some of those people will not have started yet, which is the pending-start population in EXEC-04.

**Common misreadings.** Assuming acceptances equal people at work. They equal completed recruitments.

---

### EXEC-13 — Stage-to-Stage Conversion

**Business question.** At which step are we losing candidates faster than we should?

**Plain-English definition.** The percentage of candidates entering a stage who progress to the next stage, compared with the target conversion rate for that stage.

**Formula.** `Stage Conversion = Candidates entering next stage ÷ Candidates entering this stage`

**Date basis.** Application date.

**How to read it.** Conversion is read against its own target, not against other stages. A low conversion rate is not automatically bad, and a high one is not automatically good. Converting well above target early in the funnel means screening is letting more people through than planned, which pushes weaker candidates into expensive later stages. That is a quality signal worth investigating, not a success.

**Common misreadings.**
- Comparing conversion rates between different stages. They have different targets and different purposes.
- Reading high conversion as always positive.

---

### EXEC-14 — Cumulative Conversion

**Business question.** How many applications do we need to produce one filled position?

**Plain-English definition.** The percentage of the original application base that survives to each stage.

**Formula.** `Cumulative Conversion = Candidates at stage ÷ Total applications`

**Date basis.** Application date.

**How to read it.** The end-to-end value is the sourcing volume planning number. Because a fill is an acceptance, its inverse is the number of applications required per filled position. If Demand rises, the application base must rise by a similar factor unless conversion improves.

**Common misreadings.** Treating cumulative conversion as a quality measure. It is primarily a volume-planning measure and is heavily influenced by sourcing channel mix.

---

### EXEC-15 — Median Days in Stage

**Business question.** Which stage is slow, and is it slower than we allow?

**Plain-English definition.** The median calendar days a candidate spends in a stage, measured from stage entry to the next stage entry or final outcome, compared with the agreed service level for that stage.

**Formula.** `Median (Next stage entry date − This stage entry date), across completed stage records.`

**Date basis.** Stage entry date.

**Excluded.** Candidates still sitting in a stage. Including them would bias the median downward, because slow-moving candidates have not finished yet.

**Comparison.** The configured per-stage SLA.

**Important reconciliation note.** Stage medians use different populations at each stage and therefore **do not add up to Time to Fill**. The stage medians answer "how slow is this step for a typical candidate?" while Time to Fill answers "how long did a typical successful fill take end to end?" This is stated on the dashboard itself because it is the single most common question asked about the funnel.

**How to read it.** Compare each median to its own SLA, not to the other stages.

---

### EXEC-16 — Bottleneck Stage

**Business question.** If we could fix one step in the process, which one?

**Plain-English definition.** The stage with the worst combined performance on speed and conversion against its own targets. It is derived from the data rather than written in by hand, so it moves as performance changes.

**Formula.** `Rank stages by variance against SLA and variance against target conversion, among stages with sufficient candidate volume. Flag the worst.`

**Excluded.** Stages below the configured minimum volume, so that a small population cannot produce an unstable callout.

**How to read it.** Two dimensions matter, and their combination determines the action:

| Pattern | Meaning | Action |
|---|---|---|
| Slow **and** low conversion | True bottleneck | Process intervention |
| Slow but high conversion | Capacity-constrained, still working | Add capacity |
| Fast but low conversion | Quality or sourcing problem | Fix candidate fit upstream |

**Common misreadings.** Assuming a flagged bottleneck is a TA failure. The constraint frequently sits with hiring managers, which the constraint breakdown in EXEC-10 will confirm or contradict.

---

## Early Attrition metrics

### EXEC-17 — Early Attrition (Rolling Matured Cohorts)

**Business question.** Of the people we hired, what share left within their first weeks?

**Plain-English definition.** The percentage of new hires who left within the observation window of starting, measured across the rolling set of fully matured monthly cohorts.

**Formula.**
```
Early Attrition = Hires terminating within observation_window_days of start
                  ÷ Hires eligible for observation (mature cohorts only)
```

**Date basis.** Employee start date determines cohort membership. Termination date determines the outcome.

**Excluded.** Hires who have not completed the observation window. Immature cohorts are outside the calculation entirely, on both sides of the ratio.

**Comparison.** FY target and the equivalent prior period.

**Slicer behaviour.** This metric appears on both pages and behaves the same way on each: the Date Range slicer does not apply to it. On the Early Attrition page the cohort window is shown as its own control, set automatically to the fully matured cohorts. On the Executive Summary the card is static for the same reason. Business Unit and Job Family filters do apply on both pages.

**How to read it.** This is the quality counterweight to the delivery metrics on the Executive Summary. Fill Rate and Time to Fill measure whether TA delivered. This measures whether what was delivered stayed. A hire who leaves in week seven consumed full recruitment cost, produced little, and returns the requisition to the open book.

Read it against Time to Fill in particular. Attrition rising while Time to Fill improves is the signal that speed may be coming at the cost of hire quality, and it is the reason both pages must be read together.

**Common misreadings.**
- Including immature cohorts to make the number look better. This is the failure mode the whole page is designed to prevent.
- Reading a low percentage as a small problem. Convert it to a count. Each one is a full recruitment cycle repeated.
- Assuming all early leaving is a TA problem. See ATTR-03.

**Where to go next.** ATTR-05 for the trend, ATTR-08 for the cause, ATTR-02 for the most recent complete signal.

| Reason | Attribution |
|---|---|
| Role / job expectation mismatch | TA-related |
| Compensation expectation mismatch | TA-related |
| Work arrangement / location mismatch | TA-related |
| Schedule / shift mismatch | TA-related |
| Expectation setting during hiring | TA-related |
| Onboarding / manager experience | Not TA-related |
| Performance / capability gap | Not TA-related |
| Personal / health / other | Not TA-related |
| Counter-offer from former employer | Not TA-related |
| Not coded / no exit interview | Unknown |

**How to read it.** The four expectation-mismatch reasons (role, pay, work arrangement, schedule) are the clearest recruitment-quality signal available, because they all point at the same fixable behaviour: what candidates are told during screening and at offer.

Concentration matters as much as the total. A problem spread evenly across every business unit and job family is a systemic issue. A problem concentrated in one cohort, one function or one role type is a solvable one, and the reason detail is where that concentration becomes visible.

**Common misreadings.**
- Adding percentages across the TA and non-TA lists and expecting them to reach 100%. The unknown group must be included.
- Treating "counter-offer from former employer" as a TA failure. It is attributed outside TA because it reflects a candidate's existing employer acting after acceptance.

---

## 6. Reading the two pages together

The metrics are designed to be read as one story, not as two independent scorecards. Four relationships carry most of the meaning.

**Fill Rate against the risk bands.** These are the same positions viewed twice. Fill Rate says how many were missed; the bands say how late those same positions are; the constraint mix says what is blocking them. If the open-position count ever stops matching the Fill Rate shortfall, the model is wrong.

**Fill Rate against Demand.** A falling Fill Rate does not prove that recruitment got worse. Check whether Demand rose. Delivery can be flat while the ratio deteriorates, which is a planning conversation rather than a performance conversation.

**Time to Fill against At-Risk exposure.** Time to Fill only counts completed fills, so improving speed and rising risk can be true at the same time. When both move in that pattern, the difficult requisitions are accumulating in the still-open book and have not yet reached the speed calculation.

**Bottleneck and constraint against ownership.** The funnel bottleneck and the hiring constraint breakdown usually agree with each other. When they both point outside TA, the delivery shortfall is a shared accountability problem and should be presented as one.

**Speed against early attrition.** This is the most important pairing on the dashboard. Fast hiring is not automatically good hiring. When Time to Fill improves while early attrition worsens, the likely explanation is that screening depth and expectation setting were traded away for speed. The Executive Summary alone would show that as success. The two pages together show it as a trade-off, which is the reason both exist.

The dashboard is deliberately built so that a delivery signal and a quality signal cannot hide each other.

---

## 7. Governance rules

These rules exist to keep the numbers defensible. They should not be changed without a documented decision recorded in section 9.

| Rule | Reason |
|---|---|
| A position is filled at offer acceptance | Acceptance is the last event recruitment controls; start-date scheduling is not a TA outcome |
| Started positions are reported separately, never merged into Fill Rate | The business needs both the recruitment result and the capacity result, and they are different numbers |
| A position leaves the open book at acceptance | Keeps the open book, TOAD risk and the Fill Rate numerator stopping at the same event |
| Fill Rate is anchored on Target Hire Date | The denominator must reflect what the business needed, not when recruitment finished |
| Risk is classified by TOAD alone | Schedule risk must be auditable and comparable; pipeline judgement is a separate diagnostic |
| Hiring constraints never change a risk band | Diagnostics explain risk, they do not reclassify it |
| One primary constraint per requisition | Keeps position counts additive and prevents double counting |
| Every Executive Summary visual except the attrition card is filtered by Target Hire Date | One demand window across the page; the risk bands become a breakdown of the Fill Rate shortfall rather than a separate book of work |
| Total Open Positions for a period must equal Unfilled Demand | The two are the same population by definition; any difference is a defect |
| Risk banding always compares TOAD against the fixed as-of date, never against the end of the selected range | Lateness is a fact about today, not about the period being viewed |
| The full open-book portfolio view lives on the At Risk Requisitions detail page | The Executive Summary trades portfolio coverage for reconciliation; the coverage still has to exist somewhere |
| Immature cohorts never enter an attrition denominator | Otherwise recent hires are silently counted as retained |
| A monthly cohort is shown only when fully matured | Partial cohorts always understate attrition |
| The Executive Summary attrition card ignores the date range | A shortened window would include cohorts that have not completed the observation period |
| Executive Summary counts acceptances, Early Attrition counts starts | Retention cannot be measured before someone starts; the seam is documented, not hidden |
| Uncoded exits stay in the overall rate, outside the TA split | The headline is never understated and neither side of the split is inflated |
| Stage medians are not expected to sum to Time to Fill | Different populations, different questions |
| All "current" logic uses the fixed as-of date | Results must be reproducible regardless of when the pipeline runs |

---

## 8. Governed parameters

Every number the dashboard compares against lives in configuration, not in a measure or in this text. This section names the parameters and where they are held. Values are read from the configuration files at run time.

**Reporting window** — `config/business_rules.yml`

| Parameter | Meaning |
|---|---|
| `data_start_date` | First date covered by the dataset |
| `data_end_date` | Last date of completed business events |
| `as_of_date` | The fixed reporting date for all "current" logic |
| `max_target_hire_date` | Planning horizon for open requisitions |

**Performance targets** — `config/targets.yml`

| Parameter | Applies to |
|---|---|
| `fill_rate_target` | EXEC-01, EXEC-05, EXEC-06 |
| `time_to_fill_target_days` | EXEC-11 |
| `at_risk_rate_threshold` | EXEC-09 |

**Risk bands** — `config/business_rules.yml`

| Parameter | Applies to |
|---|---|
| `high_risk_max_days` | Upper bound of the High band, in days to TOAD |
| `medium_risk_max_days` | Upper bound of the Medium band, in days to TOAD |


**Value lists** — `config/business_rules.yml`

| Parameter | Applies to |
|---|---|
| `hiring_constraint_values` | EXEC-10 |
| `min_stage_volume` | EXEC-16 bottleneck eligibility |

**Rule.** A target or threshold must never be hardcoded in a DAX measure, a Python module or this document. If a comparison value appears anywhere other than configuration, it is a defect.

---

## 9. Change log

| Version | Change |
|---|---|
| 1.0 | Initial version. Executive Summary and Early Attrition pages. |
