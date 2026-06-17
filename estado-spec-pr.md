# ESTADO DE SPEC — MÓDULO PR

> Documento canónico de estado. Bajo Spec Driven Development, **este documento ES el estado del proyecto**. No existe memoria entre sesiones distinta de este archivo. Protocolo: se relee completo al inicio de cada sesión; se reemplaza con la versión más reciente al cierre (disciplina state-commit). Cualquier contradicción entre este documento y la conversación se resuelve a favor de este documento o se eleva como conflicto explícito.

**Versión:** S4 → S5, fecha 2026-06-17.
**Sesión de origen:** Bloque 2 (Domain) — cierre de #4 (gobernanza del filtro de vigencia, ruteo Finalizado/Programado, disposición residual no-lossy, contrato de tipo en ingesta para ESTATUS C.CLOUD, y mapeo del campo estatus en la tabla 3).
**Módulo en foco:** PR — scope-lock activo (un solo módulo hasta PENDIENTE crítico vacío)
**Estado global:** reglas de negocio definidas: 13 (R1–R13) + R3.1, R6.1 bordes
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

**Tabla Mapeo de Tipologías (DEFINIDA):** tabla de dimensión curada, un archivo por obra, mantenida por analista. Esquema crudo: `[N1, N2, N3, N4, N5, Tipologia, Obra]`. Llave de join contra PR: `(obra_norm, N1_norm..N5_norm)`, ejecutada sobre columnas `_norm` en ambas tablas (R7). El sistema deriva en pipeline `obra_norm, N1_norm..N5_norm, tipologia_norm, id_tipologia`; ninguna `_norm` vive en el archivo. `obra_norm` proviene de la columna Obra cruda vía R5 (no del nombre de archivo). Vacío en Tipologia → rechazo de ingesta (R6.1).

## 4. SALIDA OBJETIVO (DECLARADA, parcialmente indefinida)

1. **Append PR Finalizado** — PR vigentes cuyo `ESTATUS_C_CLOUD_norm` cae dentro del conjunto declarado en R9 (criterio completo: 5.2/R9; ya no se reduce a la literal única "FINALIZADA"). Esquema declarado: `[id_actividad, id_tipologia, count unidades] → [obra:Utf8, id_actividad:UInt64, id_tipologia:UInt64 (nullable; null = NO_MATCH), count_unidades:UInt32]`.
2. **Append PR Programado** — PR vigentes cuyo `ESTATUS_C_CLOUD_norm` cae dentro del conjunto declarado en R10 (criterio completo: 5.2/R10). Esquema declarado: `[obra:Utf8, id_actividad:UInt64, id_tipologia:UInt64 (nullable; null = NO_MATCH), fecha_inicio:Date, count_unidades:UInt32]`.
3. **Reporte desnormalizado (humano)** — unión de ambos estados, desnormalizada. Esquema declarado: `[usar, estatus, nombre_actividad, nivel_1..5, fecha_inicio, macropartida, obra:Utf8]`. El campo `estatus` se puebla exclusivamente desde `ESTATUS C.CLOUD` crudo, sin transformación (R12). Ningún otro campo de esta tabla cambia de fuente: la tabla 3 conserva, sin excepciones, el contrato original de consumir columnas crudas en su totalidad (decisión explícita de la sesión S5 — ver bitácora).
> Términos indefinidos dentro de esta sección: `count unidades`, semántica de `fecha_inicio` por estatus. Ver auditoría.

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

