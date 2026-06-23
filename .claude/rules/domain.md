# Domain Rules — PR

Reference material only. Derived mechanically from Bloque 2 of the closed
specification (estado-spec-pr.md, S21). No new decisions are made here.

## Glossary

- **obra** — the construction project unit. Parsed from `nombre_programa`
  as the substring before the first `"_"` (R3); a no-`"_"` filename routes
  to the anomaly queue instead (R3.1) — the full string is never taken as a
  fallback obra. Present as a normalized join key, `obra_norm`, throughout
  the pipeline (R5).

- **macropartida** — the per-obra scheduling grouping. Excluded wholesale
  for an obra when its mapeo file resolves to zero or ambiguous matches
  (R32) — a scoped exclusion, not a global abort; the run continues for
  other obras.

- **tipologia** — the work-type classification. Resolved via the mapeo join
  (R7) keyed on `(obra_norm, N1_norm..N5_norm)`; its identity, `id_tipologia`,
  is hashed from the mapeo side (R6: `FNV-1a-64(obra_norm + "|" + tipologia_norm)`,
  fixed operand order). A mapeo row with empty `tipologia_norm` halts that
  mapeo's ingestion entirely (R6.1, fail-fast, distinct from R2's row-level
  routing). A PR-side tuple with no mapeo match gets `id_tipologia=null`,
  is excluded from the main frame, and is recorded to `recon_nomatch` (R8) —
  the negative-outcome state that completes this term's definition.

- **N1–N5** — the five-level hierarchical breakdown columns. Used as
  join/identity keys, normalized to `N1_norm..N5_norm`, between the PR file
  and the mapeo (R7). Consumed entirely by the join; not present in the
  local `stg_matched` output frame (1.8.1).

- **estatus / ESTATUS C.CLOUD** — the raw status column. Force-cast to
  `Utf8` before any normalization, preserving null as null, never the
  literal string `"null"` (R13). `estatus_c_cloud_norm` survives into
  `stg_matched` as a GROUP BY key / R9-R10 predicate — it is not consumed
  upstream like the other `_norm` columns.

- **USAR** — the raw inclusion flag. Rows where raw `USAR ≠ "SI"` are
  evicted upstream of all normalization and routing (R14); no `USAR_norm`
  is ever generated — an explicit exception to the general `_norm` rule
  (R5). Closed-domain assumption: casing/whitespace variants of `"SI"` are
  silently dropped, not coerced.

## Row-Level Rule Index (by ID, executable definition in development.md)

- **R3 / R3.1** — obra parse from `nombre_programa`; no-separator guard.
- **R5** — parallel `_norm` generation, including `obra_norm` derivation and
  the `USAR` no-norm exception.
- **R6 / R6.1** — `id_tipologia` hashing (mapeo); mapeo fail-fast on empty
  `tipologia_norm`.
- **R7** — tipología join on the six `_norm` columns.
- **R8** — no-match disposition (`recon_nomatch`, scoped exclusion from the
  main frame, no abort).
- **R13** — ESTATUS C.CLOUD type contract (force-cast to Utf8).
- **R14** — USAR eviction gate, closed-domain assumption.
- **R22** — active FECHA validation (malformed FECHA routes to anomaly
  queue, no abort).
- **R31** — mapeo-folder accessibility gate (global abort on definitive
  negative).
- **R32** — per-obra mapeo presence resolution (scoped exclusion on
  zero/ambiguous match).
