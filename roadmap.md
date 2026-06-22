# PR Roadmap — Phases 0 through 9

## FASE 0: Local Repo & Scaffolding

OBJETIVO: Establecer la estructura local del repositorio Alberto-dot-dot/pr necesaria para empezar a escribir código del pipeline, sin tocar provisión de GCP.

ENTREGABLES:
- Estructura del paquete Python (src-layout, nombre pr)
- Carpeta tests/ (vacía — fixtures pertenecen a Fase 1)
- Backend de build declarado (pyproject.toml mínimo, sin Poetry)
- Manifiesto de dependencias (requirements.txt, pinneado)
- Entorno virtual instalado y verificado (smoke test)
- .gitignore, README.md, .vscode/extensions.json

## FASE 1 — Polars Core Pipeline (local)

Objetivo: Implementar y validar contra fixtures locales todas las reglas
  de fila propiedad de Polars (D-10.3), produciendo el frame stg_matched +
  recon_nomatch sin dependencia alguna de GCP, Drive ni BigQuery.
Entregables: pipeline Polars ejecutable end-to-end sobre archivos fixture;
  suite de fixtures cubriendo cada gate y cada cola; frame de salida
  row-level con esquema fijado y contrato de salida documentado.

1.1 Test Fixtures
  Objetivo: Construir el sustrato de prueba — archivos PR, mapeo y
    nombre_programa válidos y deliberadamente corruptos — contra el que
    se valida cada regla aguas abajo.
  Entregables: fixtures PR (válido + corruptos), fixtures mapeo
    (válido + tipología vacía), fixtures nombre_programa (con/sin "_"),
    fixtures de salida esperada por regla.

1.2 Structural Ingestion Gates & Type Contracts   [R16, R17, R18, R13]
  Objetivo: Validar formato/layout/encabezado de archivo (abort global)
    y fijar el contrato de tipo de ingesta antes de normalizar.
  Entregables: gate de formato .xlsx; descarte incondicional filas 1–5 /
    cols A–S, fila 6 = header, col T = origen; validación literal de
    encabezados con abort multiobra; cast forzado de ESTATUS C.CLOUD a Utf8.

1.3 USAR Gate   [R14]
  Objetivo: Evictar filas USAR≠"SI" contra valor crudo, upstream de todo
    el pipeline, sin log ni cola.
  Entregables: gate destructivo posicionado antes de 5.1.

1.4 Normalization & Obra Derivation   [5.1, R5, R3, R3.1]
  Objetivo: Derivar obra (parse + guard) y producir las columnas _norm
    paralelas no destructivas para identidad/match/join/filtro.
  Entregables: parse de obra desde nombre_programa + guard sin "_";
    pipeline 5.1 de 10 pasos; generación <col>_norm vía R5.

1.5 Identity Hashing   [R1, R2, R6, R6.1]
  Objetivo: Construir id_actividad e id_tipologia y disponer sus casos
    borde de vacío.
  Entregables: FNV-1a-64 sobre actividad_norm (R1) + ruteo a anomalías
    si vacío (R2); FNV-1a-64(obra_norm|tipologia_norm) sobre mapeo (R6) +
    rechazo fail-fast del mapeo si tipología vacía (R6.1).

1.6 Tipología Join & Disposition   [R7, R8]
  Objetivo: Resolver tipología por join sobre _norm y disponer el
    no-match de forma no-lossy.
  Entregables: join (obra_norm, N1..N5_norm); MATCHED/NO_MATCH;
    exclusión de NO_MATCH del frame principal + escritura a recon_nomatch.

1.7 FECHA Handling   [R19, R22, (R20 obligación de columna cruda)]
  Objetivo: Validar FECHA y garantizar su disponibilidad cruda aguas abajo.
  Entregables: validación activa no-lossy ante nulo/blanco/malformado →
    cola de anomalías sin abortar (R22); columna fecha cruda preservada
    en el frame de salida para passthrough posterior (R19/R20).

1.8 stg_matched Assembly & Row-Level Output Contract   [D-10.3, R8, columnas crudas R12/R15/R20]
  Objetivo: Ensamblar y fijar el esquema del frame row-level que Polars
    entrega como frontera a Fase 4.
  Entregables: esquema explícito de stg_matched (crudas + _norm que T1/T2/T3
    requieren); esquema de recon_nomatch; stub de cola de anomalías;
    contrato de salida documentado. run_date NO se estampa aquí (Fase 4).

## FASE 2 — GCP Infrastructure Provisioning

Objetivo: Aprovisionar y configurar la infraestructura base en GCP (Job, Scheduler, 
  Artifact Registry, Datasets) para la ejecución autónoma del pipeline, estableciendo
  identidades (least-privilege) y observabilidad, sirviendo de base comprobada para Fase 3.
