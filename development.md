# DEVELOPMENT

## Context

For every main task in roadmap.md there exist one or more sub-tasks (atomic tasks).
Every atomic task references a main task in roadmap.md through a foreign key.

Foreign key rule: for an atomic task ID of the form X.Y.Z, the foreign key is the
segment before the last dot (X.Y), which is the main task ID in roadmap.md.
Example: atomic task 0.3.1 references main task 0.3

To run a task, Alberto must declare the task ID before any interaction.
See the Task Protocol defined in CLAUDE.md.

## Schema

### Status enum
Every atomic task carries exactly one status from this closed set:

[BACKLOG]    not yet started.
[EXECUTING]  currently being executed by the agent.
[REVIEWING]  finished by the agent, under review by Alberto.
[REJECTED]   reviewed and rejected; must return to execution.
[COMPLETED]  reviewed, committed, and closed.
[BLOCKED]    cannot proceed due to an unresolved external dependency,
             a pending decision, or a failed prerequisite.

### Per-task fields
Status:        one value from the status enum.
Done Criteria: a list of requirements that define success. Alberto proposes the
               Done Criteria for each sub-task before any input.
Done Evidence: test result, logs, prints, and the commit hash (mandatory).
               The agent writes a concise evidence entry only after the task
               reaches [COMPLETED].

## State Machine

Legal transitions:
[BACKLOG]   -> [EXECUTING]   task starts.
[EXECUTING] -> [REVIEWING]   agent finishes and hands the task to Alberto.
[REVIEWING] -> [REJECTED]    Alberto rejects.
[REVIEWING] -> [COMPLETED]   approved path, only after the commit is executed
                             (see Control Gates).
[REJECTED]  -> [EXECUTING]   rework.
(any)       -> [BLOCKED]     a blocking condition appears.
[BLOCKED]   -> [BACKLOG]     blocker resolved; task re-queued for execution.

Rejection handling: when a task becomes [REJECTED], the agent must:
1. Rewrite the status to [EXECUTING].
2. Wait for Alberto's feedback.
3. Re-execute the task.

Frozen-phase defect handling (Fases 0-3 only): when a frozen, authoritative
atomic task (Fases 0-3) is found ill-defined during MODE: EXECUTION, the agent
sets it to [BLOCKED]. Resolution follows the "Frozen-phase defect handling"
procedure in the CLAUDE.md Task Protocol (propose a scoped amendment -> Alberto
ratifies via the Commit gate -> [BACKLOG] -> re-execute). For this sub-case the
[BLOCKED] -> [BACKLOG] transition is gated on the ratified amendment commit, not
on a bare status edit. The procedure is owned by CLAUDE.md and is not restated
here.

Invariant: at most one task may hold the status [EXECUTING] at any time.
The agent executes one task at a time.

## Control Gates
Completion is gated by the commit. The full commit-and-merge sequence is owned by
"Git Practices" in CLAUDE.md. A task is set to [COMPLETED] only after its work-commit
executes and the work-commit hash is recorded in Done Evidence.

## Sub-tasks

### FASE 0.0 — Governance Rules Authoring

* 0.0.1 — Author .claude/rules/stack.md
  Mechanical derivation from Bloque 3 (no new spec decisions): document PR's
  stack constraints and prohibited alternatives for agent consultation during
  EXECUTION.
  Status: [REVIEWING]
  Done Criteria:
  1. Declare the mandated stack: Polars as the row-level ETL engine, BigQuery
     (staging + serving), Cloud Run Job + Cloud Scheduler (autonomous weekly
     runtime), Google Shared Drive as the sole input channel (service account
     membership, no GCS sync), Artifact Registry for the container image.
  2. Enumerate the D-10.x / D-14.x constraints that bind stack choice: D-10.4
     (staging/serving dataset separation), D-14.1/D-14.2 (Scheduler params,
     not business rules), D-14.3 (no resource-tier override on the Cloud Run
     Job), D-14.4 (no domain-wide delegation, no forced GCS sync), D-14.5
     (atomic write / failure-alerting requirement).
  3. List the prohibited alternatives explicitly: pandas as the row engine,
     INT64 (vs. STRING-cast) ids at the BigQuery write boundary, any Cloud Run
     Job resource override absent an explicit ratified exception, GCS as an
     input source.
  4. Exclude executable code and any new rule content beyond what Bloque 3
     already settled — the file holds reference material only.
  Done Evidence:

### FASE 0 - Local Repo & Scaffolding
#### 0.1 - Enviroment

* 0.1.1 — Pin Python interpreter version
Create .python-version declaring the target interpreter version for local development, matching what the eventual Cloud Run Job container will use.
Status: [BACKLOG]
Done Criteria:
.python-version exists at repo root with a single version string.
Done Evidence:

* 0.1.2 — Create package structure (src-layout)
Create src/pr/__init__.py. Establishes the importable package skeleton; src-layout forces explicit installation rather than accidental path-based imports.
Status: [BACKLOG]
Done Criteria:
src/pr/__init__.py exists.
Done Evidence:


