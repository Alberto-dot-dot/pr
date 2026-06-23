# Stack Rules — PR

Reference material only. Derived mechanically from Bloque 3 of the closed
specification (estado-spec-pr.md, S21). No new decisions are made here.

## Mandated Stack

- **Polars** — the row-level ETL engine. All row-level transforms (5.1
  normalization, hashing, joins, gates) run in Polars. No other row engine
  is permitted.
- **BigQuery** — staging (`pr_staging`) and serving (`pr_serving`) datasets,
  physically separated (D-10.4).
- **Cloud Run Job + Cloud Scheduler** — the autonomous weekly production
  runtime. Run-to-completion Job, no HTTP service. Scheduler triggers via
  the Execute API only, never direct container invocation.
- **Google Shared Drive** — the sole input channel, accessed via service
  account membership (read role). No GCS sync, no domain-wide delegation.
- **Artifact Registry** — hosts the container image for the Cloud Run Job.

## Binding Constraints (D-10.x / D-14.x)

- **D-10.4** — `pr_staging` and `pr_serving` are separate datasets; no
  shared tables, no cross-dataset writes outside the declared IAM grants.
- **D-14.1 / D-14.2** — Scheduler cadence (day/hour) and retry posture are
  deployment parameters, not business rules. They are configurable without
  touching row-level logic.
- **D-14.3** — the Cloud Run Job runs on default CPU/memory/timeout. No
  resource-tier override without an explicit ratified exception.
- **D-14.4** — Drive access is direct via the Drive API and service account
  membership. No domain-wide delegation, no forced sync to GCS.
- **D-14.5** — staging writes are atomic and gated by failure-alerting; a
  structural failure must not produce partial output or go unnoticed.

## Prohibited Alternatives

- pandas (or any other DataFrame library) as the row engine.
- INT64 ids at the BigQuery write boundary — `id_actividad` / `id_tipologia`
  are FNV-1a-64 (UInt64 in Polars) cast to STRING (decimal representation)
  before the write, to avoid signed INT64 overflow.
- Any Cloud Run Job resource override without a ratified exception to D-14.3.
- GCS as an input source for PR data.