Entregables: APIs habilitadas y repositorio AR creado; Cloud Run Job y Scheduler
  vinculados con roles estrictos; SA del Job con lectura en Shared Drive; Datasets 
  físicos en BQ creados; contrato de salida validado y documentado.

2.1 Aprovisionamiento base y Despliegue del Cloud Run Job
  Objetivo: Provisionar el repositorio de imágenes, el Job como recurso 
    run-to-completion (D-14.1) con recursos validados contra D-14.3, y
    establecer su observabilidad básica.
  Entregables: APIs habilitadas (incluyendo Drive y Artifact Registry);
    repositorio AR creado; Job creado en el proyecto/región target; manifest
    documentado; alerta de fallos de ejecución configurada; smoke test.

2.2 Configuración del disparo programado
  Objetivo: Cloud Scheduler invocando la Execute API del Job (D-14.2), con 
    identidad de invocación least-privilege y permisos explícitos de ejecución.
  Entregables: Scheduler job creado; cadencia configurada; SA de invocación 
    con permiso run.jobs.run scoped al Job.

2.3 Autenticación de la Service Account contra Shared Drive
  Objetivo: SA del Job como miembro directo de la Shared Drive (D-14.4), 
    sin domain-wide delegation, sin sync forzada a GCS.
  Entregables: SA creada/identificada; membership confirmado.

2.4 Provisión de datasets BigQuery (container-level, D-10.4)
  Objetivo: Crear los dos datasets físicos (staging oculto, servicio) con IAM base. 
    NO crea tablas, vistas, ni el link de authorized views — propiedad de Fase 4.
  Entregables: pr_staging y pr_serving creados; IAM restringido a la SA del Job 
    (write en staging únicamente); frontera Fase2/Fase4 documentada.

2.5 Verificación de aprovisionamiento y contrato de salida
  Objetivo: Convertir "los recursos existen" en "la cadena funciona extremo a extremo", 
    y declarar explícitamente qué puede asumir Fase 3 como cerrado. No incluye lógica de R31/R32.
  Entregables: lectura real de Drive confirmada vía la SA; config física de datasets verificada; 
    smoke test Scheduler→Job completo; contrato de salida declarado.

## FASE 3 — Live Drive File Resolution

**Objetivo de Fase:** Resolver, contra la Shared Drive real, los archivos físicos
vigentes antes de que el pipeline Polars (Fase 1) procese cualquier fila. Cubre la
resolución de versión vigente del archivo PR (R23–R26) y la resolución de presencia/
match del archivo de mapeo de tipologías (R31–R32). Depende explícitamente de la
service account provisionada y verificada end-to-end en Fase 2 (Paso 2.5).

**Decisión arquitectónica de gate (S18):** R31 (accesibilidad de carpeta global de
mapeos) y R23–R26 (resolución de archivo PR vigente) son pistas estructuralmente
independientes — tocan árboles de carpetas sin relación entre sí. Se diseñan como un
único Precondition Gate concurrente con semántica OR-abort (si cualquiera de las dos
pistas falla, la corrida completa aborta antes de todo procesamiento de datos) y
AND-success (ambas pistas deben resolver para levantar la barrera). R32 NO forma parte
de este gate: falla por-obra (exclusión acotada), no por-batch (abort global), y vive
en su propio Paso aguas abajo.

---

### Paso 3.1 — Production Image Build & Job Redeployment

**Nombre:** Construcción de imagen de producción y redespliegue del Job.

**Objetivo:** Reemplazar el placeholder de imagen empujado en 2.1.3 por el código real
del pipeline de Fase 1, redesplegar el Cloud Run Job contra esa imagen, y reconfirmar
que el Job ejecuta sin errores de runtime. Resuelve el gap explícito diferido al cierre
de Fase 2: ninguna Fase poseía la tarea de imagen de producción, y la resolución en vivo
no puede testearse contra un contenedor placeholder.

**Entregables:**
- Imagen de producción del pipeline (código de Fase 1) en Artifact Registry, reemplazando
  el tag placeholder de 2.1.3.
- Cloud Run Job redesplegado referenciando la imagen de producción.
- Smoke test (homólogo a 2.1.6) re-ejecutado contra la imagen real — confirma ejecución
  sin error de runtime, aún no contra datos en vivo de Drive.

---

### Paso 3.2 — Precondition Gate: Mapeo-Folder Accessibility (R31) ∥ PR-Vigente Resolution (R23–R26)

**Nombre:** Gate de precondición — accesibilidad de carpeta de mapeos ∥ resolución de PR vigente.

**Objetivo:** Implementar la barrera concurrente que resuelve ambas pistas independientes
antes de que cualquier lógica row-level de Polars se ejecute. Incluye manejo de errores
transitorios de la Drive API (retry-before-abort) diferenciado de los fallos definitivos.

**Entregables:**
- Gate de accesibilidad de la carpeta global `Tipologias Obras` (R31), con reintento ante
  error transitorio y abort global ante fallo definitivo o reintentos agotados.