* 0.1.3 — Create tests directory
Create empty tests/ directory. Fixtures and actual test files are out of scope — they belong to Fase 1.
Status: [BACKLOG]
Done Criteria:
tests/ exists, no content required.
Done Evidence:


* 0.1.4 — Declare build backend
Create minimal pyproject.toml with [build-system] and [project] tables. Enables editable install of pr without adopting Poetry.
Status: [BACKLOG]
Done Criteria:
pyproject.toml exists with [build-system] and [project] tables populated.
pip install -e . succeeds with no error.
Done Evidence:


* 0.1.5 — Declare third-party dependencies
Create requirements.txt declaring polars, exact-pinned (==), no version ranges.
Status: [BACKLOG]
Done Criteria:
requirements.txt exists, contains at least polars==<version>.
No >=, ~=, or unpinned entries present.
Done Evidence:

* 0.1.6 — Create virtual environment and install
Create a clean local virtual environment. Run pip install -r requirements.txt followed by pip install -e ..
Status: [BACKLOG]
Done Criteria:
Both install commands complete with exit code 0.
Done Evidence:


* 0.1.7 — Smoke-test imports
With the environment active, confirm both polars and pr import without error.
Status: [BACKLOG]
Done Criteria:
python -c "import polars; import pr" exits 0, prints polars.__version__.
Done Evidence:


* 0.1.8 — Configure .gitignore
Create .gitignore excluding the virtual environment directory, __pycache__/, and .env.
Status: [BACKLOG]
Done Criteria:
.gitignore excludes venv dir, __pycache__/, .env.
Done Evidence:


* 0.1.9 — Write README
Create README.md referencing estado-spec-pr.md as the canonical spec, without duplicating its content.
Status: [BACKLOG]
Done Criteria:
README.md exists, links/references estado-spec-pr.md, no duplicated rule content.
Done Evidence:


* 0.1.10 — Configure recommended VS Code extensions
Create .vscode/extensions.json recommending ms-python.python and ms-python.vscode-pylance only. No GCP-related extensions yet.
Status: [BACKLOG]
Done Criteria:
.vscode/extensions.json lists exactly these two extension IDs.
Done Evidence:

* 0.1.11 — Commit scaffolding
Single git commit covering 0.1.1–0.1.10. Working tree clean afterward.
Status: [BACKLOG]
Done Criteria:
One commit contains all scaffolding files.
git status clean after commit.
Done Evidence:

### FASE 1 — Polars Core Pipeline (local)
#### 1.1 — Test Fixtures

* 1.1.1 — Build valid PR file fixture
  Construct one structurally-correct .xlsx PR fixture: rows 1–5 and cols
  A–S as noise, row 6 header matching the declared raw schema exactly from
  col T, representative data rows. This is the happy-path substrate for
  every downstream gate.
  Status: [BACKLOG]
  Done Criteria:
  1. Fixture is a binary .xlsx, openable by the Polars reader.
  2. Rows 1–5 and cols A–S contain arbitrary noise; data origin is row 6 / col T.
  3. Header row 6 from col T matches the Section 3 raw schema string-for-string.
  4. Data rows include at least one USAR="SI" and one USAR="NO".
  Done Evidence:

* 1.1.2 — Build corrupt PR file fixtures (gate triggers)
  Construct the negative fixtures that must trip R16/R18: wrong extension,
  corrupt/unreadable file, header with an extra/missing/reordered/cosmetic
  variant column.
  Status: [BACKLOG]
  Done Criteria:
  1. One non-.xlsx fixture (R16 trigger).
  2. One .xlsx with a header discrepancy of each class: extra, missing,
     reordered, casing/whitespace variant (R18 triggers).
  3. Each fixture is labelled with the gate and discrepancy it targets.
  Done Evidence:

* 1.1.3 — Build mapeo fixtures
  Construct a valid mapeo fixture [N1..N5, Tipologia, Obra] plus an
  empty-Tipologia fixture (R6.1 trigger) and a niveles set with no PR
  counterpart (R8 NO_MATCH trigger).
  Status: [BACKLOG]
  Done Criteria:
  1. Valid mapeo joins cleanly against 1.1.1 on (Obra, N1..N5).
  2. One mapeo row with empty Tipologia (R6.1).
  3. PR fixture contains at least one niveles tuple absent from the mapeo (R8).
  Done Evidence:

* 1.1.4 — Build nombre_programa and expected-output fixtures
  Provide nombre_programa strings (with "_" and without, for R3/R3.1) and
  the expected stg_matched / recon_nomatch / anomaly rows for the valid
  inputs, to assert pipeline correctness.
  Status: [BACKLOG]
  Done Criteria:
  1. One nombre_programa with "_" (valid obra parse) and one without (R3.1).
  2. Expected stg_matched rows enumerated for the valid PR fixture.
  3. Expected recon_nomatch and anomaly-queue rows enumerated.
  Done Evidence:


#### 1.2 — Structural Ingestion Gates & Type Contracts