R6 [cierra #2] DADO una fila del mapeo de tipologías con tipologia_norm no vacío
   CUANDO se construye la identidad de la tipología
   ENTONCES el sistema DEBE asignar
   id_tipologia = FNV-1a-64( obra_norm + "|" + tipologia_norm ),
   orden de operandos fijo (obra_norm, luego tipologia_norm), NO conmutable,
   separador "|" (∉ [a-z0-9_], garantiza serialización inyectiva),
   donde obra_norm y tipologia_norm son salidas del pipeline 5.1.
   Tipo de salida: UInt64. Homologado a R1: una sola forma de calcular hash.

R6.1 [borde, fail-fast] DADO una fila del mapeo con tipologia_norm vacío tras 5.1
   CUANDO se ingesta el archivo de mapeo
   ENTONCES el sistema DEBE detener/rechazar la ingesta de ese mapeo y reportar la
   fila, sin generar id_tipologia ni continuar el join.
   Distinto de R2: el mapeo es dimensión curada; el vacío es corrupción de origen,
   no anomalía transaccional enrutable por fila.

R7 [join, cierra #2] DADO una fila de PR con sus cinco niveles
   CUANDO se resuelve su tipología
   ENTONCES el sistema DEBE joinear contra el mapeo por la llave
   (obra_norm, N1_norm, N2_norm, N3_norm, N4_norm, N5_norm), sobre columnas
   _norm en AMBAS tablas (R5), heredando tipología e id_tipologia.
   obra_norm del mapeo se deriva de su columna Obra cruda vía R5; R3 no aplica al
   mapeo (no hay parseo de nombre de archivo).

R8 [cierra #12, provisional] DADO una fila de PR cuya llave (obra_norm, N1_norm..N5_norm)
   no tiene contraparte en el mapeo de tipologías
   CUANDO se ejecuta el join (R7)
   ENTONCES el sistema DEBE conservar la fila, asignar id_tipologia = null,
   marcar tipologia_status = NO_MATCH, y registrar la fila en el reporte de
   reconciliación, sin descartarla ni detener el pipeline.
   tipologia_status es un enum binario por fila {MATCHED, NO_MATCH}: las filas
   con match llevan MATCHED. Garantiza auditabilidad; ninguna señal queda implícita.
   El reporte de reconciliación DEBE contener al menos las tuplas distintas de
   ubicación sin match (obra + N1..N5); esquema fino y destino → Bloque 3.
   Disposición no-lossy (no se elimina dato). Provisional: la elección
   complete-vs-clean depende del contrato de consumo (E, EN DISPUTA); el default
   no-lossy preserva ambas salidas (filtrar NO_MATCH = volumen limpio; conservar
   = volumen completo).

### 5.2 Contrato de Filtro de Vigencia — ESTATUS C.CLOUD  [cierra #4]

Gobernanza: el filtro de vigencia (ruteo a Finalizado/Programado) se gobierna
exclusivamente por la columna ESTATUS C.CLOUD. La columna ESTATUS, aunque
presente en el esquema crudo, no tiene autoridad ni participación alguna en
esta lógica, bajo ninguna circunstancia. Toda comparación del filtro se
ejecuta contra el valor _norm (pipeline 5.1), nunca contra el valor crudo.

R13 [cierra #4, contrato de tipo en ingesta] DADO que ESTATUS C.CLOUD puede
   llegar desde origen con tipo subyacente incierto (texto o numérico)
   CUANDO se ingesta el archivo de PR
   ENTONCES el sistema DEBE forzar el cast de la columna completa a Utf8
   antes de ejecutar cualquier paso del pipeline 5.1, preservando los
   valores null como null (nunca como el string literal "null").
   Declarado: el valor numérico, de existir, es siempre literalmente "0",
   sin variantes decimales ni de formato; bajo este supuesto no se requiere
   un paso adicional de normalización de formato flotante. Si este supuesto
   deja de cumplirse (aparecen otros valores numéricos), esta regla debe
   reabrirse.

R9 [cierra #4, ruteo Finalizado] DADO una fila de PR
   CUANDO ESTATUS_C_CLOUD_norm ∈ {finalizada, 2da_revision, con_fallas, revisada, en_proceso}
   ENTONCES el sistema DEBE rutear la fila a Append PR Finalizado.

R10 [cierra #4, ruteo Programado] DADO una fila de PR
   CUANDO ESTATUS_C_CLOUD_norm ∈ {nueva, "0", null, ""}
   ENTONCES el sistema DEBE rutear la fila a Append PR Programado.

R11 [cierra #4, disposición residual, no-lossy] DADO una fila de PR cuyo
   ESTATUS_C_CLOUD_norm no pertenece ni al conjunto de R9 ni al de R10
   CUANDO se ejecuta el filtro
   ENTONCES el sistema DEBE conservar la fila (no descartarla), rutearla a
   una cola de reconciliación/excepción no-lossy, y NO asignarla a Append PR
   Finalizado ni a Append PR Programado. Homologado en espíritu a R8: ningún
   estatus no reconocido se pierde silenciosamente. Identidad exacta del
   artefacto de destino (¿mismo reporte de reconciliación de R8 sirviendo dos
   modos de fallo, o uno separado?) y su esquema fino: diferido a Bloque 3,
   igual que R8.

R12 [cierra #4, mapeo de salida — tabla humana] DADO que la tabla 3 consume
   columnas crudas en su totalidad (reafirmación explícita del contrato 5.1,
   sin excepciones de campo — se evaluó y se rechazó una enmienda global que
   habría migrado la tabla 3 completa a consumir _norm, por destruir la
   legibilidad humana que es la razón de ser de esa tabla)
   CUANDO se puebla el campo estatus de la tabla 3
   ENTONCES el sistema DEBE poblarlo exclusivamente con el valor crudo de
   ESTATUS C.CLOUD, sin transformación alguna, y nunca con ESTATUS ni con su
   forma _norm.

## 6. AUDITORÍA — GRIETAS ABIERTAS

"AUDITORÍA es descriptiva; el LEDGER (§8) es autoritativo para estado" 

~~**A. Identidad y llaves (núcleo).** Regla de construcción de `id_actividad` no declarada. Llave de join de `id_tipologia` contra mapeo de tipologías no declarada (¿5 niveles, subconjunto, ACTIVIDAD?). Llave de `obra` inexistente en todo esquema declarado, pese a que el output es "por cada obra".~~

**B. Filtrado y estatus (parcial).** ~~Dos columnas de estatus (`ESTATUS`, `ESTATUS C.CLOUD`): cuál gobierna el filtro no está declarado.~~ Resuelto: `ESTATUS C.CLOUD` gobierna en exclusiva (5.2, R9–R13). ~~Filas fuera de {FINALIZADA, Nueva}: destino no declarado (riesgo de drop silencioso).~~ Resuelto: cola de reconciliación no-lossy, homologada a R8 (R11). Pendiente: semántica de `USAR` (valores y efecto) — ver #5.
 
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
  (obra = substring antes del primer "_", R3; obra_norm vía R5; guard R3.1
  para nombre sin "_")
- Llave de join PR ↔ mapeo de tipologías: (obra_norm, N1_norm..N5_norm)
  sobre _norm en ambas tablas. [A] (R7)
- Contrato de identidad id_tipologia: FNV-1a-64(obra_norm + "|" +
  tipologia_norm), orden fijo, separador "|", tipo UInt64. [A] (R6)
- Guard fail-fast: Tipologia vacía en mapeo → rechazo de ingesta, sin
  generar id ni continuar join. [A] (R6.1)
- Esquema y topología del mapeo: [N1..N5, Tipologia, Obra] crudo, un
  archivo por obra; obra_norm vía R5 sobre columna Obra. [A]
- DECISIÓN: el sistema confía en que el analista ingresa Obra
  correctamente; validación nombre_archivo ↔ columna Obra diferida a
  diseño futuro. Riesgo de null por desajuste: aceptado conscientemente.
- Disposición del no-match referencial PR↔mapeo (provisional): fila se
  conserva, id_tipologia = null + tipologia_status = NO_MATCH (enum
  binario por fila {MATCHED, NO_MATCH}) + reporte de reconciliación.
  No-lossy. [A] (R8) Dependiente de E (EN DISPUTA): default no-lossy bajo
  incertidumbre del consumidor.
- Gobernanza del filtro de vigencia: ESTATUS C.CLOUD exclusivamente;
  ESTATUS no participa bajo ninguna circunstancia. Comparación del filtro
  exclusivamente contra _norm. [B] (5.2)
- Contrato de tipo de ESTATUS C.CLOUD en ingesta: cast forzado a Utf8 antes
  de 5.1, preservando null; valor numérico, si existe, siempre literal "0".
  [B] (R13)
- Conjuntos de ruteo del filtro de vigencia: Finalizado (R9), Programado
  (R10). [B]
- Disposición residual no-lossy para ESTATUS_C_CLOUD_norm no reconocido:
  cola de reconciliación/excepción, exclusión de ambas tablas append,
  homologada a R8. Identidad exacta del artefacto destino diferida a
  Bloque 3, igual que R8. [B] (R11)
- Mapeo del campo estatus en la tabla 3: exclusivamente desde ESTATUS
  C.CLOUD crudo, sin transformación. Reafirma sin excepciones que la tabla
  3 consume columnas crudas en su totalidad; se evaluó y se rechazó
  explícitamente una enmienda global al contrato 5.1 que habría migrado la
  tabla 3 completa a consumir _norm. [B] (R12)

### PARCIALMENTE DEFINIDO
- Canonicalización de `""` vs `null` en ESTATUS C.CLOUD. Ambos valores ya
  están aceptados independientemente en el conjunto de ruteo de Programado
  (R10); el comportamiento de ruteo no depende de esta distinción y por
  tanto no bloqueó el cierre de #4. Abierto: si deben colapsarse a una sola
  representación canónica en ingesta, o conservarse como estados crudos
  distintos — relevante específicamente para el campo estatus de la tabla
  3 (R12), que al consumir el valor crudo sí distingue entre ambos.

### PENDIENTE (crítico — bloquea generación de archivos)
5. Semántica de USAR (valores y efecto). [B]
6. Definición de "unidad" y nivel de agrupación del count. [C]
   (anchor: id_tipologia es identidad-de-tipología (R6); con R8, null en
   id_tipologia es valor de output legítimo (NO_MATCH). El grano de R4
   (obra_norm, id_actividad) DEBE modificarse para preservar la distinción
   matched/NO_MATCH al agregar, o colapsa filas con tipología real contra
   filas sin ella. Grano nuevo: pendiente en #6.)
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

**S3 — 2026-06-17**: cierre de #2. Llave de join PR↔mapeo = (obra_norm, N1_norm..N5_norm) sobre _norm; id_tipologia = FNV-1a-64(obra_norm + "|" + tipologia_norm), UInt64, orden fijo; guard fail-fast R6.1 para tipología vacía en mapeo; mapeo = archivo por obra, obra_norm desde columna Obra cruda. Surgió grieta nueva #12. Próximo foco candidato: #12, luego barrer #4/#5.

**S4 — 2026-06-17**: cierre provisional de #12. Disposición no-lossy del no-match referencial: fila se conserva, id_tipologia = null + tipologia_status = NO_MATCH (enum binario por fila) + reporte de reconciliación (R8). Consecuencias: id_tipologia pasa a UInt64 nullable en §4; #6 se agudiza (el grano debe preservar matched/NO_MATCH). Provisional, dependiente de E (contrato de consumo, EN DISPUTA). Próximo foco candidato: barrer #4/#5.

**S5 — 2026-06-17**: cierre de #4. Decisiones: ESTATUS C.CLOUD gobierna el filtro de vigencia en exclusiva, ESTATUS sin autoridad alguna; comparación del filtro exclusivamente contra _norm; conjuntos de ruteo declarados para Finalizado (R9) y Programado (R10); disposición residual no-lossy para estatus no reconocidos, homologada a R8 (R11); contrato de tipo en ingesta para ESTATUS C.CLOUD — cast forzado a Utf8 antes de 5.1, null preservado, valor numérico siempre literal "0" (R13); campo estatus de la tabla 3 mapea exclusivamente desde ESTATUS C.CLOUD crudo (R12). Nota de proceso: se evaluó y se rechazó explícitamente una enmienda global al contrato 5.1 que habría migrado la tabla 3 completa a consumir _norm — la tabla 3 conserva su contrato original de columnas crudas, sin excepciones, porque es la única tabla cuya función es ser leída por un humano. Abierto, no bloqueante: canonicalización `""` vs `null` en ESTATUS C.CLOUD; identidad exacta del artefacto de reconciliación de R11, diferida a Bloque 3 junto con R8. Próximo foco candidato: #5 (semántica de USAR) o #6/#7/#8, a decisión de Alberto.
