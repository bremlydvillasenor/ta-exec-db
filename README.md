# TA Executive Dashboard

Specification and analytics data contracts for a Talent Acquisition Executive Summary
dashboard: a one-page executive view of hiring demand, delivery, risk, pipeline and hiring
quality, reported against a fixed as-of date of **2026-05-31**.

## What this repository contains

This repository is the **definition layer**. It states what must be built and what must be
true, not how it is built.

| File | Contents |
|---|---|
| `spec.md` | The dashboard specification: scope, governed vocabulary, metric rules, data requirements, business rules and acceptance criteria |
| `metric-def.yaml` | Metric definitions — EXEC-01 to EXEC-14, FCST-01 to FCST-04, SUPP-01 to SUPP-08 — with grain, filters, source dataset and DAX |
| `wireframe.html` | The page layout the metrics are built for |
| `schemas/` | Dataset contracts: grain, columns, relationships, business rules and required data-quality tests for every dimension, fact, mart and reference table |
| `dbt-ownership.md` | Which calculations belong in dbt, which stay in Power BI, and the dbt architecture the implementation must follow |

## What this repository does not contain

**No Python and no dbt project files.** Raw synthetic ATS and HR source generation, the
executable dbt models, tests, orchestration and the CSV / Parquet outputs are built in a
**separate implementation repository**. The dbt architecture described in
`dbt-ownership.md` is the required downstream implementation contract, not code that is
expected to live here.

## Responsibility split

| Layer | Owns |
|---|---|
| **This repository (`ta-exec-db`)** | Required grains, columns, metrics, business rules, validation tests, and the dbt architecture the implementation must follow |
| **Implementation repository** | Python synthetic-source generation, dbt models, executable tests, orchestration, and production of the CSV / Parquet outputs |
| **Power BI** | Consuming the validated outputs, and owning filter-responsive ratios, medians and presentation |

The rule behind the split: governed business logic is defined once, by the layer that owns
it, and consumed by the others. Reference calculations may legitimately appear in more than
one place — the marts store row-grain rates and medians so a build can prove the mart and
the fact agree — but those are hidden validation values. The figure an executive reads is
calculated once.

## Where to start

1. `spec.md` — sections 5 (governed vocabulary) and 12 (business rules) carry the rules
   everything else depends on.
2. `schemas/README.md` — the star schema, the dependency flow the models must satisfy, and
   the wireframe-to-dataset map.
3. `dbt-ownership.md` — the required dbt project shape and the build phases.