* 1.2.1 — R16 file format gate
  Reject any file that is not a valid binary .xlsx (wrong extension,
  corrupt, unreadable). On failure, abort the full multiobra run, produce
  no output for any obra, report the failure. Run does not resume until
  every batch file is valid .xlsx.
  Status: [BACKLOG]
  Done Criteria:
  1. A non-.xlsx fixture (1.1.2) triggers a full-run abort, no partial output.
  2. The failing file is reported.
  3. A valid .xlsx fixture (1.1.1) passes the gate.
  Done Evidence:

* 1.2.2 — R17 structural noise discard
  On a file that passed R16, unconditionally discard rows 1–5 and cols
  A–S without inspecting their content; set row 6 as header and col T as
  data origin (offset 0).
  Status: [BACKLOG]
  Done Criteria:
  1. Rows 1–5 and cols A–S are dropped without validation.
  2. Row 6 is established as header, col T as offset 0.
  3. First parsed data column aligns with the raw-schema first entry (USAR).
  Done Evidence:

* 1.2.3 — R18 literal header validation gate
  Compare row 6 from col T onward against the full declared raw schema:
  exact string match, case- and whitespace-sensitive, fixed positional
  order, no 5.1 normalization applied. Any discrepancy (extra, missing,
  reordered, cosmetic) aborts the full multiobra run.
  Status: [BACKLOG]
  Done Criteria:
  1. Each discrepancy-class fixture (1.1.2) triggers a full-run abort.
  2. The log records obra, file, and the specific header in discrepancy.
  3. The valid fixture passes; no normalization is applied to header text.
  Done Evidence:

* 1.2.4 — R13 ESTATUS C.CLOUD type contract
  Force-cast the entire ESTATUS C.CLOUD column to Utf8 before any 5.1 step,
  preserving null as null (never the literal string "null").
  Status: [BACKLOG]
  Done Criteria:
  1. Column is Utf8 after this step regardless of inbound underlying type.
  2. null values remain null, not "null".
  3. The documented assumption (numeric value, if present, is always literal
     "0") is asserted/guarded; a non-"0" numeric is surfaced, not silently cast.
  Done Evidence:


#### 1.3 — USAR Gate

* 1.3.1 — Implement USAR eviction gate
  Evict every row whose raw USAR ≠ "SI", positioned upstream of 5.1, R1,
  R7 and the routing rules. Comparison against raw value; no USAR_norm.
  Status: [BACKLOG]
  Done Criteria:
  1. Rows with raw USAR ≠ "SI" are removed before any normalization step runs.
  2. No USAR_norm column is generated.
  3. No count, log, or reconciliation artifact is produced for evicted rows.
  Done Evidence:

* 1.3.2 — Verify gate ordering against fixtures
  Assert against 1.1.1 that surviving rows are exactly USAR="SI" and that
  downstream stages never observe a non-"SI" row.
  Status: [BACKLOG]
  Done Criteria:
  1. On the valid fixture, post-gate row count equals the USAR="SI" count.
  2. A trace confirms 5.1/R1/R7 receive only USAR="SI" rows.
  3. Accepted-risk note recorded: casing/whitespace variants of "SI" are
     silently dropped (R14 closed-domain assumption).
  Done Evidence:

#### 1.4 — Normalization & Obra Derivation

* 1.4.1 — R3 obra parse
  Extract obra = substring before the first "_" in nombre_programa and
  embed it as a column obra on every row of that PR.
  Status: [BACKLOG]
  Done Criteria:
  1. For a nombre_programa with "_", obra equals the substring before it.
  2. The obra column is present on all rows of the file.
  3. The raw nombre_programa is preserved.
  Done Evidence:

* 1.4.2 — R3.1 no-separator guard
  A nombre_programa with no "_" routes to the anomaly queue; the system
  must NOT assign obra and must NOT take the full string as obra.
  Status: [BACKLOG]
  Done Criteria:
  1. The no-"_" fixture (1.1.4) is routed to the anomaly queue.
  2. obra is not assigned for that row/file.
  3. The full string is never used as a fallback obra.
  Done Evidence:

* 1.4.3 — 5.1 normalization pipeline
  Implement the 10-step ordered pipeline (order is normative, not
  reorderable), parallel and non-destructive: raw column preserved with its
  type, <col>_norm produced. Surface an emptiness flag after step 8 for
  downstream disposition.
  Status: [BACKLOG]
  Done Criteria:
  1. All 10 steps execute in declared order on a known input → known output.
  2. Raw column is untouched; <col>_norm is the only addition.
  3. An empty-after-pipeline flag is produced for consumption by R2 (1.5.2)
     for identity columns and by each column's downstream rule otherwise.
  Done Evidence:

* 1.4.4 — R5 parallel _norm generation
  For every column participating in identity, match, join, or filter,
  generate <col>_norm via 5.1, including obra_norm from the raw obra column
  (1.4.1). USAR is the explicit exception (no _norm, per R14).
  Status: [BACKLOG]
  Done Criteria:
  1. _norm columns exist for actividad, obra, N1..N5, tipologia, ESTATUS C.CLOUD.
  2. obra_norm derives from the raw obra column, not from nombre_programa directly.
  3. No USAR_norm is generated.
  Done Evidence:


#### 1.5 — Identity Hashing

