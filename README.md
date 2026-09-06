# TA Executive Dashboard

Contracts for a one-page Power BI Talent Acquisition Executive Summary covering
demand, delivery, speed, risk, pipeline and early attrition as of **2026-05-31**.
This repository defines required behavior; executable Python and dbt projects
live separately. Current contract release: **1.3**.

## Source of truth and authority

When files disagree, use this order, highest first. File format does not determine
authority: a Markdown contract can be as binding as YAML.

| Rank | File | Authority |
|---|---|---|
| 1 | [spec.md](spec.md) | Scope, business meaning, reporting periods and acceptance criteria |
| 2 | [metric-def.yaml](metric-def.yaml) | Metric populations, filters, formulas and DAX within the spec |
| 3 | [schemas/](schemas/) dataset YAML files | Analytics output grains, columns, types, relationships and tests |
| 4 | [raw-data-generation-contract.md](raw-data-generation-contract.md) | Raw ATS/HR inputs, simulation rules and generator deliverables |
| 5 | [dbt-ownership.md](dbt-ownership.md) | Transformation ownership, dependencies and implementation guidance |
| 6 | README files and other supporting documentation | Navigation, explanations and examples; this table defines the ordering |
| 7 | [wireframe.html](wireframe.html) | Layout and visual styling only; always last |

Apply the higher-ranked rule and correct the lower-ranked file in the same change.
A lower-ranked file may add detail where a higher-ranked file is silent; it may not
redefine a metric or expand scope. Explicit project-owner decisions authorize
contract changes and should be recorded in the appropriate governing file.
Wireframe numbers are illustrations, never generation targets or acceptance tests.

## Repository responsibilities

| Repository / deliverable | Owns |
|---|---|
| This repository (`ta-exec-db`) | Business, metric, raw-input and analytics-output contracts |
| [Raw-data generator](https://github.com/bremlydvillasenor/ta-exec-db-data-gen) | Separate uv + Python + Polars project; synthetic ATS/HR CSVs and source validation |
| Separate dbt repository (link to be added when created) | Ingestion, transformations, executable business tests and analytics exports |
| Power BI report | Relationships, filter-responsive ratios and medians, presentation |

Raw offer input is one current row per application in `offers.csv`; no offer-version
resolution is required. Every synthetic raw file includes `updated_at` and
`extracted_at`, with their different meanings defined in the raw-data contract.

Both implementation repositories must record the **contract release and exact
commit SHA** they implement. A README entry and run manifest are sufficient; no
package registry or automatic contract synchronization is required.

## Where to start

1. Read `spec.md` for business scope and acceptance examples.
2. For generation, use `raw-data-generation-contract.md` and spec section 11.
3. For dbt, use `metric-def.yaml`, dataset YAML files and `dbt-ownership.md`.
4. For Power BI, use `schemas/README.md`, metric DAX, then the wireframe.

## Portfolio evidence

As implementation becomes available, link reproducible run instructions, a successful
dbt test summary, lineage image and Power BI screenshot here. Add a short narrative
connecting synthetic findings to executive decisions. Do not present planned features
or wireframe numbers as implemented results.
