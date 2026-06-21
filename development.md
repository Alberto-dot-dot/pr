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

Rejection handling: when a task becomes [REJECTED], the agent must:
1. Rewrite the status to [EXECUTING].
2. Wait for Alberto's feedback.
3. Re-execute the task.

Invariant: at most one task may hold the status [EXECUTING] at any time.
The agent executes one task at a time.

## Control Gates
Completion is gated by the commit. The full commit-and-merge sequence is owned by
"Git Practices" in CLAUDE.md. A task is set to [COMPLETED] only after its work-commit
executes and the work-commit hash is recorded in Done Evidence.

## Sub-tasks

### Governance


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

### Fase 1.1 — Test Fixtures

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

### Fase 1.2 — Structural Ingestion Gates & Type Contracts

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


### Fase 1.3 — USAR Gate

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

### Fase 1.4 — Normalization & Obra Derivation

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


### Fase 1.5 — Identity Hashing

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


### Fase 1.6 — Tipología Join & Disposition

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


### Fase 1.7 — FECHA Handling

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


### Fase 1.8 — stg_matched Assembly & Row-Level Output Contract

* 1.8.1 — Define and pin stg_matched schema
  Enumerate the exact column set Polars hands to Fase 4: raw columns
  required by T3 (obra, estatus=ESTATUS C.CLOUD raw, nombre_actividad=
  ACTIVIDAD raw, tipologia raw, fecha raw), _norm columns required for
  routing/aggregation (ESTATUS_C_CLOUD_norm, obra_norm, etc.), id_actividad,
  id_tipologia, and N1..N5. run_date is excluded (stamped in Fase 4, R27).
  Status: [BACKLOG]
  Done Criteria:
  1. Every column downstream phases consume is present and typed.
  2. run_date is absent (deferred to the write boundary, Fase 4).
  3. The schema is documented as the producer-side contract for Fase 4.
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