* 1.5.1 — R1 id_actividad hashing
  Assign id_actividad = FNV-1a-64(actividad_norm), output UInt64, for rows
  with non-empty actividad_norm.
  Status: [BACKLOG]
  Done Criteria:
  1. id_actividad is deterministic FNV-1a-64 over actividad_norm.
  2. Output type is UInt64.
  3. A known actividad_norm yields a stable, reproducible hash.
  Done Evidence:

* 1.5.2 — R2 empty actividad disposition
  A row whose actividad_norm is empty (flag from 1.4.3) routes to the
  anomaly queue; id_actividad is NOT generated for it.
  Status: [BACKLOG]
  Done Criteria:
  1. Empty-actividad fixture row is routed to the anomaly queue.
  2. No id_actividad is generated for that row.
  3. Pipeline continues processing remaining rows.
  Done Evidence:

* 1.5.3 — R6 id_tipologia hashing (mapeo)
  Assign id_tipologia = FNV-1a-64(obra_norm + "|" + tipologia_norm) on the
  mapeo, fixed operand order (obra_norm then tipologia_norm, not commutable),
  "|" separator, output UInt64.
  Status: [BACKLOG]
  Done Criteria:
  1. Operand order is fixed; swapping operands is rejected/not implemented.
  2. Separator is "|"; output type is UInt64.
  3. Hashing method is identical in form to R1 (single way to hash).
  Done Evidence:

* 1.5.4 — R6.1 mapeo fail-fast
  A mapeo row with tipologia_norm empty after 5.1 halts/rejects ingestion
  of that mapeo and reports the row; no id_tipologia generated, join not
  continued. Distinct from R2 (file-level reject, not row-level routing).
  Status: [BACKLOG]
  Done Criteria:
  1. Empty-Tipologia mapeo fixture (1.1.3) halts that mapeo's ingestion.
  2. The offending row is reported; no id_tipologia produced.
  3. The join (R7) does not proceed for that mapeo.
  Done Evidence:


#### 1.6 — Tipología Join & Disposition

* 1.6.1 — R7 tipología join
  Join PR against the mapeo on (obra_norm, N1_norm..N5_norm) over _norm
  columns in both tables, inheriting tipologia and id_tipologia. The mapeo's
  obra_norm derives from its raw Obra column via R5 (R3 does not apply to
  the mapeo).
  Status: [BACKLOG]
  Done Criteria:
  1. Join key is the six _norm columns on both sides.
  2. Matched rows inherit tipologia and id_tipologia.
  3. Mapeo obra_norm comes from its Obra column, not from a filename parse.
  Done Evidence:

* 1.6.2 — R8 no-match disposition
  No-match rows get id_tipologia=null, tipologia_status=NO_MATCH, are
  excluded from the main frame, and are recorded to recon_nomatch (at least
  the distinct obra+N1..N5 tuples). Matched rows carry tipologia_status=
  MATCHED. tipologia_status is internal routing only — absent from any
  output schema. Pipeline does not halt.
  Status: [BACKLOG]
  Done Criteria:
  1. NO_MATCH fixture rows are excluded from the main frame and written to
     recon_nomatch with at least distinct obra+N1..N5 tuples.
  2. Matched rows carry MATCHED; only MATCHED proceed downstream.
  3. tipologia_status appears in no output schema; the run does not halt.
  Done Evidence:

#### 1.7 — FECHA Handling

* 1.7.1 — R22 active FECHA validation
  A row surviving R14 whose FECHA is null, blank, or does not cast cleanly
  to date routes to the anomaly queue; no fecha value generated; the run
  is not aborted. FECHA is a date field — not in 5.1, no _norm.
  Status: [BACKLOG]
  Done Criteria:
  1. Malformed-FECHA fixture (1.1.4) routes to the anomaly queue.
  2. No fecha value is produced for that row; remaining rows process normally.
  3. FECHA generates no _norm column and skips the 5.1 pipeline.
  Done Evidence:

* 1.7.2 — R19/R20 raw fecha preservation
  Preserve the raw FECHA value untransformed at original row grain in the
  output frame, available for downstream T3 passthrough (R20) and T2 grain
  (R4-T2, Fase 4). R19 semantics (real activity start date, invariant to
  estatus/routing) documented as the field contract; no transformation here.
  Status: [BACKLOG]
  Done Criteria:
  1. Raw fecha column is present in the output frame at row grain, untransformed.
  2. The R19 invariance contract is documented against the column.
  3. No aggregation or estatus-dependent reshaping is applied in Fase 1.
  Done Evidence:


#### 1.8 — stg_matched Assembly & Row-Level Output Contract

