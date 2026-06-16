# ESTADO DE SPEC — MÓDULO PR
 
> Documento canónico de estado. Bajo Spec Driven Development, **este documento ES el estado del proyecto**. No existe memoria entre sesiones distinta de este archivo. Protocolo: se relee completo al inicio de cada sesión; se reemplaza con la versión más reciente al cierre (disciplina state-commit). Cualquier contradicción entre este documento y la conversación se resuelve a favor de este documento o se eleva como conflicto explícito.
 
**Versión:** S1 → S2, fecha 2026-06-16.
**Sesión de origen:** Bloque 2 (Domain) — definición de id_actividad (#1) y normalización PR (#9).
**Módulo en foco:** PR — scope-lock activo (un solo módulo hasta PENDIENTE crítico vacío)
**Estado global:** reglas de negocio definidas: 5 (R1–R5) + R3.1 borde
**Test de verificabilidad (Definition of Done):** una regla está DEFINIDA si y solo si se expresa como
`DADO [precondición] CUANDO [evento] ENTONCES el sistema DEBE [resultado observable]`, sin huecos.
Si no, va a PENDIENTE. No se juzga ambigüedad por intuición; se aplica este test.
 
---
 
## 1. OBJETIVO (DEFINIDO)
 
Consolidar todos los Programas Rítmicos (PR) vigentes de cada obra y dejarlos para consumo en 3 tablas, migrando el pipeline desde Power Query / M Language a un entorno de cómputo en la nube con procesamiento columnar. Sustituye un proceso local actual de >6 min por ejecución.
 
## 2. DRIVER (DEFINIDO)
 
- Latencia: actualización actual >6 min, dependiente de ancho de banda y CPU local.
- Auditabilidad: M no tiene entorno de desarrollo propio; difícil debuggear.
- Automatización ausente: depende de clic manual; los errores se descubren tras esperar el ciclo completo.
- Sin query folding (impacto menor).
- Migración a entorno Workspace prevista para el próximo año; obliga a iniciar el trabajo de migración.
> Nota de rol: el driver motiva el proyecto pero NO es especificación. "Más rápido" no entra en reglas de negocio.
 
## 3. ENTRADA CONOCIDA (DEFINIDO como hecho declarado)
 
**Hoja de programas vigentes:** [ruta_archivo:Utf8, nombre_programa:Utf8], fuente del acrónimo de obra y de la resolución de "vigente" (toca #11). 

**Programa Rítmico (PR):** programa de ejecución de obra, estandarizado en Excel. Cada PR corresponde a una macro partida.
Macro partidas declaradas (≈4; variabilidad del conjunto no confirmada → ver PENDIENTE):
`OBRA GRUESA · TERMINACIONES_DEPTOS · TERMINACIONES_EECC · FACHADAS`

**Esquema crudo del PR (declarado):**
`USAR · AO · PT · SUPERVISOR · EMPRESA · ESTATUS · ESTATUS C.CLOUD · ACTIVIDAD · Nivel 1 · Nivel 2 · Nivel 3 · Nivel 4 · Nivel 5 · TIPO ACTIVIDAD · FECHA · SEM CAL · SEM OBRA · CONT PLAN · AVANCE · DIF AVANCE · CONT REAL · DIC CONT REAL · CORTE · INDICE FRENTE · INDICE · POR ASIGNAR · ACT PEND · ACT PEND ACUM · ASIGNAR · SUG · PAC`
 
Semántica de niveles declarada: N1 frente de trabajo · N2 edificio/torre · N3 bloque · N4 piso · N5 ciclo de ejecución o departamento (nivel mínimo).
 
**Columnas obligatorias / de trabajo (declaradas):** `USAR · ESTATUS · ACTIVIDAD · NIVELES (1–5) · FECHA`
 
**Precondición estructural de parseo (declarada):** las primeras 5 filas y la primera columna del archivo están vacías. (Universalidad para todas las obras: no confirmada → PENDIENTE F.)
 
**Tabla Mapeo de Tipologías (declarada):** tabla de dimensiones que mapea niveles a una tipología específica. Llave de join: no declarada → PENDIENTE A.
 
## 4. SALIDA OBJETIVO (DECLARADA, parcialmente indefinida)
 
1. **Append PR Finalizado** — PR vigentes con estatus "FINALIZADA". Esquema declarado: `[id_actividad, id_tipologia, count unidades] → [obra:Utf8, id_actividad:UInt64, id_tipologia:TBD, count_unidades:UInt32]`.
2. **Append PR Programado** — PR vigentes con estatus "Nueva". Esquema declarado: `[obra:Utf8, id_actividad:UInt64, id_tipologia:TBD, fecha_inicio:Date, count_unidades:UInt32]`.
3. **Reporte desnormalizado (humano)** — unión de ambos estados, desnormalizada. Esquema declarado: `[usar, estatus, nombre_actividad, nivel_1..5, fecha_inicio, macropartida, obra:Utf8]`.
> Términos indefinidos dentro de esta sección: `id_tipologia`, `count unidades`, semántica de `fecha_inicio` por estatus. Ver auditoría.

## 5. REGLAS DE NEGOCIO — DOMAIN (Bloque 2)

R1 [cierra #1] DADO una fila de PR con ACTIVIDAD no nula
   CUANDO se construye su identidad
   ENTONCES el sistema DEBE asignar
   id_actividad = FNV-1a-64( actividad_norm ),
   donde actividad_norm es la salida del pipeline 5.1 sobre ACTIVIDAD.

R2 [cierra #9, borde] DADO una ACTIVIDAD cuyo pipeline 5.1 resulta vacío
   CUANDO se intenta hashear
   ENTONCES el sistema DEBE enrutar la fila a la cola de anomalías
   y NO generar id_actividad.

R3 [cierra #3] DADO un nombre_programa de la hoja de programas vigentes
   CUANDO el motor ingesta cada PR
   ENTONCES el sistema DEBE extraer obra = substring anterior al primer "_",
   incrustarla como columna obra, y derivar obra_norm vía 5.1 (R5).

R3.1 [borde] DADO un nombre_programa sin "_"
   CUANDO se intenta extraer obra
   ENTONCES el sistema DEBE enrutar a la cola de anomalías y NO asignar obra
   (no tomar el string completo).   

R4 [grano] DADO el set consolidado multiobra
   CUANDO se agregan los counts
   ENTONCES el sistema DEBE agrupar por el par (obra_norm, id_actividad).

R5 [aplicación de #9] DADO una columna de texto que participa en
   identidad, match, join o filtro
   CUANDO se prepara el dataset
   ENTONCES el sistema DEBE generar una columna paralela <col>_norm
   aplicando el pipeline 5.1, preservando intacta la columna cruda.

### 5.1 Contrato de Normalización PR  [cierra #9]

Naturaleza: transformación paralela y no destructiva. Por cada columna de
texto que participa en identidad/match/join/filtro, el sistema genera
<col>_norm; la columna cruda permanece intacta y conserva su tipo. La
tabla 3 (humana) consume columnas crudas; hash, join (#2) y filtro (#4)
consumen las _norm.

Tipos: entrada <col>:Utf8 no-nulo → salida <col>_norm:Utf8.
Instancia de identidad: actividad_norm:Utf8 → FNV-1a-64 → id_actividad:UInt64.

Pipeline ordenado (el orden es normativo, no reordenar):
1. Validación no-nulo (origen).
2. NFKD (descomposición Unicode de compatibilidad).
3. Strip de marcas combinantes (diacríticos).
4. Forzado ASCII (descarte de no-ASCII residual).
5. Lowercase.
6. Símbolos → espacio (dígitos preservados).
7. Whitespace (\t \n \r, U+00A0, Unicode) → espacio único; colapso; trim.
8. Si vacío: columnas de identidad (ACTIVIDAD) → cola de anomalías, NO hashear
   (ver R2); otras columnas → flag, disposición según su regla downstream.
9. snake_case (espacios → underscore, último).
10. Salida (token Utf8). En el caso identidad, alimenta FNV-1a-64.

## 6. AUDITORÍA — GRIETAS ABIERTAS

"AUDITORÍA es descriptiva; el LEDGER (§8) es autoritativo para estado" 

~~**A. Identidad y llaves (núcleo).** Regla de construcción de `id_actividad` no declarada. Llave de join de `id_tipologia` contra mapeo de tipologías no declarada (¿5 niveles, subconjunto, ACTIVIDAD?). Llave de `obra` inexistente en todo esquema declarado, pese a que el output es "por cada obra".~~
 
**B. Filtrado y estatus.** Dos columnas de estatus (`ESTATUS`, `ESTATUS C.CLOUD`): cuál gobierna el filtro no está declarado. Filas fuera de {FINALIZADA, Nueva}: destino no declarado (riesgo de drop silencioso). Semántica de `USAR` (valores y efecto) no declarada.
 
**C. Agregación.** "Unidad" no definida; nivel de agrupación del count no declarado. `FECHA` cambia de significado según estatus (Programado conserva fecha, Finalizado la descarta) sin regla declarada.
 
**D. Frontera arquitectónica.** Tensión entre "migrar todo a GCP+Polars" y "consumir vía PQ/Sheets". Sin definir: dónde aterrizan los outputs (BigQuery / GCS / Sheet), qué extrae PQ (crudo vs final), qué significa técnicamente "compatible con Sheets", y el mecanismo que resuelve "versión vigente" fuera de M.
 
**E. Contrato de consumo y grafo.** El "MRP Excel" consumidor no está identificado contra el grafo del sistema (¿PLANTIR? ¿módulo sin nombre? ¿legado externo?). Riesgo: el consumidor actual espera 13 columnas a nivel de fila; el nuevo output entrega tablas agregadas (count). Si necesita detalle por ubicación, la agregación rompe el contrato.
 
~~**F. Precondiciones estructurales.** El artefacto "5 filas + 1 columna vacías" debe entrar al contrato de ingesta con su caso borde: ¿garantizado para todas las obras o hay desviaciones que rompen el lector? Reglas de normalización propias de PR no declaradas (¿reutilizan las de MIDAS o son propias?).~~

 
## 7. PLAN DE DESARROLLO POR BLOQUE
 
Orden por dependencia. Gating: no se genera un archivo si su bloque tiene PENDIENTE crítico abierto.
 
- **Bloque 0 — Contrato de consumo y posición en grafo** (resuelve E). Sistema-nivel; puede quedar EN DISPUTA pero al menos fija formato de entrega.
- **Bloque 1 — Objetivo + Requerimientos** (archivo). Objetivo casi listo; requerimientos dependen de B0.
- **Bloque 2 — Domain** (archivo) ← **EN CURSO**. Resuelve A, B, C, F. Núcleo Socrático.
- **Bloque 3 — Arquitectura + Stack** (archivo). Resuelve D.
- **Bloque 4 — Roadmap por fases** (archivo). Depende de B2 + B3. Checkpoints críticos candidatos: integridad de llaves con join sin pérdida; resolución determinista de versión vigente.
- **Bloque 5 — Development Fase 1** (archivo). Depende de B4.
- **Bloque 6 — CLAUDE.md** (condicional). Solo si arquitectura sin Colab y contrato de PR exigen cambios de gobernanza.
---
 
## 8. LEDGER CANÓNICO
 
### DEFINIDO
- Objetivo de negocio de PR (sección 1).
- Driver (sección 2).
- Esquema crudo del PR y columnas obligatorias (sección 3).
- Esquemas de las 3 tablas de salida (sección 4), con columna obra
  incorporada y tipos declarados.
- Regla de construcción de id_actividad: FNV-1a-64 sobre actividad_norm. [A] (R1)
- Reglas de normalización propias de PR: pipeline 5.1, paralelo y no
  destructivo, convención <col>_norm. [F] (5.1, R5)
- Consolidación multiobra y grano de agregación (obra_norm, id_actividad). (R4)
- Llave de obra. [A] Existe como columna generada e integra el grano
  (R3, R4). Residuo: regla de parseo del acrónimo desde nombre_programa
  (obra = substring antes del primer "_", R3; obra_norm vía R5; guard R3.1)

### PARCIALMENTE DEFINIDO

### PENDIENTE (crítico — bloquea generación de archivos)
2. Llave de join de id_tipologia contra mapeo de tipologías. [A]
   (anchor: al definirla, el mapeo debe pasar por 5.1 en sus columnas de
   join o el join produce nulls silenciosos.)
4. Columna de estatus que gobierna el filtro + destino de filas fuera de
   {FINALIZADA, Nueva}. [B]
   (anchor: declarar si el filtro opera sobre crudo o sobre _norm.)
5. Semántica de USAR (valores y efecto). [B]
6. Definición de "unidad" y nivel de agrupación del count. [C]
7. Semántica de FECHA/fecha_inicio por estatus. [C]
8. Universalidad de la precondición estructural de parseo (5 filas + 1
   columna vacías). [F]
10. Frontera GCP+Polars ↔ PQ/Sheets y ubicación de fuentes/outputs. [D] (Bloque 3)
11. Mecanismo de resolución de "versión vigente". [D] (Bloque 3)

### EN DISPUTA
- Posición de PR en el grafo de dependencias del sistema PLAN.
- Identidad del consumidor "MRP Excel": ¿PLANTIR, módulo sin nombre o legado externo?
- Si la salida agregada (count) rompe al consumidor que hoy espera 13
  columnas a nivel de fila.

---
 
## 9. BITÁCORA DE SESIONES
 
**S1 — 2026-06-15.** Recepción y auditoría de la historia de usuario. Construcción del plan por bloque. Generación de este documento de estado. Sin reglas de negocio definidas aún. Próximo paso: Bloque 2 (Domain), Modo Socrático, primera pregunta = regla de construcción de `id_actividad`.

**S2 — 2026-06-16**: cierre de #1 y #9. Decisiones: id_actividad = FNV-1a-64 sobre texto normalizado; consolidado multiobra Camino B; pipeline de normalización de 10 pasos. "#3 cerrado — obra = string antes del primer ''; obra_norm por R5; guard R3.1 para nombre sin ''.". Proximo foco #3.