- Resolución de archivo PR vigente (R23–R26): recorrido recursivo, descarte de carpetas no
  conformes, selección por fecha máxima, abort global ante cero coincidencias o empate
  (mtime excluido como criterio de desempate), con reintento ante error transitorio.
- Wrapper de concurrencia con semántica OR-abort / AND-success: ambas pistas corren en
  paralelo; el primer fallo definitivo en cualquiera detiene el batch de inmediato; ambas
  deben resolver para proceder a 3.3.
- Diferenciación explícita transitorio (reintenta) vs. definitivo (aborta en primera
  respuesta) registrada en el log con su clase de fallo.

---

### Paso 3.3 — Per-Obra Mapeo Presence Resolution (R32)

**Nombre:** Resolución de presencia de mapeo por obra.

**Objetivo:** Para cada obra que pasó el gate de 3.2, resolver su archivo de mapeo
específico dentro de la carpeta global ya confirmada accesible. Falla acotada a la obra
afectada (exclusión), no abort global — divergencia deliberada respecto a R25/R26, fijada
en S15.

**Entregables:**
- Match de nombre de archivo normalizado (pipeline 5.1) de `{obra_norm}_mapeo_tipologias`
  contra `obra_norm` — divergencia deliberada de la postura literal de R18.
- Manejo de cero/ambiguo: exclusión de todas las macro_partidas de la obra afectada de la
  corrida actual, log de obra + motivo, continuación del resto del batch sin abortar.
- Handoff del path de mapeo resuelto hacia la lógica existente de Fase 1 (R6, R6.1, R7)
  para las obras con match exacto.

---

### Paso 3.4 — Live Resolution Verification & Exit Contract toward Fase 4

**Nombre:** Verificación de resolución en vivo y contrato de salida hacia Fase 4.

**Objetivo:** Probar, contra la Shared Drive real, un ciclo completo de resolución en vivo
— no contra fixtures. Homólogo al Paso terminal 2.5 de Fase 2. Garantiza que Fase 4
(agregación y vistas BigQuery) puede asumir, sin dependencia residual, que datos vivos y
validados ya fluyen al landing point.

**Entregables:**
- Un ciclo completo en vivo para al menos una obra real con archivos PR y de mapeo
  conocidos-buenos: resolución exitosa a través de 3.2 y 3.3.
- Confirmación de que el pipeline de Fase 1 (construido/testeado solo contra fixtures) se
  comporta idénticamente alimentado con paths resueltos en vivo.
- Escritura real WRITE_TRUNCATE a `stg_matched` / `recon_nomatch` en `pr_staging`, con
  `run_date` estampado correctamente (R27).
- Verificación de cada ruta de abort/exclusión contra artefactos de prueba desechables (no
  datos de producción): carpeta de mapeos inaccesible, PR vigente cero/empate, mapeo por
  obra cero/ambiguo — confirma abort global para las dos primeras y exclusión acotada para
  la tercera.
- Contrato de salida documentado: al cierre de Fase 3, `stg_matched` / `recon_nomatch`
  contienen datos vivos y validados; Fase 4 sin dependencia residual de Fase 3.
  
## FASE 4 — BigQuery Aggregation & Service Views (producer side)
Owns content inside pr_staging/pr_serving (containers provisioned in Fase 2, D-10.4).

### Paso 4.1 — Service Aggregation Views (T1, T2)
  Objetivo: define the two aggregation views in pr_serving — Append PR Finalizado
    (R4-T1, routing R9) and Append PR Programado (R4-T2, routing R10) — counting
    MATCHED rows, run_date as a GROUP BY member.
  Entregables: T1 view def, T2 view def (pr_serving).

### Paso 4.2 — Service Denormalized View (T3)
  Objetivo: define the row-level passthrough view in pr_serving — raw values
    (R15/R12/R20), membership filter R9∪R10, run_date passthrough.
  Entregables: T3 view def (pr_serving).

### Paso 4.3 — Residual Exception View (R30, row-level)
  Objetivo: expose the R11 residual (MATCHED rows whose estatus_c_cloud_norm
    ∉ R9∪R10) as a row-level view in pr_staging, raw identifying columns for
    upstream correction, isolated from the serving dataset.
  Entregables: R30 residual view def (pr_staging).

### Paso 4.4 — Authorized-View Link (serving → staging)
  Objetivo: authorize T1/T2/T3 against pr_staging so the serving views can read
    their source; confirm queryability; confirm R30 is excluded from any outward grant.
  Entregables: authorized-view grant on pr_staging covering T1/T2/T3 only.

### Paso 4.5 — Terminal Live Verification & Exit Contract
  Objetivo: confirm T1/T2/T3 + R30 return correct results against the live
    stg_matched written in Fase 3; confirm run_date is passthrough-only; declare
    the Fase 4 → Fase 5 exit contract with no residual dependency.
  Entregables: verification record; exit contract to Fase 5.