* 1.8.1 — Define and pin stg_matched local producer schema
  The Fase 1 Polars frame — the LOCAL, fixture-validated face of stg_matched,
  before the Fase 3 write boundary. Nine columns, Arrow types:
    obra (Utf8), obra_norm (Utf8), estatus_c_cloud (Utf8),
    estatus_c_cloud_norm (Utf8), nombre_actividad (Utf8),
    id_actividad (UInt64), tipologia (Utf8), id_tipologia (UInt64),
    fecha (Date).
  Surviving _norm columns are exactly obra_norm and estatus_c_cloud_norm
  (BigQuery still evaluates these: GROUP BY key, R9/R10 predicate). N1..N5
  (raw and _norm), actividad_norm, tipologia_norm are NOT present — consumed
  upstream (join R7, hashes R1/R6), no downstream reader (S19).
  run_date and the STRING id representation are NOT in this frame: both are
  Fase 3 write-boundary transforms (3.4.3, R27/D-14.5). ids are UInt64 here.
  Status: [BACKLOG]
  Done Criteria:
  1. The 9-column local frame is documented exactly as above, Arrow types.
  2. Surviving _norm set = {obra_norm, estatus_c_cloud_norm} only; N1..N5,
     actividad_norm, tipologia_norm explicitly absent.
  3. run_date absent from the local frame; documented as the Fase 3
     write-boundary injection (3.4.3), not Fase 1, not Fase 4.
  4. ids are UInt64; STRING cast documented as a Fase 3 write-boundary transform.
  5. This 9-column frame is the producer contract that the Fase 3 write boundary
     converts to the 10-column BigQuery stg_matched table consumed by Fase 4.
  Done Evidence:

* 1.8.2 — Assemble MATCHED output frame
  Produce stg_matched as the MATCHED-only, USAR-gated, normalized frame
  conforming to 1.8.1, validated end-to-end against the valid fixture set.
  Status: [BACKLOG]
  Done Criteria:
  1. stg_matched contains only MATCHED, USAR="SI" rows.
  2. The assembled frame matches the expected stg_matched fixture (1.1.4).
  3. Frame schema is exactly 1.8.1, no extra/missing columns.
  Done Evidence:

* 1.8.3 — Assemble recon_nomatch and anomaly-queue stub
  Produce recon_nomatch per R8 and a stub artifact collecting the R2/R6.1/
  R22 anomaly rows. The anomaly artifact's fine schema is deferred — Fase 1
  asserts routing into it, not its final shape.
  Status: [BACKLOG]
  Done Criteria:
  1. recon_nomatch carries at least distinct obra+N1..N5 NO_MATCH tuples.
  2. R2, R6.1, and R22 routed rows land in the anomaly stub.
  3. The anomaly artifact's fine schema is explicitly marked deferred.
  Done Evidence:

### FASE 2 — GCP Infrastructure Provisioning
#### 2.1 — Aprovisionamiento base y Despliegue del Cloud Run Job

* 2.1.1 — Habilitar APIs de GCP requeridas
  Habilitar Cloud Run Admin API, Cloud Scheduler API, BigQuery API, Artifact Registry API y Google Drive API en el proyecto target. Precondición absoluta de la fase.
  Status: [BACKLOG]
  Done Criteria:
  1. Las cinco APIs (Run, Scheduler, BQ, Artifact Registry, Drive) aparecen como habilitadas en el proyecto.
  Done Evidence:

* 2.1.2 — Provisión de Artifact Registry
  Crear el repositorio físico en Artifact Registry para alojar la imagen del pipeline, requisito bloqueante antes de instanciar el Job.
  Status: [BACKLOG]
  Done Criteria:
  1. Repositorio con formato Docker creado en Artifact Registry en la región target.
  Done Evidence:

* 2.1.3 — Crear el Cloud Run Job
  Crear el Job (tipo Jobs, no Service) en el proyecto/región target. Subir una imagen placeholder/stub al repositorio de 2.1.2 para permitir la creación física del recurso.
  Status: [BACKLOG]
  Done Criteria:
  1. Imagen placeholder subida al Artifact Registry.
  2. Job existe, tipo confirmado run-to-completion, sin trigger HTTP.
  Done Evidence:

* 2.1.4 — Configurar recursos del Job sin override (D-14.3)
  Documentar explícitamente que CPU/memoria/timeout permanecen en sus valores default de Cloud Run Jobs, con referencia a D-14.3.
  Status: [BACKLOG]
  Done Criteria:
  1. Manifest/config documenta "sin override; default suficiente," con referencia explícita a D-14.3.
  2. Ningún tier de recursos elevado fue solicitado.
  Done Evidence:

* 2.1.5 — Configurar observabilidad y alertas del Job
  Añadir telemetría básica (vía Cloud Logging alerts o Log Metrics) para evitar el punto ciego del Scheduler ante fallos internos del Job.
  Status: [BACKLOG]
  Done Criteria:
  1. Canal de alerta/métrica configurado para capturar y notificar fallos internos durante la ejecución del contenedor.
  Done Evidence:

* 2.1.6 — Smoke test manual de ejecución
  Ejecutar el Job manualmente utilizando la imagen placeholder.
  Status: [BACKLOG]
  Done Criteria:
  1. Ejecución completa sin error de infraestructura.
  Done Evidence:

#### 2.2 — Configuración del disparo programado

* 2.2.1 — Crear el Cloud Scheduler job
  Apuntar al endpoint de la Execute API del Job de 2.1.3, no a invocación HTTP directa al contenedor.
  Status: [BACKLOG]
  Done Criteria:
  1. Scheduler job creado; target = Execute API del Job correcto.
  Done Evidence:

* 2.2.2 — Configurar cadencia de disparo
  Cron expression para la cadencia indicativa (lunes 01:00 AM), declarada como parámetro de despliegue (D-14.2).
  Status: [BACKLOG]
  Done Criteria:
  1. Cron configurado; documentado explícitamente como parámetro, no regla de negocio.
  Done Evidence:

* 2.2.3 — Identidad de invocación del Scheduler (least-privilege)
  Crear/asignar una SA de invocación distinta de la SA runtime del Job. Asignar el permiso exacto run.jobs.run scoped exclusivamente al recurso del Job.
  Status: [BACKLOG]
  Done Criteria:
  1. SA de invocación distinta de la SA del Job confirmada.
  2. Permiso run.jobs.run garantizado a la SA y scoped al Job específico, sin permisos a nivel de proyecto ni acceso HTTP genérico.
  Done Evidence:

#### 2.3 — Autenticación de la Service Account contra Shared Drive

* 2.3.1 — Crear/identificar la service account del Job
  Identidad runtime que utilizará el pipeline para la lectura en Drive y escritura en BQ.
  Status: [BACKLOG]
  Done Criteria:
  1. SA existe y está asociada como identidad runtime del Job de 2.1.3.
  Done Evidence:

* 2.3.2 — Agregar la SA como miembro de la Shared Drive
  Rol de lectura, sin domain-wide delegation (D-14.4).
  Status: [BACKLOG]
  Done Criteria:
  1. SA aparece como miembro directo de la Shared Drive con rol de lectura.
  2. Domain-wide delegation explícitamente NO configurada.
  Done Evidence:

* 2.3.3 — Confirmar ausencia de sincronización forzada a GCS
  Verificar que el acceso es directo vía Drive API (D-14.4).
  Status: [BACKLOG]
  Done Criteria:
  1. Ningún mecanismo de sync automático a GCS está configurado.
  Done Evidence:

#### 2.4 — Provisión de datasets BigQuery (container-level, D-10.4)

* 2.4.1 — Crear dataset de staging oculto (pr_staging)
  Status: [BACKLOG]
  Done Criteria:
  1. Dataset pr_staging existe, ubicación/proyecto correctos.
  Done Evidence:

* 2.4.2 — Crear dataset de servicio (pr_serving)
  Status: [BACKLOG]
  Done Criteria:
  1. Dataset pr_serving existe, separado físicamente de pr_staging.
  Done Evidence:

* 2.4.3 — Aplicar IAM base sobre ambos datasets
  SA del Job: rol de escritura exclusivamente sobre pr_staging. Ningún otro principal con acceso en esta fase.
  Status: [BACKLOG]
  Done Criteria:
  1. SA del Job tiene write sobre pr_staging únicamente.
  2. pr_serving no tiene ningún grant más allá de admin del proyecto.
  Done Evidence:

* 2.4.4 — Documentar la frontera container/contenido Fase2/Fase4
  Registrar explícitamente que este Paso no crea tablas, vistas, ni el link de authorized views (D-10.4, propiedad de Fase 4).
  Status: [BACKLOG]
  Done Criteria:
  1. Nota de frontera escrita y referenciada desde el roadmap de Fase 4.
  Done Evidence:

#### 2.5 — Verificación de aprovisionamiento y contrato de salida

* 2.5.1 — Prueba real de lectura de Shared Drive vía la SA
  Listar el contenido de una carpeta de prueba genérica, validando que la Drive API responde correctamente.
  Status: [BACKLOG]
  Done Criteria:
  1. La SA lista contenido real de una carpeta vía Drive API.
  2. Explícitamente fuera de scope: cualquier lógica de R31/R32.
  Done Evidence:

* 2.5.2 — Verificar configuración física de ambos datasets
  Status: [BACKLOG]
  Done Criteria:
  1. Ubicación, nombre, y proyecto de pr_staging y pr_serving verificados.
  Done Evidence:

* 2.5.3 — Smoke test de la cadena completa Scheduler → Job
  Disparo manual del Scheduler para confirmar la resolución exitosa del flujo y los permisos ajustados en 2.2.3.
  Status: [BACKLOG]
  Done Criteria:
  1. Scheduler invoca el Job exitosamente vía la Execute API sin error de permisos.
  Done Evidence:

* 2.5.4 — Declarar el contrato de salida de Fase 2
  Por escrito: la única precondición que Fase 3 puede asumir cerrada es "SA con lectura confirmada sobre Shared Drive."
  Status: [BACKLOG]
  Done Criteria:
  1. Contrato de salida escrito y referenciado desde la entrada de Fase 3.
  Done Evidence:

### FASE 3 — Live Drive File Resolution
#### 3.1 — Production Image Build & Job Redeployment

* 3.1.1 — Package Fase 1 pipeline into production image
Build a container image from src/pr/ using Fase 0's pinned requirements.txt.
No GCP interaction yet — local/CI build only.
Status: [BACKLOG]
Done Criteria:
1. Image builds from src/pr/ against the exact pins in requirements.txt (== only).
2. Image entrypoint invokes the Fase 1 pipeline as a run-to-completion job (no HTTP server).
3. Build is reproducible: same requirements.txt produces the same dependency set.
4. No GCP credentials or live Drive access required to build.
Done Evidence:

* 3.1.2 — Push production image to Artifact Registry
Replace the 2.1.3 placeholder tag with the production image.
Status: [BACKLOG]
Done Criteria:
1. Image pushed to the Artifact Registry repo provisioned in 2.1.2.
2. The 2.1.3 placeholder tag is superseded — the Job's referenced tag now resolves to the production image.
3. Push uses the Job's deploy identity; no domain-wide credentials introduced.
Done Evidence:

* 3.1.3 — Redeploy Cloud Run Job against production image
Status: [BACKLOG]
Done Criteria:
1. Cloud Run Job updated to reference the production image tag from 3.1.2.
2. Resource bounds unchanged from 2.1.4 (no override — D-14.3 holds).
3. Scheduler binding from 2.2 intact after redeploy (run.jobs.run identity unaffected).
Done Evidence:

* 3.1.4 — Smoke test against real image
Re-run the 2.1.6 smoke test against the production image. Confirms runtime health
only — NOT live Drive resolution (that is 3.4).
Status: [BACKLOG]
Done Criteria:
1. Job executes to completion against the production image without runtime/import error.
2. Test asserts the pipeline reaches its first Drive-read boundary and halts cleanly when no live resolution is wired yet (expected controlled stop, not a crash).
3. No write to pr_staging occurs in this test.
Done Evidence:


#### Paso 3.2 — Precondition Gate: Mapeo-Folder Accessibility (R31) ∥ PR-Vigente Resolution (R23–R26)

> Retry posture (proposed, reversible): retry count (3) and inter-attempt delay are
> DEPLOYMENT PARAMETERS, not hardcoded constants — homologous to D-14.2's treatment of
> Scheduler day/hour. Retry coverage extends to 3.2.1, 3.2.2 AND 3.2.3. Override either
> by request; both are one-line changes.
>
> Transient vs. definitive distinction (normative for all of 3.2): transient =
> connection-class error (timeout, rate-limit, 5xx) → retry up to 3 attempts total.
> Definitive = clean negative result (folder absent, permission denied, zero matches,
> tie) → abort on first response, NO retry. Retry sits BEFORE the abort signal; it does
> not amend R25/R31's abort conditions.

* 3.2.1 — R31: mapeo-folder accessibility gate, with retry
Verify Tipologias Obras is reachable via the Drive API.
Status: [BACKLOG]
Done Criteria:
1. Verifies the declared global folder Tipologias Obras is reachable via the Drive API using the Fase 2 service account identity.
2. Transient errors (timeout, rate-limit, 5xx) retried up to 3 attempts total, with an inter-attempt delay; retry params read from deployment config.
3. Definitive negative (folder absent, permission denied) aborts on first response — no retry.
4. On exhausted retries OR definitive negative: raises global abort, produces no output for any obra, logs declared path + error type + failure class (transient-exhausted vs. definitive).
5. This check is independent of the R23–R26 track (no shared folder/data); it does NOT block R23–R26 from running.
Done Evidence:

* 3.2.2 — R23–R24: recursive traversal & date extraction, with retry
Status: [BACKLOG]
Done Criteria:
1. For each (ruta_archivo, nombre_programa) fila in the programas_vigentes sheet, recursively traverses the tree under ruta_archivo.
2. Identifies folders matching <PREFIJO>_SEM_<AAAAMMDD>, extracts the encoded date from the folder name (never mtime).
3. Non-conforming folders (R24) are ignored as date candidates without aborting; traversal continues.
4. Transient Drive API errors during traversal retried up to 3 attempts total, inter-attempt delay from deployment config.
5. On exhausted retries: raises global abort, logs the specific fila + failure class.
Done Evidence:

* 3.2.3 — R25–R26: max-date selection & match resolution, with retry
Status: [BACKLOG]
Done Criteria:
1. Within date-conforming folders, matches nombre_programa exactly (case/space-sensitive, homologous to R18); selects the file under the maximum encoded date as vigente.
2. Zero matches anywhere in the tree → definitive abort on first conclusive result, no retry (R25).
3. More than one match under the max date → definitive abort, no retry; mtime explicitly NOT used as tiebreaker (R26).
4. Transient Drive API errors during this resolution retried up to 3 attempts total, inter-attempt delay from deployment config.
5. On exhausted retries (transient) OR definitive zero/tie: raises global abort, produces no output for any obra, logs the specific fila + failure class.
Done Evidence:

* 3.2.4 — Concurrency wrapper (OR-abort)
Status: [BACKLOG]
Done Criteria:
1. Runs the R31 track (3.2.1) and the R23–R26 track (3.2.2 + 3.2.3 across all declared filas) concurrently.
2. The first definitive failure on EITHER track halts the entire batch immediately (OR-abort) — the other track's in-flight work is not awaited for completion before abort.
3. A transient retry in progress on one track does not by itself abort the run; only an exhausted-retry or definitive failure does.
4. Abort signal carries the originating track + failure class for the log.
Done Evidence:

