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