* 3.2.5 — AND-success barrier
Status: [BACKLOG]
Done Criteria:
1. Proceeds to Paso 3.3 only if BOTH conditions hold: Tipologias Obras confirmed accessible (3.2.1) AND every declared fila resolved to exactly one vigente PR file (3.2.2 + 3.2.3).
2. If either condition is unmet, the run has already aborted via 3.2.4 — the barrier is never reached in a partial-success state.
3. No Polars row-level processing (5.1, R14, R1/R6, R7) executes before this barrier lifts.
Done Evidence:

#### Paso 3.3 — Per-Obra Mapeo Presence Resolution (R32)

* 3.3.1 — R32: normalized per-obra mapeo filename match
Status: [BACKLOG]
Done Criteria:
1. For each obra whose obra_norm was derived (R3+R5), searches Tipologias Obras for a file whose normalized name (pipeline 5.1) equals {obra_norm}_mapeo_tipologias.
2. Match is executed on the NORMALIZED filename — deliberate divergence from R18's literal posture, per S15 (filename is free-typed by the analyst).
3. Produces exactly one of three outcomes per obra: single-match, zero-match, ambiguous-match (>1).
Done Evidence:

* 3.3.2 — R32: zero/ambiguous handling, scoped exclusion
Status: [BACKLOG]
Done Criteria:
1. Zero-match AND ambiguous-match receive identical treatment: exclude ALL macro_partidas of that obra from the current run.
2. The run does NOT abort — remaining obras continue processing (scoped exclusion, deliberate divergence from R25/R26's global abort).
3. Logs the excluded obra + reason (zero vs. ambiguous) to the Job execution log.
4. No persisted exception artifact is produced — the execution log is the sole record (per R32).
Done Evidence:

* 3.3.3 — Handoff of resolved mapeo to Fase 1 join pipeline
Status: [BACKLOG]
Done Criteria:
1. For single-match obras, the resolved mapeo file path is handed to Fase 1's existing R6/R6.1/R7 logic.
2. No new validation logic is introduced here — R6 (id_tipologia), R6.1 (empty-tipologia reject), R7 (join) are unchanged by Fase 3 (per R32 closing clause).
3. Excluded obras (3.3.2) are absent from the set handed to the join.
Done Evidence:

#### Paso 3.4 — Live Resolution Verification & Exit Contract toward Fase 4
* 3.4.1 — Single-obra live resolution cycle
Status: [BACKLOG]
Done Criteria:
1. Runs one full live cycle against the real Shared Drive for at least one real obra with known-good PR and mapeo files.
2. Resolution succeeds through the 3.2 gate and 3.3 per-obra match.
3. Uses the Fase 2 service account end-to-end; no developer credentials.
Done Evidence:

* 3.4.2 — Fixture/live parity confirmation
Status: [BACKLOG]
Done Criteria:
1. The Fase 1 pipeline (built/tested only against fixtures) produces identical row-level behavior when fed live-resolved file paths.
2. Any divergence between fixture and live behavior is treated as a blocking defect, not an accepted variance.
Done Evidence:

* 3.4.3 — Real staging write with run_date and BigQuery type mapping
Status: [BACKLOG]
Done Criteria:
1. A real WRITE_TRUNCATE write to stg_matched and recon_nomatch in pr_staging occurs from live-resolved data.
2. run_date is stamped on every row of both tables, single value for the run (R27).
3. id_actividad and id_tipologia are cast from UInt64 to STRING (decimal representation) in both tables' BigQuery target schema, avoiding INT64 signed overflow — ~50% of FNV-1a-64 hashes exceed INT64 max (S19).
4. The BigQuery stg_matched target schema is exactly the 10-column contract: obra STRING, obra_norm STRING, estatus_c_cloud STRING, estatus_c_cloud_norm STRING, nombre_actividad STRING, id_actividad STRING, tipologia STRING, id_tipologia STRING, fecha DATE, run_date DATE.
5. Write is atomic per D-14.5: it executes only because no abort gate fired in this cycle.
Done Evidence:

* 3.4.4 — Abort/exclusion path injection
Status: [BACKLOG]
Done Criteria:
1. Against DISPOSABLE test artifacts (never production data), each failure path is injected and verified:
   a. mapeo folder inaccessible → global abort (R31).
   b. PR vigente zero match / tie → global abort (R25/R26).
   c. per-obra mapeo zero / ambiguous → scoped exclusion, run continues (R32).
2. Transient-error retry is exercised: a simulated transient error recovers within 3 attempts and does NOT abort; an exhausted transient DOES abort.
3. Each path's log output carries the correct failure class.
Done Evidence:

* 3.4.5 — Exit contract toward Fase 4
Status: [BACKLOG]
Done Criteria:
1. Documents that at Fase 3 close, stg_matched / recon_nomatch carry live, validated data.
2. Confirms Fase 4 (aggregation/views) has no residual dependency on Fase 3 resolution logic — it consumes the staging tables only.
3. The two-track gate behavior and the per-obra exclusion are recorded as verified, not assumed.
Done Evidence:


