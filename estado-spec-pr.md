# ESTADO DE SPEC — MÓDULO PR

> Documento canónico de estado. Bajo Spec Driven Development, **este documento ES el estado del proyecto**. No existe memoria entre sesiones distinta de este archivo. Protocolo: se relee completo al inicio de cada sesión; se reemplaza con la versión más reciente al cierre (disciplina state-commit). Cualquier contradicción entre este documento y la conversación se resuelve a favor de este documento o se eleva como conflicto explícito.

**Versión:** S18 → S19, fecha 2026-06-22
**Sesión de origen:** Bloque 4 — descomposición de Fase 4 (BigQuery Aggregation & Service
Views, lado productor) a NIVEL DE ROADMAP (Paso). Auditoría previa detectó que Paso 1.8 (Fase 1)
portaba la etiqueta [CLOSED] sobre un contrato de salida row-level nunca enumerado — la interfaz
Polars→BigQuery (corte D-10.3) existía solo por nombre. Resuelto: stg_matched tiene DOS caras
de tipo — frame Arrow/Polars local de 9 columnas (Fase 1, ids UInt64, sin run_date) y tabla
BigQuery de 10 columnas (frontera de escritura Fase 3, ids STRING, run_date estampado). La
derivación de las 10 columnas es determinada por reglas cerradas (R5, R8/D-10.3, R9/R10,
R15/R20, R4-T1/T2, R1/R6, R27), no elegida; columnas upstream-only (N1..N5, actividad_norm,
tipologia_norm) excluidas del staging por ausencia de lector downstream. Corrección de
atribución: run_date y el cast STRING son transformaciones de la frontera de escritura (3.4.3,
Fase 3), NO de Fase 1 ni Fase 4 — el frame local no puede portar un timestamp de Job que aún no
existe (rompería 1.8.2 contra fixture 1.1.4). Cast STRING para id_actividad/id_tipologia:
BigQuery no tiene UINT64; ~50% de los hashes FNV-1a-64 exceden INT64 max y desbordarían a
negativo — STRING elegido por join bijectivo y estable. Grano de R30 resuelto: row-level
(simetría de mecanismo con T1/T2, no de grano; trazabilidad para corrección upstream, espejo de
recon_nomatch). Frontera authorized-view aislada en Paso 4.4 propio; R30 confirmado en
pr_staging, fuera del grant de servicio. Decisión de proceso mayor: descomposición atómica pasa a
JIT/incremental desde Fase 4 inclusive (MODE: PLANNING → EXECUTION del Task Protocol de
CLAUDE.md) — los atómicos de Fase 4 NO se escriben upstream, se proponen en ejecución, homologada
a Fase 5; Fases 0–3 congeladas como referencia revisable; Bloque 5 disuelto (el spec de
development se escribe durante la ejecución, no upstream). Fase 4 cerrada a nivel de roadmap
(5 Pasos, sin atómicos). Próximo foco: terminar roadmap.md (Fase 5 a nivel Paso) y luego
Bloque 6 (CLAUDE.md), precondición dura de la ejecución.

**Módulo en foco:** PR — scope-lock activo (un solo módulo hasta PENDIENTE crítico vacío)
**Estado global:** reglas de negocio activas: R1–R3, R3.1, R4-T1, R4-T2, R5–R7, R8 (enmendada S9/S10/S14), R9–R10, R11 (enmendada S10), R12–R22, R23–R26 (S11, cierre #11), R27 (S13, §5.9, cierre parcial #10), R28–R30 (S14, nuevas, §5.10, cierre total #10), R31–R32 (S15, nuevas, §5.11, cierre #13), R6.1 borde. R4 retirada en S9. R15 enmendada en S10: esquema Tabla 3 reducido a [obra, estatus, nombre_actividad, tipologia, fecha]; enmendada nuevamente en S13 (R27): se agrega run_date. EN DISPUTA vacío desde S10. PENDIENTE crítico vacío desde S15.

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

**Hoja de programas vigentes:** [ruta_archivo:Utf8, nombre_programa:Utf8]. Cardinalidad: una fila por (obra, macro_partida) — macro_partida está embebida en nombre_programa, no es columna separada. ruta_archivo es una ruta de carpeta estable, sin año, declarada por el analista; no apunta directamente al archivo físico. nombre_programa es el nombre de archivo objetivo exacto. Mecanismo de resolución de "vigente" definido en 5.6 (R23–R26, cierra #11). Fuente del acrónimo de obra: ver R3.

**Programa Rítmico (PR):** programa de ejecución de obra, estandarizado en Excel. Cada PR corresponde a una macro partida.
Macro partidas declaradas (≈4; variabilidad del conjunto no confirmada → ver PENDIENTE):
`OBRA GRUESA · TERMINACIONES_DEPTOS · TERMINACIONES_EECC · FACHADAS`

**Esquema crudo del PR (declarado):**
`USAR · AO · PT · SUPERVISOR · EMPRESA · ESTATUS · ESTATUS C.CLOUD · ACTIVIDAD · Nivel 1 · Nivel 2 · Nivel 3 · Nivel 4 · Nivel 5 · TIPO ACTIVIDAD · FECHA · SEM CAL · SEM OBRA · CONT PLAN · AVANCE · DIF AVANCE · CONT REAL · DIC CONT REAL · CORTE · INDICE FRENTE · INDICE · POR ASIGNAR · ACT PEND · ACT PEND ACUM · ASIGNAR · SUG · PAC`

Semántica de niveles declarada: N1 frente de trabajo · N2 edificio/torre · N3 bloque · N4 piso · N5 ciclo de ejecución o departamento (nivel mínimo).

**Columnas obligatorias / de trabajo (declaradas):** `USAR · ESTATUS · ACTIVIDAD · NIVELES (1–5) · FECHA`

**Precondición estructural de parseo (DEFINIDA, cierra #8):** todo archivo PR, sin excepción para ninguna obra, sigue este layout físico fijo: las filas 1 a 5 y las columnas A a S (columnas 1–19) son ruido estático no estructurable — nunca se validan, se descartan incondicionalmente en la ingesta. La fila 6 es la fila de encabezado absoluta; la columna T es el origen de datos absoluto (offset 0), alineado con la primera entrada del esquema crudo (USAR). El formato de archivo se restringe exclusivamente a binario .xlsx. Reglas que gobiernan: R16–R18.

**Tabla Mapeo de Tipologías (DEFINIDA):** tabla de dimensión curada, un archivo por obra, mantenida por analista. Esquema crudo: `[N1, N2, N3, N4, N5, Tipologia, Obra]`. Llave de join contra PR: `(obra_norm, N1_norm..N5_norm)`, ejecutada sobre columnas `_norm` en ambas tablas (R7). El sistema deriva en pipeline `obra_norm, N1_norm..N5_norm, tipologia_norm, id_tipologia`; ninguna `_norm` vive en el archivo. `obra_norm` proviene de la columna Obra cruda vía R5 (no del nombre de archivo). Vacío en Tipologia → rechazo de ingesta (R6.1).

**Vigencia del archivo de mapeo (DEFINIDA, cierra #13):** a diferencia del archivo PR (§5.6), no existe mecanismo de versionado ni de "vigente" — es un archivo único y estático por obra, sobrescrito in situ por el analista; no hay estructura de carpetas por fecha. El problema resuelto en §5.11 es de presencia/identificación, no de selección temporal.

**Carpeta global de mapeos de tipología (DEFINIDA, cierra #13):** ruta declarada como valor de configuración único y global del sistema, estructuralmente distinto de `programas_vigentes` — no es una columna de esa hoja, porque su valor no varía por (obra, macro_partida); es una constante de todo el sistema. Nombre de la carpeta: `Tipologias Obras`. Convención de nombre de archivo dentro de ella: `{obra}_mapeo_tipologias`, uno por obra. Accesibilidad de la carpeta gobernada por R31; resolución del archivo de una obra específica dentro de ella gobernada por R32


## 4. SALIDA OBJETIVO (DECLARADA, parcialmente indefinida)

1. **Append PR Finalizado** — PR vigentes cuyo `ESTATUS_C_CLOUD_norm` cae dentro del conjunto declarado en R9 (criterio completo: 5.2/R9; ya no se reduce a la literal única "FINALIZADA"). Esquema declarado: `[obra:Utf8, id_actividad:UInt64, id_tipologia:UInt64, count_unidades:UInt32, run_date:Date] (R27, S13: señal de frescura de snapshot)`. id_tipologia no nulo por construcción: las filas NO_MATCH son excluidas del output final antes de la agregación (R8 enmendada, R4-T1). Excluye el campo fecha por decisión de esquema, no por vaciamiento semántico (R21).

2. **Append PR Programado** — PR vigentes cuyo `ESTATUS_C_CLOUD_norm` cae dentro del conjunto declarado en R10 (criterio completo: 5.2/R10). Esquema declarado: `[obra:Utf8, id_actividad:UInt64, id_tipologia:UInt64, fecha:Date, count_unidades:UInt32, run_date:Date] (R27, S13; no confundir con fecha, dato de negocio R19–R22)`. id_tipologia no nulo por construcción: las filas NO_MATCH son excluidas del output final antes de la agregación (R8 enmendada, R4-T2). Semántica de `fecha` fijada por R19; validación ante nulo/malformado por R22. La distribución de count_unidades por fecha es el resultado correcto e intencional (R4-T2): no se colapsa.

3. **Reporte desnormalizado (humano)** — unión de ambos estados, desnormalizada. Esquema declarado (enmendado S10, enmendado nuevamente S13): `[obra:Utf8, estatus:Utf8, nombre_actividad:Utf8, tipologia:Utf8, fecha:Date, run_date:Date] (R27: señal de frescura; coexiste con fecha sin relación entre ambas)`. La columna usar se excluye: toda fila sobreviviente al gate de R14 tiene USAR=="SI" por construcción (R15). Los campos nivel_1..5 y macropartida se excluyen: el consumidor humano no requiere detalle de ubicación ni tipo de actividad (R15 enmendada, S10).
Declaraciones de fuente por campo: `obra` — derivada vía R3 (substring antes del primer "_" en nombre_programa); `estatus` — valor crudo de ESTATUS C.CLOUD, sin transformación (R12); `nombre_actividad` — columna cruda ACTIVIDAD del archivo PR; `tipologia` — columna cruda Tipologia del archivo de mapeo, pre-normalización, disponible tras el join R7; no nulo por construcción: las filas NO_MATCH son excluidas de la Tabla 3 antes del armado (R8 enmendada S10); `fecha` — columna cruda FECHA del archivo PR, sin transformación (R20).
La tabla 3 conserva, sin excepciones, el contrato de consumir valores crudos (5.1, R12). Membresía de la Tabla 3: solo filas MATCHED (R8 enmendada S10) y solo filas ruteadas a Finalizado o Programado (R9/R10); las residuales de estatus no reconocido (R11 enmendada S10) quedan excluidas, en simetría con las NO_MATCH.

> Términos indefinidos dentro de esta sección: ninguno. #6 cerrado en S9: "unidad" definida, grano corregido (R4-T1, R4-T2), R8 enmendada.

## 5. REGLAS DE NEGOCIO — DOMAIN (Bloque 2)

### 5.0 Contrato de Gating — USAR [cierra #5]

Gobernanza: el dominio de valores de USAR se declara cerrado y binario,
{SI, NO}, sin variantes observadas. La evaluación del gate se ejecuta
contra el valor crudo de USAR; la columna no genera contraparte _norm
bajo el pipeline 5.1 (excepción explícita a R5).

R14 [cierra #5, gating] DADO una fila de PR cuyo valor crudo de USAR es distinto de "SI"
   CUANDO se ingesta el archivo
   ENTONCES el sistema 
   DEBE evictar la fila de forma completa y permanente
   antes de ejecutar 5.1 (normalización), R1 (hash id_actividad), R7 (join
   de tipología) y R9–R11 (filtro de vigencia), sin generar conteo, log, ni
   artefacto de reconciliación. La comparación se ejecuta contra el valor
   crudo; USAR no genera columna _norm. Excepción explícita a la postura
   no-lossy del sistema, de signo opuesto a R8/R11: riesgo aceptado de
   pérdida silenciosa de filas válidas ante variación de casing o
   espacios, bajo el supuesto declarado de dominio cerrado {SI, NO}.
   Nota S13 (R27, §5.9): run_date (señal de frescura operacional, distinta de
   fecha) se incluye como miembro adicional del GROUP BY; al ser constante
   dentro de cualquier snapshot, su inclusión no fragmenta este grano.

R15 [cierra #5, esquema Tabla 3; enmendada S10] DADO que toda fila sobreviviente a R14
   tiene USAR=="SI" por construcción
   CUANDO se define el esquema de salida de la Tabla 3 (reporte desnormalizado)
   ENTONCES el sistema DEBE excluir la columna usar (constante por construcción),
   excluir nivel_1..5 y macropartida (el consumidor humano no requiere detalle de
   ubicación ni tipo de actividad), e incluir tipologia como el valor crudo de la
   columna Tipologia del archivo de mapeo, pre-normalización, disponible tras el join R7.
   tipologia es no nulo por construcción: solo filas MATCHED integran la Tabla 3 (ver R8 enmendada S10).
   Esquema vigente (enmendado S10, enmendado S13):
   [obra:Utf8, estatus:Utf8, nombre_actividad:Utf8, tipologia:Utf8, fecha:Date, run_date:Date].
   ESTATUS C.CLOUD permanece en Tabla 3 como campo estatus vía R12. run_date es señal de
   frescura operacional (R27, §5.9), distinta de fecha (dato de negocio, R19–R22); ambas
   coexisten sin relación entre sí.

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

~~R4 [grano] DADO el set consolidado multiobra
   CUANDO se agregan los counts
   ENTONCES el sistema DEBE agrupar por el par (obra_norm, id_actividad)~~
> R4 retirada en S9. Reemplazada por R4-T1 y R4-T2 en Sección 5.5.

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

R8 [cierra #12, enmendada en S9 y S10] DADO una fila de PR cuya llave (obra_norm, N1_norm..N5_norm)
   no tiene contraparte en el mapeo de tipologías
   CUANDO se ejecuta el join (R7)
   ENTONCES el sistema DEBE asignar id_tipologia = null, marcar
   tipologia_status = NO_MATCH, excluir la fila del output final de Tabla 1,
   Tabla 2 y Tabla 3, y registrarla en el reporte de reconciliación para
   tratamiento posterior por el analista. El pipeline no se detiene.
   tipologia_status es un enum binario interno por fila {MATCHED, NO_MATCH}:
   las filas con match llevan MATCHED. Es una columna de enrutamiento interno;
   no aparece en ningún esquema de output final.
   Solo las filas con tipologia_status = MATCHED pasan a la agregación de R4-T1
   y R4-T2; id_tipologia es no nulo en ambos outputs finales por construcción.
   La Tabla 3 también consume exclusivamente filas MATCHED: tipologia es no nulo
   en su esquema por construcción (enmienda S10).
   El reporte de reconciliación DEBE contener al menos las tuplas distintas de
   ubicación sin match (obra + N1..N5); esquema fino y destino → Bloque 3.
   Enmienda respecto a versión provisional: el contrato de consumo (E) está
   parcialmente resuelto para Tabla 1 y Tabla 2 — los módulos PLAN consumen
   únicamente datos validados (MATCHED); las filas NO_MATCH van exclusivamente
   al reporte de reconciliación. La no-lossiness se mantiene a nivel de pipeline;
   los outputs finales son limpios.

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

R11 [cierra #4, disposición residual, no-lossy, enmendada S10] DADO una fila de PR cuyo
   ESTATUS_C_CLOUD_norm no pertenece ni al conjunto de R9 ni al de R10
   CUANDO se ejecuta el filtro
   ENTONCES el sistema DEBE conservar la fila (no descartarla), rutearla a
   una cola de reconciliación/excepción no-lossy, y NO asignarla a Append PR
   Finalizado ni a Append PR Programado. Adicionalmente, la fila se excluye de
   la Tabla 3 (reporte humano): la Tabla 3 consume exclusivamente filas ruteadas
   a Finalizado (R9) o Programado (R10), por lo que una fila residual, que no
   pertenece a ninguno de los dos estados, no aparece en la unión (enmienda S10,
   simétrica a la exclusión de NO_MATCH en R8). Homologado en espíritu a R8: ningún
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

### 5.3 Contrato de Ingesta Estructural — Layout de Archivo PR [cierra #8]

Gobernanza: estos chequeos se ejecutan antes de 5.1, R1, R7, R9–R11 y R14.
Los tres son globales: un solo archivo que falla bloquea la salida de
todas las obras de la corrida, no solo la de la obra que falló. Decisión
deliberada de máxima rigidez: se sacrifica disponibilidad a cambio de
forzar la corrección upstream del defecto.

R16 [cierra #8, gate de formato de archivo] DADO cualquier archivo
   sometido a ingesta como parte de una corrida de PR
   CUANDO ese archivo no es un .xlsx binario válido (extensión
   incorrecta, corrupto, o ilegible)
   ENTONCES el sistema DEBE abortar la corrida de ingesta completa para
   todas las obras de inmediato, no producir salida para ninguna obra, y
   reportar la falla. La corrida no se reanuda hasta que todos los
   archivos del batch sean .xlsx válidos.

R17 [cierra #8, ruido estructural] DADO un archivo PR que ha pasado R16
   CUANDO el archivo se parsea
   ENTONCES el sistema DEBE descartar incondicionalmente las filas 1 a 5
   y las columnas A a S, sin inspeccionar ni validar su contenido, y
   establecer la fila 6 como fila de encabezado y la columna T como
   origen estructural (offset 0) del dominio de datos.

R18 [cierra #8, gate de validación literal de encabezado] DADO un
   archivo PR que ha pasado R16 y R17
   CUANDO la fila 6, desde la columna T en adelante, se compara contra
   el esquema crudo completo declarado en la Sección 3
   ENTONCES el sistema DEBE ejecutar una comparación de string literal
   exacta, sensible a mayúsculas y espacios, contra cada encabezado
   esperado, en orden posicional fijo; ante cualquier discrepancia —
   encabezado extra, faltante, reordenado o con variación cosmética
   (mayúsculas, espacios al inicio/final) — para cualquier archivo de
   cualquier obra, el sistema DEBE abortar la corrida de ingesta completa
   para todas las obras, no producir salida, y registrar en el log la
   obra, archivo y encabezado específico en discrepancia, para
   corrección upstream. No se aplica normalización (5.1) al texto del
   encabezado antes de esta comparación.

### 5.4 Contrato de Semántica y Validación de FECHA — Campo `fecha`  [cierra #7]

Gobernanza: FECHA es un campo de tipo fecha, no de texto; no participa del pipeline de
normalización 5.1 y no genera contraparte _norm. Su semántica es invariante respecto a
ESTATUS_C_CLOUD y a cualquier ruteo de salida.

R19 [cierra #7, semántica] DADO una fila de PR sobreviviente al gate de R14 (USAR=="SI")
   CUANDO se interpreta el valor crudo de la columna FECHA
   ENTONCES el sistema DEBE tratarlo como la fecha real de inicio de la actividad en una
   ubicación física específica (resoluble vía N1..N5 / id_tipologia), de forma invariante
   respecto a ESTATUS_C_CLOUD o a cualquier ruteo de salida. Este es un hecho de origen sobre
   el dato crudo; cómo el grano de agregación (R4) lo consume es materia de #6, no de esta
   regla.

R20 [cierra #7, passthrough Tabla 3] DADO que la Tabla 3 consume columnas crudas en su
   totalidad (5.1, R12)
   CUANDO se puebla su esquema
   ENTONCES el sistema DEBE poblar el campo fecha exclusivamente con el valor crudo de FECHA
   de esa fila, sin transformación ni agregación, al grano de fila original — no al grano de
   R4.

R21 [cierra #7, exclusión Tabla 1] DADO que Append PR Finalizado representa actividades para
   las que el consumidor no necesita una fecha de inicio (decisión de negocio confirmada)
   CUANDO se define su esquema
   ENTONCES el sistema DEBE excluir el campo fecha. Exclusión de esquema, no de semántica —
   FECHA sigue siendo válida en el dato subyacente. Verificado contra el riesgo cruzado de
   "ritmos históricos" (mandato de módulo PR): ese cálculo se resuelve fuera de esta tabla, sin
   conflicto.

R22 [cierra #7, validación activa, no-lossy, homologada a R2] DADO una fila sobreviviente a
   R14 cuyo FECHA es nulo, blanco, o no castea limpiamente a tipo fecha
   CUANDO se ingesta y valida la fila
   ENTONCES el sistema DEBE enrutarla a la cola de anomalías, no generar valor de fecha para
   ella, y continuar procesando el resto del archivo sin abortar la corrida. Defensiva:
   inspección empírica (Alberto, S8) confirmó que bajo USAR=="SI" esto no ocurre hoy — sin
   nulos, sin blancos, sin artefactos de época Excel, sin centinelas de futuro lejano, formato
   DIA-MES-AÑO consistente. Esta regla cubre una violación futura del supuesto, no un problema
   observado.

### 5.5 Contrato de Grano y Conteo [cierra #6]

Gobernanza: estos contratos se ejecutan después de R8 (filtro MATCHED/NO_MATCH),
después de R14 (gate USAR) y después de 5.1 (normalización). Solo las filas con
tipologia_status = MATCHED participan en la agregación.

Precondición estructural declarada (invariante de origen):
La combinación (obra_norm, id_actividad, N1_norm..N5_norm) aparece exactamente
una vez en un archivo PR después del gate de R14. Una fila superviviente equivale
a una unidad física de ejecución. El pipeline depende de este invariante sin
verificarlo activamente. Si se viola, count_unidades sobrecuenta silenciosamente.

Definición de "unidad": unidad física de ejecución hiper-localizada en el mundo
real (departamento, ciclo, unidad de ejecución). Su ubicación está dada por
(obra_norm, N1_norm..N5_norm); su tipología por id_tipologia; su actividad por
id_actividad; su fecha de inicio por fecha (Tabla 2 únicamente).

R4-T1 [cierra #6, grano Tabla 1, reemplaza R4]
DADO el set consolidado multiobra filtrado exclusivamente a filas con
   tipologia_status = MATCHED
CUANDO se agregan los counts para Append PR Finalizado (Tabla 1)
ENTONCES el sistema DEBE agrupar por (obra_norm, id_actividad, id_tipologia),
   produciendo count_unidades = número de filas en el grupo.
   id_tipologia es UInt64 no nulo por construcción (las filas NO_MATCH fueron
   excluidas en R8 antes de esta agregación).

R4-T2 [cierra #6, grano Tabla 2]
DADO el set consolidado multiobra filtrado exclusivamente a filas con
   tipologia_status = MATCHED
CUANDO se agregan los counts para Append PR Programado (Tabla 2)
ENTONCES el sistema DEBE agrupar por (obra_norm, id_actividad, id_tipologia, fecha),
   produciendo count_unidades = número de filas en el grupo.
   id_tipologia es UInt64 no nulo por construcción. fecha es Date no nulo por
   construcción (R22 garantiza que ninguna fila con FECHA nulo o malformado
   llega a esta etapa). La distribución de count_unidades por fecha es el
   resultado correcto e intencional: unidades de la misma tipología con fechas
   de inicio distintas generan filas separadas. No se colapsan.

Nomenclatura de salida: el campo se llama `fecha` (no `fecha_inicio`), confirmado por Alberto
en S8.

### 5.6 Contrato de Resolución de Versión Vigente — Archivo PR [cierra #11]

Gobernanza: este contrato resuelve, para cada fila declarada en la hoja de
programas vigentes (§3), cuál archivo físico es el vigente, antes de que
ese archivo entre a R16–R18 (gate estructural) o a cualquier otro
procesamiento.

R23 [cierra #11, mecanismo primario] DADO una fila (ruta_archivo,
   nombre_programa) declarada en la hoja de programas vigentes
   CUANDO el sistema resuelve el archivo vigente
   ENTONCES DEBE recorrer recursivamente el árbol bajo ruta_archivo,
   identificar las carpetas cuyo nombre coincide con el patrón
   <PREFIJO>_SEM_<AAAAMMDD>, extraer la fecha codificada de cada una, y
   dentro de esas carpetas buscar archivos cuyo nombre coincide de forma
   exacta (sensible a mayúsculas y espacios, homologado a R18) con
   nombre_programa. DEBE seleccionar como vigente el archivo bajo la
   carpeta de fecha máxima.

R24 [cierra #11, borde: carpetas no conformes] DADO una carpeta
   encontrada durante el recorrido cuyo nombre no coincide con el patrón
   <PREFIJO>_SEM_<AAAAMMDD>
   CUANDO el sistema recorre el árbol
   ENTONCES DEBE ignorarla como candidata de fecha (no extraer fecha de
   ella) y continuar el recorrido sin abortar.

R25 [cierra #11, borde: cero coincidencias] DADO que el recorrido para
   una fila declarada no produce ningún archivo coincidente en ningún
   punto del árbol
   CUANDO se ejecuta la resolución
   ENTONCES el sistema DEBE abortar la corrida completa multiobra, no
   producir salida para ninguna obra, y registrar en el log la fila
   específica en falla. Homologado a R16/R18: precisión de origen es un
   hard constraint, no una anomalía de fila.

R26 [cierra #11, borde: empate] DADO que el recorrido produce más de un
   archivo coincidente bajo la fecha máxima
   CUANDO se ejecuta la resolución
   ENTONCES el sistema DEBE abortar la corrida completa multiobra (misma
   postura que R25) y registrar la fila ambigua, SIN recurrir al
   timestamp de modificación del sistema de archivos (mtime) como
   criterio de desempate — riesgo de mtime explícitamente rechazado por
   inconsistencia bajo sincronización de Google Drive.

### 5.7 Contrato de Frontera de Salida — Staging Polars / BigQuery [avance parcial de #10]

Estado: PARCIALMENTE DEFINIDO. Resuelta la frontera de salida y el split
de motor (Opción C, S11); resuelta la fuente de entrada (Drive, D-14.4,
S12); resuelto el entorno de ejecución (#14, 5.8, S12); cerrado el
sub-item (d) (R27/§5.9 + atomicidad D-14.5, S13). Pendiente dentro de
#10: mecanismo de conexión Sheets (c), motor del artefacto residual de
R11.

Decisiones tomadas (S11):

D-10.1 [landing de salida] Las tres tablas de salida (T1, T2, T3) aterrizan
   en BigQuery. PQ (transitorio, Modelo 2) y la futura capa Sheets/Workspace
   se conectan a BigQuery. PQ no es parte de la arquitectura final: es
   andamiaje para mantener vivo el MRP actual (CÓMPUTO) hasta la migración
   final, momento en el cual se decomisiona. El driver de esta elección es
   la latencia (§2): pull desde BigQuery vía conector, no recálculo local.

D-10.2 [qué extrae PQ] PQ extrae las tablas de salida ya procesadas (las
   vistas de BigQuery), no los archivos PR crudos. La lógica de limpieza/
   normalización/join/agregación ya no vive en M; vive en el pipeline
   GCP+Polars+BigQuery.

D-10.3 [split de motor — Opción C] Polars posee todas las reglas a nivel de
   fila hasta R8 inclusive: normalización (5.1), gate USAR (R14), hash de
   identidad (R1, R6), join de tipología (R7), disposición no-lossy (R8).
   Polars stagea a BigQuery una tabla a nivel de fila ya matcheada
   (stg_matched: solo filas MATCHED, USAR-gated, normalizada) y, por
   separado, el artefacto de reconciliación de R8 (recon_nomatch).
   BigQuery posee la agregación (R4-T1, R4-T2), el ruteo de vigencia
   (R9/R10) y las vistas de servicio de las tres tablas:
   - T1: vista de agregación, grano R4-T1, filtro R9.
   - T2: vista de agregación, grano R4-T2, filtro R10.
   - T3: vista a grano de fila, passthrough crudo (R15, R20), filtro
     R9∪R10.
   Justificación del corte: las reglas de "muerte de fila" no-lossy
   (R8/R11/R14/R22) permanecen consolidadas en Polars — el corte cae entre
   R6 y R7 a nivel conceptual, manteniendo la trazabilidad de "dónde muere
   una fila" en un solo motor. Solo migran a SQL las reglas de conteo
   (R4) y de ruteo (R9/R10), que no son reglas de muerte de fila. Se
   evaluó y descartó la Opción A (join R7 + disposición R8 en BigQuery)
   por dispersar la filosofía no-lossy entre dos motores; se evaluó la
   Opción B (todo en Polars, BigQuery solo almacena) y se prefirió C por
   legibilidad del contrato de servicio como vistas SQL versionadas, sin
   sacrificar la consolidación de reglas de fila.

Implicancia de motor para artefactos diferidos: bajo Opción C, recon_nomatch
(R8) es producido por Polars. El artefacto residual de R11 (estatus no
reconocido) queda implícitamente del lado del ruteo R9/R10 — que vive en
BigQuery — y por tanto su motor NO está resuelto por esta decisión. Su
esquema y destino siguen diferidos a Bloque 3 (consistente con R8/R11
originales); la asignación de motor del artefacto R11 queda explícitamente
abierta, no cerrada por defecto.

Nota S13 (R27, §5.9): stg_matched y recon_nomatch incluyen además
run_date:Date, estampado por el Job al momento de la escritura (D-14.5);
las vistas T1, T2 y T3 heredan esta columna por passthrough.

Nota de cierre S13 — atomicidad de consumo (cierra parcialmente #10,
sub-item (d)): confirmado que la atomicidad de D-14.5 (escritura única
WRITE_TRUNCATE, diferida hasta corrida exitosa sin abort) es garantía de
consumo suficiente para T1/T2/T3. Un consumidor que lea durante la
ventana de escritura observará o el snapshot completo de la semana
anterior, o el snapshot completo de la semana actual — nunca una mezcla
parcial. No se requiere regla ni artefacto adicional. D-14.5 ya cubre
esta garantía en su forma actual.

D-10.4 [partición de datasets — staging oculto / serving expuesto, S14,
cierre total de #10] El landing de BigQuery declarado en D-10.1 se precisa
en dos datasets físicamente separados: un dataset de staging, destino
exclusivo de la escritura WRITE_TRUNCATE del Job (D-14.5) — contiene
stg_matched y recon_nomatch, nunca expuesto a analistas humanos —, y un
dataset de servicio, que contiene las vistas T1, T2 y T3 (R4-T1, R4-T2,
R15), declaradas como vistas autorizadas (BigQuery authorized views) contra
el dataset de staging. Esta partición es la que permite que el rol IAM de
lectura de los analistas individuales se otorgue exclusivamente sobre el
dataset de servicio (R29, §5.10), sin requerir ni otorgar acceso transitivo
al dataset de staging. La vista del artefacto residual de R11 (R30, §5.10)
reutiliza este mismo dataset de staging como su ubicación, por la misma
razón. No modifica D-10.1–D-10.3 ni D-14.5: precisa el destino físico
dentro de BigQuery, no el motor que escribe ni la mecánica de escritura.

### 5.8 Contrato de Entorno de Ejecución — Polars en PR [cierra #14]

Estado: DEFINIDO. Resuelve el entorno de ejecución del pipeline Polars de PR
dentro de GCP, su modelo de disparo, perfil de carga, autenticación a Drive,
y la mecánica de escritura a BigQuery. Resuelve además, como consecuencia,
la bifurcación de fuente de entrada (Drive vs GCS) que estaba abierta dentro
de #10.

D-14.1 [entorno de ejecución] El pipeline Polars de PR se ejecuta como un
   Cloud Run Job — ejecución de tipo run-to-completion, sin servidor HTTP —,
   no como un Cloud Run Service. Se descartan explícitamente: GCE (recurso
   ocioso e injustificado para una carga semanal de pocos minutos), Dataflow
   (sin caso de streaming ni escala que lo justifique), y Cloud Run Service
   (impondría un contrato request/response sobre una tarea batch no
   interactiva, sin necesidad).

D-14.2 [modelo de disparo] El Job se invoca bajo un esquema programado
   (scheduled), mediante Cloud Scheduler invocando la Cloud Run Jobs Execute
   API — no on-demand, no event-driven por llegada de archivo. Día y hora
   exactos son un parámetro de despliegue (indicativo: lunes 01:00 AM), no
   una regla de negocio del sistema.

D-14.3 [perfil de ejecución y dimensionamiento] Validado contra el peor caso
   declarado: hasta 6–7 obras activas, hasta 4 archivos PR activos por obra
   (según vigencia, hoja de programas vigentes), cada archivo con
   aproximadamente 8.000–12.000 filas. Peor caso ≈ 24 archivos × ≤12.000
   filas ≈ ≤288.000 filas por corrida semanal. Carga trivial para Polars
   (cómputo en segundos); las cotas de recursos por defecto de Cloud Run
   Jobs (CPU, memoria, timeout) son suficientes sin requerir un tier de
   recursos elevado.

D-14.4 [autenticación a Drive] Tanto los archivos PR por obra (estructura de
   carpetas recorrida por R23–R26, bajo ruta_archivo) como la hoja de
   programas vigentes (§3) residen en la misma Shared Drive. La service
   account del Job se agrega como miembro directo de esa Shared Drive con
   acceso de lectura. No se requiere domain-wide delegation ni
   sincronización forzada de los archivos fuente a GCS: el Job accede a
   ambos directamente vía la Drive API. Esta decisión resuelve, dentro de
   #10, la bifurcación de ubicación de fuentes de entrada a favor de Drive
   (no GCS).

D-14.5 [mecánica de escritura a BigQuery — disposición y atomicidad] Cada
   corrida semanal reemplaza por completo el contenido de stg_matched y
   recon_nomatch — snapshot de estado actual, no un log histórico
   acumulativo —, decisión consistente con que PR alimenta decisiones
   operativas vigentes (despacho, detención de despacho, lot sizing), no
   análisis histórico. El reemplazo es diferido y atómico respecto de los
   gates de abort ya existentes: el Job procesa y valida las obras
   completas del batch en memoria — pipeline 5.1, gate R14, hashes R1/R6,
   join R7, disposición R8, y los gates estructurales y de vigencia R16–R18
   y R23–R26 — antes de ejecutar una única escritura a BigQuery
   (WRITE_TRUNCATE sobre stg_matched y recon_nomatch). La escritura se
   ejecuta solo si la corrida completa no disparó ningún abort. Si R16,
   R18, R25 o R26 disparan en cualquier punto del batch, la escritura nunca
   se alcanza, y stg_matched/recon_nomatch de la semana anterior quedan
   intactos sin modificación — CÓMPUTO continúa operando sobre el último
   snapshot exitoso. Esta decisión no modifica R16, R18, R25 ni R26; define
   por primera vez qué significa concretamente "no producir salida" ahora
   que el destino es una tabla persistente y no un archivo generado una
   sola vez.
   Consecuencia aceptada explícitamente (S12): dado que R16/R18 abortan la
   corrida completa ante la falla estructural de un solo archivo de
   cualquier obra, un defecto de encabezado en una sola obra congela el
   refresco de datos de las 6–7 obras esa semana — no solo el de la obra
   con el archivo defectuoso. Alberto evaluó esta consecuencia
   explícitamente y la aceptó sin solicitar cambios a R16–R18/R25–R26.


### 5.9 Contrato de Trazabilidad de Snapshot — run_date [cierra parcialmente #10, sub-item (d)]

Estado: DEFINIDO. Resuelve la señal de frescura que un consumidor de
T1/T2/T3 necesita para saber qué corrida semanal produjo el snapshot
que está leyendo. Distinto de la semántica de negocio de fecha (R19–R22):
run_date es metadato operacional del Job, no dato de la actividad.

R27 [cierra parcialmente #10, sub-item (d) — señal de frescura]
DADO que stg_matched y recon_nomatch son las tablas físicas escritas por el
   Job de Polars en cada corrida semanal (D-14.5)
CUANDO el Job ejecuta la escritura WRITE_TRUNCATE de una corrida completa y
   exitosa (sin abort en ningún gate — R16, R18, R25, R26)
ENTONCES el sistema DEBE estampar en cada fila de ambas tablas una columna
   run_date:Date, cuyo valor es la fecha de ejecución del Job para esa
   corrida — un único valor, idéntico en cada fila de stg_matched y de
   recon_nomatch para ese snapshot, independiente de la fecha de vigencia
   resuelta por obra en R23–R26 (que puede diferir entre obras y no
   participa en esta columna).
   Las vistas T1, T2 y T3 en BigQuery DEBEN heredar run_date mediante
   passthrough directo desde stg_matched, sin transformación.
   En las agregaciones de R4-T1 y R4-T2, run_date DEBE incluirse como
   miembro adicional del GROUP BY; dado que su valor es constante dentro
   de cualquier snapshot, su inclusión no fragmenta el grano declarado
   (obra_norm, id_actividad, id_tipologia[, fecha]).
   Distinción terminológica obligatoria: run_date (metadato operacional,
   constante por snapshot, "cuándo corrió el Job") es una columna distinta
   de fecha (dato de negocio, R19–R22, varía por fila en Tabla 2 y Tabla 3).
   Ambas columnas Date coexisten en Tabla 2 y Tabla 3 sin relación entre
   sí; no deben confundirse en ningún consumo posterior.
   Granularidad: Date, no Timestamp — D-14.5 garantiza que una escritura
   solo ocurre tras una corrida completa y exitosa; no existe escenario de
   colisión intra-día entre un intento fallido y uno exitoso, porque el
   fallido nunca alcanza BigQuery.

### 5.10 Contrato de Consumo Externo — Sheets / Power Query / Aislamiento de Datasets [cierra #10 en su totalidad]

Estado: DEFINIDO. Resuelve los dos sub-items restantes de #10: (c) el
mecanismo de conexión de Sheets/Power Query, y el motor + destino del
artefacto residual de R11. Con esta subsección, #10 queda cerrado por
completo.

Gobernanza: este contrato resuelve cómo los analistas humanos y el proceso
CÓMPUTO actual (Excel + Power Query) leen T1, T2 y T3 desde BigQuery, y cómo
se garantiza que los artefactos de excepción no-lossy — stg_matched (vía
recon_nomatch) y el residual de R11 — permanezcan inaccesibles a los
analistas sin comprometer la capacidad de las vistas de servicio de leer
internamente sus fuentes.

R28 [cierra parcialmente #10, sub-item (c) — mecanismo de conexión]
DADO un analista humano que requiere consultar T1, T2 o T3
CUANDO accede a los datos desde su herramienta de consumo (Google Sheets vía
   Connected Sheets para T3; Power Query en Excel para T1 y T2)
ENTONCES el sistema DEBE exponer la vista correspondiente exclusivamente en
   modo pull: la herramienta de consumo se conecta directamente a la vista
   de BigQuery, sin ningún componente intermedio — agente, servicio o
   script — que materialice o escriba los datos en la hoja; la
   actualización (refresh) es ejecutada manualmente por el analista, nunca
   de forma automática ni programada; la autenticación se ejecuta bajo la
   identidad individual de Google del propio analista, sin credenciales
   compartidas ni service account intermedia.
   No introduce paso de escritura adicional sobre el output de PR: D-14.5
   no se modifica, el único sink del Job sigue siendo BigQuery.

R29 [cierra parcialmente #10, sub-item (c) — partición de datasets y
   aislamiento de acceso]
DADO que T1, T2 y T3 son vistas de BigQuery definidas sobre stg_matched (y,
   para la disposición no-lossy, sobre recon_nomatch)
CUANDO se diseña el modelo de acceso de lectura para analistas individuales
ENTONCES el sistema DEBE alojar stg_matched y recon_nomatch en un dataset
   de staging oculto, y las vistas T1, T2 y T3 en un dataset de servicio
   separado, declarado como vista autorizada (BigQuery authorized views)
   contra el dataset de staging. El rol IAM de lectura otorgado a cada
   analista — identidad individual de Google, administrado exclusivamente
   por el dueño del sistema — se concede únicamente sobre el dataset de
   servicio (a nivel de dataset completo); bajo ninguna circunstancia se
   otorga acceso, directo o transitivo, al dataset de staging.
   Enmienda explícita (S14): se retira la restricción provisional "los
   analistas nunca acceden a T1 y T2". Los analistas SÍ tienen lectura
   sobre T1 y T2 (vía Power Query) además de T3 (vía Connected Sheets); el
   aislamiento se redirige exclusivamente a stg_matched y recon_nomatch, no
   a T1/T2.
   Consecuencia aceptada explícitamente: bajo este modelo el analista puede
   ver unidades agregadas de obras distintas a la suya (visibilidad
   cross-obra) en T1/T2; no se considera bloqueante. Filtrado por obra, si
   se requiere, pertenece al dashboard/CÓMPUTO consumidor, no a PR.

Amendment a R8 (destino fino de recon_nomatch, S14): recon_nomatch reside
   en el dataset de staging oculto (R29) y no es accesible a los analistas.
   La distribución de su contenido — filas falladas y acciones correctivas
   upstream — es responsabilidad exclusiva del dueño del sistema, gestionada
   manualmente fuera del mecanismo de consumo automatizado de PR. No es
   excepción a la postura no-lossy (la fila se conserva en staging); es una
   decisión de canal de distribución, no de retención.

R30 [cierra el motor y destino del artefacto residual de R11]
DADO una fila MATCHED en stg_matched cuyo ESTATUS_C_CLOUD_norm no pertenece
   a los conjuntos declarados en R9 ni en R10
CUANDO se define el mecanismo de captura del residual de R11
ENTONCES el sistema DEBE exponerla mediante una vista de BigQuery —simétrica
   a T1/T2, con condición de membresía complementaria a R9 ∪ R10— declarada
   dentro del dataset de staging oculto, no en el dataset de servicio, de
   forma que herede el mismo perímetro de aislamiento que recon_nomatch y
   quede excluida automáticamente del grant IAM de analistas otorgado a
   nivel del dataset de servicio (R29).
   La distribución de su contenido — filas con estatus no reconocido y
   acciones correctivas upstream — es responsabilidad exclusiva del dueño
   del sistema, gestionada manualmente, en simetría exacta con el
   tratamiento de recon_nomatch (R8 enmendada S14).
   No modifica D-10.3: el motor de evaluación de membresía de R9/R10
   permanece en BigQuery; esta vista no introduce lógica nueva en Polars.

### 5.11 Contrato de Presencia — Tabla Mapeo de Tipologías [cierra #13]

Gobernanza: a diferencia del archivo PR (§5.6), la Tabla Mapeo de Tipologías no
tiene mecanismo de vigencia por versión — es un archivo único por obra,
mantenido y sobrescrito in situ por el analista, sin estructura de carpetas
por fecha. El problema que resuelve esta subsección no es "cuál versión es
la vigente" sino "¿existe, de forma inequívoca, el archivo correcto para
esta obra?". Estos chequeos se ejecutan antes de R7 (join) y, en el caso de
R31, antes de cualquier otro procesamiento del batch.

R31 [cierra #13, gate de accesibilidad de carpeta global, homologado a R16]
   DADO que el sistema necesita leer el contenido de la carpeta global
   declarada (Tipologias Obras) para resolver el mapeo de cualquier obra
   CUANDO se inicia una corrida y esa carpeta resulta inexistente, eliminada,
   o sin permisos de lectura para la identidad de servicio del Job
   ENTONCES el sistema DEBE abortar la corrida de ingesta completa para
   todas las obras de inmediato, no producir salida para ninguna obra, y
   registrar en el log de ejecución la causa específica de inaccesibilidad
   (ruta declarada, tipo de error). Este chequeo precede a la resolución de
   mapeo por obra (R32) y a cualquier otro procesamiento del batch.

R32 [cierra #13, gate de presencia por obra, obra-acotado] DADO que la
   carpeta global ha pasado el chequeo de R31
   CUANDO el sistema busca, para una obra cuyo obra_norm ha sido derivado
   (R3+R5), un archivo dentro de la carpeta cuyo nombre normalizado
   (pipeline 5.1) coincida con `{obra_norm}_mapeo_tipologias`
   ENTONCES, SI la búsqueda no produce exactamente una coincidencia —ya sea
   cero coincidencias o más de una— el sistema DEBE excluir todas las
   macro_partidas de esa obra de la corrida actual (no producir T1, T2 ni
   T3 para esa obra esta semana), continuar procesando el resto de las
   obras sin abortar la corrida multiobra completa, y registrar en el log
   de ejecución la obra específica excluida y el motivo (cero coincidencias
   vs. coincidencia ambigua). No genera artefacto persistido de excepción —
   el log de ejecución del Job es el único registro.
   Divergencia deliberada respecto a R25/R26: ese par homologa "cero
   coincidencias o empate" a un abort global, aceptando que un solo archivo
   defectuoso congele a todas las obras (D-14.5). R32 homologa el mismo tipo
   de evento, exclusivamente para el artefacto de mapeo, a una exclusión
   acotada a la obra afectada — las demás obras no se ven impactadas.
   Si la búsqueda produce exactamente una coincidencia, el sistema procede a
   leer y validar su contenido bajo el contrato ya existente (R6, R6.1, R7);
   esta subsección no modifica esas reglas.

## 6. AUDITORÍA — GRIETAS ABIERTAS

"AUDITORÍA es descriptiva; el LEDGER (§8) es autoritativo para estado" 

~~**A. Identidad y llaves (núcleo).** Regla de construcción de `id_actividad` no declarada. Llave de join de `id_tipologia` contra mapeo de tipologías no declarada (¿5 niveles, subconjunto, ACTIVIDAD?). Llave de `obra` inexistente en todo esquema declarado, pese a que el output es "por cada obra".~~

~~**B. Filtrado y estatus.** Dos columnas de estatus (`ESTATUS`, `ESTATUS C.CLOUD`): cuál gobierna el filtro no está declarado. Resuelto: `ESTATUS C.CLOUD` gobierna en exclusiva (5.2, R9–R13). Filas fuera de {FINALIZADA, Nueva}: destino no declarado (riesgo de drop silencioso). Resuelto: cola de reconciliación no-lossy, homologada a R8 (R11). Semántica de `USAR` (valores y efecto). Resuelto: gate destructivo upstream de toda la tubería, comparación contra valor crudo sin _norm, sin auditoría — excepción explícita a la postura no-lossy (R14); exclusión de la columna usar en el esquema de Tabla 3 (R15).~~
 
**C. Agregación.** ~~"Unidad" no definida; nivel de agrupación del count no declarado — ver #6.~~ Resuelto en #6 (S9): "unidad" definida y grano fijado (R4-T1, R4-T2, §5.5). ~~`FECHA` cambia de significado según estatus (Programado conserva fecha, Finalizado la descarta) sin regla declarada.~~ Resuelto en #7: la hipótesis de que FECHA cambia de significado por estatus fue evaluada y descartada — su semántica es invariante (R19); Tabla 1 la excluye por decisión de esquema, no por vaciamiento semántico (R21); Tabla 3 la consume sin transformación (R20); validación activa ante nulo/malformado homologada a R2 (R22). ~~Pendiente, ligado a #6: el mecanismo de derivación de fecha en Tabla 2 bajo el grano corregido de R4.~~ Resuelto en #6 (S9): R4-T2 fija el grano de Tabla 2 con fecha.
 
~~**D. Frontera arquitectónica.** Tensión entre "migrar todo a GCP+Polars" y "consumir vía PQ/Sheets". Avance parcial S11 (5.7): outputs aterrizan en BigQuery (D-10.1); PQ extrae tablas procesadas, no crudos, y es transitorio (D-10.2); split Polars/BigQuery vía Opción C (D-10.3). El mecanismo que resuelve "versión vigente" fuera de M — resuelto para archivos PR en #11 (R23–R26, S11). Entorno de ejecución de Polars (#14), que bloqueaba la decisión de entrada — resuelto en #14 (S12, 5.8): Cloud Run Jobs, disparo programado vía Cloud Scheduler, perfil de carga validado, autenticación directa a Shared Drive (D-14.1–D-14.4). Bifurcación de fuentes de entrada (Drive vs GCS) resuelta a favor de Drive (D-14.4). Sub-item (d), interfaz concreta de CÓMPUTO contra el landing point — cerrado en S13: corrección de alcance, PR define solo su contrato de frontera de consumo propio (R27, §5.9, atomicidad D-14.5). Sub-item (c), mecanismo técnico de conexión Sheets, y motor del artefacto residual de R11 — cerrados en S14.~~
> Cerrado en S14: #10 resuelto en su totalidad. Frontera de salida
> (BigQuery + Opción C, D-10.1/2/3), fuente de entrada (Drive, D-14.4),
> entorno de ejecución (Cloud Run Job, D-14.1–D-14.5), señal de frescura
> (R27, §5.9), atomicidad de consumo (§5.7), mecanismo de conexión
> Sheets/PQ + partición de datasets staging/serving (R28/R29, §5.10,
> D-10.4), y motor + destino del artefacto residual de R11 (R30, §5.10) —
> todos resueltos. Sin items abiertos restantes en esta grieta.

~~**G. Resolución de vigencia — Tabla Mapeo de Tipologías.** El mecanismo de R23–R26 resuelve vigencia exclusivamente para archivos PR (hoja de programas vigentes, §3). La Tabla Mapeo de Tipologías se declara en §3 como "un archivo por obra", sin estructura de carpeta ni mecanismo de versionado definidos. Pendiente: ¿existe versionado real (carpetas por fecha, análogas a R23) o es un archivo verdaderamente único y estático por obra, sin ambigüedad de "vigente"? Si hay versionado, ubicación de carpeta y patrón de nombre no declarados. [G] #13.~~
> Cerrado en S15. Confirmado estructuralmente (no por analogía): no hay
> versionado, archivo único y estático por obra, sin ambigüedad de
> "vigente". El problema real era de presencia/identificación, no de
> selección temporal — resuelto por R31 (accesibilidad de carpeta global) y
> R32 (presencia y match por obra). Ver §5.11.
 
~~**E. Contrato de consumo y grafo.** El "MRP Excel" consumidor no está identificado contra el grafo del sistema (¿PLANTIR? ¿módulo sin nombre? ¿legado externo?). Riesgo: el consumidor actual espera 13 columnas a nivel de fila; el nuevo output entrega tablas agregadas (count). Si necesita detalle por ubicación, la agregación rompe el contrato.~~
> Cerrado S10: consumidor = CÓMPUTO (módulo declarado). T1 y T2 entregan counts agregados; CÓMPUTO solo necesita (obra, id_tipologia, id_actividad, count_unidades [+ fecha en T2]) — contrato no roto. T3 entrega reporte humano directo con esquema enmendado.
 
~~**F. Precondiciones estructurales.** El artefacto "5 filas + 1 columna vacías" debe entrar al contrato de ingesta con su caso borde: ¿garantizado para todas las obras o hay desviaciones que rompen el lector? Reglas de normalización propias de PR no declaradas (¿reutilizan las de MIDAS o son propias?). Resuelto: layout universal sin excepciones para ninguna obra; filas 1–5 y columnas A–S ruido no validado, descarte incondicional; fila 6 = encabezado, columna T = origen; formato exclusivo .xlsx; validación literal completa de encabezados con abort global de pipeline ante cualquier discrepancia (R16–R18). Nota: la segunda parte de esta grieta (reglas de normalización propias de PR) ya estaba resuelta desde S2 — pipeline 5.1 es propio de PR, no reutiliza MIDAS — y nunca se tachó; queda corregido aquí.~~

 
## 7. PLAN DE DESARROLLO POR BLOQUE
 
Orden por dependencia. Gating: no se genera un archivo si su bloque tiene PENDIENTE crítico abierto.
 
- **Bloque 0 — Contrato de consumo y posición en grafo** ← **CERRADO (S10)**. Resolvió E.

- **Bloque 1 — Objetivo + Requerimientos** (archivo) ← **DESBLOQUEADO (S10)**, no iniciado. Objetivo casi listo; requerimientos dependían de B0, ya cerrado.

- **Bloque 2 — Domain** (archivo) ← **CERRADO (S9)**. Resolvió A, B, C, F.

- **Bloque 3 — Arquitectura + Stack** (archivo) ← **CERRADO (S15)**. #11 cerrado (S11, R23–R26). #14 cerrado (S12, 5.8). #10 cerrado en su totalidad (S11, 5.7; S12, D-14.4; S13, R27/§5.9; S14, R28–R30/§5.10 + D-10.4). #13 cerrado (S15, R31–R32/§5.11). Resuelve D y G en su totalidad. Sin PENDIENTE crítico restante en este bloque.

- **Bloque 4 — Roadmap por fases** (archivo) ← **EN CURSO (S16–S18)**. Depende de B2 + B3, ambos
  cerrados. Estructura fijada: dos niveles (roadmap.md Fase/Paso descriptivo, development.md atómico
  con Status/Done Criteria/Done Evidence). Partición de Fases gobernada por el corte de motor D-10.3,
  no por conteo arbitrario. Fase 0 cerrada (roadmap + 11 tareas atómicas). Fase 1 descompuesta a nivel
  atómico (8 Pasos). Fase 2 (GCP Infrastructure Provisioning) descompuesta a nivel atómico (5 Pasos,
  20 tareas); frontera de dataset con Fase 4 sobre D-10.4 resuelta (container vs. contenido). Fase 3
  (Live Drive File Resolution) descompuesta a nivel atómico (4 Pasos, 17 tareas): 3.1 imagen de
  producción + redespliegue del Job (resuelve el gap diferido de Fase 2), 3.2 Precondition Gate
  concurrente (R31 ∥ R23–R26, OR-abort/AND-success, retry-before-abort), 3.3 resolución de mapeo por
  obra (R32, exclusión acotada), 3.4 verificación en vivo + contrato de salida hacia Fase 4.   
  Fase 4 (BigQuery Aggregation & Service Views — lado productor) descompuesta a nivel de roadmap
  (Paso) únicamente: 5 Pasos (4.1 vistas de agregación T1/T2 en pr_serving — R4-T1/R9, R4-T2/R10,
  run_date como miembro del GROUP BY; 4.2 vista desnormalizada T3 en pr_serving — passthrough
  crudo R15/R12/R20, filtro R9∪R10; 4.3 vista residual R30 row-level en pr_staging — complemento
  de R9∪R10, columnas crudas identificadoras, aislada del serving; 4.4 authorized-view link
  serving→staging para T1/T2/T3, R30 excluida; 4.5 verificación en vivo terminal + contrato de
  salida hacia Fase 5). Sin descomposición atómica: bajo la decisión JIT (ver abajo), los
  atómicos de Fase 4 se proponen en ejecución, no upstream. Auditoría de cobertura, antes de
  cerrar el roadmap de Fase 4: el contrato de salida row-level de Fase 1 (Paso 1.8) estaba
  etiquetado [CLOSED] sin enumerar columnas — cerrado en esta sesión como frame local de 9
  columnas (1.8.1 reescrito); las transformaciones de frontera (run_date + cast STRING de ids)
  reasignadas a 3.4.3 (Fase 3), su lugar correcto; fixture 1.1.4 reconciliada al frame de 9
  columnas. Paso terminal 4.5 (verificación en vivo) homólogo a 2.5/3.4. Decisión metodológica
  mayor (S19): la descomposición atómica pasa a JIT/incremental desde Fase 4 inclusive; Bloque 5
  disuelto; solo Fases 0–3 conservan atómicos congelados como referencia. Con Fase 4 a nivel
  roadmap, resta únicamente Fase 5 (nivel Paso) para cerrar roadmap.md.

- **Bloque 6 — CLAUDE.md** (archivo) ← **ACTIVO (S16)**, no condicional. Enmienda: se retira la
  condicionalidad declarada en S15 — la arquitectura sin Colab y el contrato single-admin de PR sí
  exigen gobernanza propia, confirmado en S16. Hereda el esqueleto de MIDAS (MODE, Task Protocol, Git
  Practices, Commit/Merge gates, Directory Access Restriction), adaptado a runtime Cloud Run Job +
  Scheduler, sin Colab. Archivo standalone en la raíz del repo, NO embebido en este spec (mantiene
  liviano el contexto del agente). Secuenciamiento: después de que Bloque 4 complete su descomposición
  atómica (su sección de Checkpoints Críticos lo requiere) y antes de iniciar Bloque 5. Depende de B4.   
  Enmienda S19: con Bloque 5 disuelto, CLAUDE.md es el último artefacto de especificación —
  precondición dura de la ejecución (el Task Protocol que gobierna el loop incremental debe
  existir antes de construir). Secuencia final: terminar roadmap.md (Fase 5 nivel Paso) → B6
  (CLAUDE.md) → construcción.

- **Bloque 5 — Development Fase 1** (archivo) ← **DISUELTO (S19)**. Bajo descomposición atómica
  JIT/incremental, el spec de development se escribe durante la ejecución de cada fase (MODE:
  PLANNING → EXECUTION del Task Protocol), no como bloque upstream. El spec de PR completa su
  fase de especificación en Bloque 6 (CLAUDE.md); después solo hay construcción.


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

- Consolidación multiobra y grano de agregación. R4 retirada en S9. Reemplazada por:
  Tabla 1: grano (obra_norm, id_actividad, id_tipologia), filas MATCHED únicamente (R4-T1).
  Tabla 2: grano (obra_norm, id_actividad, id_tipologia, fecha), filas MATCHED únicamente;
  distribución por fecha intencional y no colapsable (R4-T2).

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

- Disposición del no-match referencial PR↔mapeo (enmendada S9 y S10): fila excluida del output
  final de Tabla 1, Tabla 2 y Tabla 3; id_tipologia = null; tipologia_status = NO_MATCH (enum
  binario interno {MATCHED, NO_MATCH} — columna de enrutamiento interno, no aparece en
  ningún output final); registrada en reporte de reconciliación para tratamiento por
  analista. id_tipologia no nulo en outputs finales de Tabla 1 y Tabla 2 por construcción.
  No-lossiness mantenida a nivel de pipeline; outputs finales limpios (MATCHED únicamente).
  Esquema fino y destino del reporte de reconciliación → Bloque 3. [A] (R8 enmendada)
  Contrato de consumo de E parcialmente resuelto para Tabla 1 y Tabla 2.

- Gobernanza del filtro de vigencia: ESTATUS C.CLOUD exclusivamente;
  ESTATUS no participa bajo ninguna circunstancia. Comparación del filtro
  exclusivamente contra _norm. [B] (5.2)

- Contrato de tipo de ESTATUS C.CLOUD en ingesta: cast forzado a Utf8 antes
  de 5.1, preservando null; valor numérico, si existe, siempre literal "0".
  [B] (R13)
- Conjuntos de ruteo del filtro de vigencia: Finalizado (R9), Programado
  (R10). [B]
  
- Disposición residual no-lossy para ESTATUS_C_CLOUD_norm no reconocido
  (enmendada S10): cola de reconciliación/excepción, exclusión de ambas tablas
  append y de la Tabla 3 (solo filas ruteadas a R9/R10 integran la Tabla 3),
  homologada a R8 en simetría. Identidad exacta del artefacto destino diferida a
  Bloque 3, igual que R8. [B] (R11 enmendada)

- Mapeo del campo estatus en la tabla 3: exclusivamente desde ESTATUS
  C.CLOUD crudo, sin transformación. Reafirma sin excepciones que la tabla
  3 consume columnas crudas en su totalidad; se evaluó y se rechazó
  explícitamente una enmienda global al contrato 5.1 que habría migrado la
  tabla 3 completa a consumir _norm. [B] (R12)

- Contrato de gating de USAR: dominio cerrado {SI, NO}, evaluado contra
  valor crudo (sin _norm); gate destructivo posicionado upstream de toda
  la tubería (antes de 5.1, R1, R7, R9–R11); excepción explícita y
  deliberada a la postura no-lossy del sistema, sin conteo, log ni
  artefacto de reconciliación — riesgo aceptado de pérdida silenciosa ante
  variación de casing/espacios. [B] (R14)

~~- Esquema de salida de la Tabla 3 actualizado: exclusión de la columna
  usar (constante y no informativa tras el gate de R14). Esquema vigente:
  [estatus, nombre_actividad, nivel_1..5, fecha_inicio, macropartida,
  obra:Utf8]. [B] (R15)~~
> Superado por la enmienda S10 de R15. Ver entrada vigente más abajo.

- Gate de ingesta estructural para archivos PR: layout universal sin
  excepciones para ninguna obra. Formato de archivo restringido a .xlsx
  (R16). Filas 1–5 y columnas A–S descartadas incondicionalmente como
  ruido no validado; fila 6 = encabezado, columna T = origen de datos
  (R17). Validación literal completa, sensible a mayúsculas y espacios,
  de los ~30 encabezados esperados desde la columna T; cualquier
  discrepancia aborta la ejecución completa multiobra, sin salida
  parcial, hasta resolución upstream (R18). [F].

- Semántica y contrato de validación del campo FECHA: invariante respecto a estatus o ruteo,
  no participa de 5.1, fecha real de inicio en ubicación física específica (R19). Passthrough
  exclusivo y sin transformación en Tabla 3, al grano de fila original (R20). Exclusión
  deliberada de esquema en Tabla 1 — Finalizado no necesita la fecha para su consumidor;
  verificado sin conflicto contra el mandato de "ritmos históricos" del módulo PR, que se
  calcula fuera de esta tabla (R21). Validación activa no-lossy ante FECHA nulo, en blanco o
  mal formado, homologada a R2 — enruta a cola de anomalías sin abortar la corrida (R22).
  Nomenclatura de salida fijada: `fecha` (no `fecha_inicio`). [C]

- Definición de "unidad" y precondición estructural de conteo: unidad = unidad física de
  ejecución hiper-localizada (departamento, ciclo, unidad de ejecución real). Invariante
  declarado: (obra_norm, id_actividad, N1_norm..N5_norm) aparece exactamente una vez en
  un archivo PR tras el gate R14; una fila superviviente = una unidad física. El pipeline
  depende de este invariante sin verificarlo activamente. count_unidades = conteo de filas
  MATCHED en el grupo de agregación, válido como proxy de unidades físicas bajo el
  invariante declarado. Nombre final del campo: count_unidades. [C] (Sección 5.5, R4-T1, R4-T2)

- Posición de PR en el grafo de PLAN: T1 y T2 → CÓMPUTO; T3 → consumidor humano directo (Google Sheets / Excel). 
   PR no alimenta a PLANTIR. [EN DISPUTA → cerrado S10]

- CÓMPUTO declarado como módulo nombrado en el grafo de PLAN. Función: recibe unidades ejecutadas/programadas 
   de PR (T1, T2) y rendimiento por tipología de MOLD; calcula material ejecutado y material programado; 
   expone el resultado al usuario humano final. Inputs: PR (T1, T2) + MOLD (yield). Output: material ejecutado, 
   material programado. Implementación actual: Excel + Power Query + Sheets, 
   sujeto a refactorización. Reemplaza el "módulo sin nombre" del grafo original. [EN DISPUTA → cerrado S10]

- Compatibilidad del esquema agregado T1/T2 con CÓMPUTO confirmada: CÓMPUTO 
   no necesita columnas a nivel de fila (nivel_1..5, nombre_actividad, macropartida). 
   El contrato de 13 columnas pertenecía al pipeline anterior y no aplica al nuevo diseño. [EN DISPUTA → cerrado S10]

- Esquema Tabla 3 enmendado (R15 enmendada, S10): reducido a 
   [obra:Utf8, estatus:Utf8, nombre_actividad:Utf8, tipologia:Utf8, fecha:Date]. 
   Eliminados: nivel_1..5, macropartida. Agregado: tipologia (valor crudo de la columna Tipologia del mapeo, pre-normalización). 
   Solo filas MATCHED: NO_MATCH excluidas de la Tabla 3 (R8 enmendada S10); tipologia no nulo por construcción. 
   Fuentes declaradas por campo en §4 y R15. [B]

- Mecanismo de resolución de "versión vigente" para archivos PR (cierra
  #11): para cada fila (ruta_archivo, nombre_programa) declarada en la
  hoja de programas vigentes, el sistema recorre recursivamente el árbol
  bajo ruta_archivo, identifica carpetas con patrón
  <PREFIJO>_SEM_<AAAAMMDD>, extrae la fecha codificada (nunca mtime),
  busca coincidencia exacta de nombre_programa (sensible a
  mayúsculas/espacios, homologado a R18) dentro de ellas, y selecciona
  el archivo de fecha máxima. Carpetas no conformes al patrón se ignoran
  sin abortar (R24). Cero coincidencias o empate en la fecha máxima
  abortan la corrida completa multiobra, homologado a R16/R18 (R25,
  R26); mtime explícitamente rechazado como criterio de desempate por
  riesgo de inconsistencia bajo sincronización de Drive. Cardinalidad de
  la hoja de programas vigentes: una fila por (obra, macro_partida),
  macro_partida embebida en nombre_programa. [D] (5.6, R23–R26)

- Frontera de salida PR (avance parcial #10, S11): las 3 tablas aterrizan
  en BigQuery; PQ y la futura capa Sheets/Workspace se conectan ahí; PQ es
  transitorio (Modelo 2), se decomisiona en la migración final; PQ extrae
  tablas procesadas (vistas), no crudos. Split de motor (Opción C): Polars
  posee reglas a nivel de fila hasta R8 (normaliza, gatea USAR, hashea,
  joinea, dispone no-match) y stagea stg_matched + recon_nomatch; BigQuery
  posee agregación (R4-T1/T2), ruteo de vigencia (R9/R10) y las vistas de
  servicio T1/T2/T3. Reglas de muerte de fila no-lossy (R8/R11/R14/R22)
  permanecen en Polars. [D] (5.7, D-10.1/D-10.2/D-10.3)

  - Entorno de ejecución del pipeline Polars de PR (cierra #14): Cloud Run
  Job (run-to-completion, sin servidor HTTP), no Cloud Run Service; GCE y
  Dataflow descartados por sobredimensionamiento frente a la carga real
  (D-14.1). Disparo programado vía Cloud Scheduler invocando la Cloud Run
  Jobs Execute API; día/hora es parámetro de despliegue, no regla de
  negocio (D-14.2). Dimensionamiento validado contra el peor caso
  declarado (≤24 archivos PR activos, ≤12.000 filas cada uno, ≤288.000
  filas/semana); cotas de recursos por defecto de Cloud Run Jobs
  suficientes (D-14.3). Autenticación a Drive: archivos PR y hoja de
  programas vigentes residen en la misma Shared Drive; service account
  del Job agregada como miembro directo, sin domain-wide delegation ni
  sincronización forzada a GCS (D-14.4) — resuelve, dentro de #10, la
  bifurcación de fuente de entrada a favor de Drive. Mecánica de
  escritura a BigQuery: reemplazo completo (no acumulativo) de
  stg_matched y recon_nomatch en cada corrida semanal, diferido y atómico
  respecto de los gates de abort existentes — la escritura solo se
  ejecuta si la corrida completa (todas las obras, todos los gates R16–
  R18/R23–R26) no disparó ningún abort; si dispara, la escritura nunca se
  alcanza y el snapshot de la semana anterior queda intacto (D-14.5). No
  modifica R16/R18/R25/R26. Consecuencia aceptada explícitamente: una
  falla estructural en una sola obra congela el refresco de las 6–7 obras
  esa semana, no solo el de la obra defectuosa. [D] (5.8, D-14.1–D-14.5)

- Cierre parcial de #10, sub-item (d): el alcance original ("interfaz
  concreta de CÓMPUTO contra BigQuery") se corrige a un contrato de
  frontera de consumo propio de PR — esquema estable (§4) + señal de
  frescura (R27, §5.9) + atomicidad (D-14.5, confirmada suficiente sin
  regla adicional). El mecanismo de lectura interno de CÓMPUTO (sustrato,
  PQ hoy, GCP-nativo en el objetivo) queda explícitamente fuera de
  alcance de PR, diferido a la especificación futura de CÓMPUTO. [D] (S13)

- Señal de frescura de snapshot: run_date:Date, estampada en stg_matched
  y recon_nomatch al momento de la escritura del Job (un único valor por
  corrida — la fecha de ejecución del Cloud Run Job, no la fecha de
  vigencia por obra de R23–R26); heredada por passthrough en las vistas
  T1, T2 y T3; incluida como miembro del GROUP BY en R4-T1/R4-T2 sin
  fragmentar el grano (constante por snapshot); distinta de fecha (dato
  de negocio, R19–R22). Granularidad Date, no Timestamp, suficiente dado
  que D-14.5 garantiza escritura solo tras corrida completa y exitosa.
  [D] (§5.9, R27, S13)

- Atomicidad de consumo confirmada suficiente: D-14.5 (WRITE_TRUNCATE
  único, diferido hasta corrida exitosa sin abort) cubre la garantía de
  lectura sin mezcla parcial entre snapshots para T1/T2/T3; no requiere
  regla ni artefacto adicional. [D] (§5.7, S13)

- Cierre de #10, sub-item (c) — mecanismo de conexión: modo pull
  exclusivo, Connected Sheets para T3, Power Query para T1/T2, sin
  componente intermedio que materialice o escriba datos, refresh manual
  por el analista, autenticación bajo identidad individual de Google. No
  modifica D-14.5. [D] (§5.10, R28, S14)

- Cierre de #10, sub-item (c) — partición de datasets: stg_matched y
  recon_nomatch en dataset de staging oculto; T1, T2 y T3 como vistas
  autorizadas en dataset de servicio separado. Rol IAM de analistas
  otorgado exclusivamente sobre el dataset de servicio, nunca sobre el de
  staging. Enmienda: retirada la restricción de exclusión de T1/T2 para
  analistas — leen T1, T2 (PQ) y T3 (Connected Sheets) bajo identidad
  individual. Visibilidad cross-obra en T1/T2 aceptada, no bloqueante;
  filtrado por obra diferido al dashboard/CÓMPUTO. [D] (§5.10, R29, S14;
  D-10.4 en §5.7)

- Destino fino de recon_nomatch (amendment a R8, S14): dataset de
  staging oculto, sin acceso de analistas; distribución exclusiva del
  dueño del sistema, gestionada manualmente. [A/B] (R8 enmendada S14)

- Cierre del motor y destino del artefacto residual de R11: vista de
  BigQuery, simétrica a T1/T2, complementaria a R9∪R10, declarada dentro
  del dataset de staging oculto (no en el de servicio) para no heredar el
  grant IAM de analistas del dataset de servicio. Distribución exclusiva
  del dueño del sistema, en simetría con recon_nomatch. No modifica
  D-10.3: el motor de R9/R10 permanece en BigQuery. Con esta regla, #10
  cierra en su totalidad. [D] (§5.10, R30, S14)

- Vigencia y presencia del archivo de mapeo de tipologías (cierra #13):
  confirmado estructuralmente, no por analogía con R23–R26 — no existe
  versionado por fecha; es un archivo único y estático por obra, sobrescrito
  in situ. Carpeta global declarada como valor de configuración único,
  fuera del grano de programas_vigentes (Tipologias Obras). Convención de
  nombre `{obra}_mapeo_tipologias`, match por nombre de archivo normalizado
  (pipeline 5.1) contra obra_norm — divergencia deliberada de la postura
  literal de R18. Gate de accesibilidad de la carpeta global, homologado a
  R16: inaccesibilidad aborta la corrida completa, log explícito de la
  causa (R31). Gate de presencia por obra: cero coincidencias y
  coincidencia ambigua reciben el mismo tratamiento — exclusión de todas
  las macro_partidas de la obra afectada en la corrida actual, las demás
  obras continúan sin abort global, solo registro en el log de ejecución
  del Job, sin artefacto persistido (R32) — divergencia deliberada de la
  postura global de R25/R26. [G] (§5.11, R31–R32, S15)

  - Estructura de Bloque 4 (Roadmap por fases): dos niveles obligatorios y simultáneos — roadmap.md
  (Fase/Paso, descriptivo: Nombre, Objetivo, Entregables, sin Done Criteria a este nivel) y
  development.md (atómico, Fase.Paso.N, Status del enum cerrado, Done Criteria, Done Evidence;
  foreign key = truncar el último segmento del ID). Mismo patrón que MIDAS. Capa de Paso y capa
  atómica no se fusionan: error de capa detectado y corregido en S16 (Done Criteria no se adjunta
  directamente a un Paso).

- Partición de Fases del roadmap gobernada por el corte de motor D-10.3 (Bloque 3): Fase 0 (scaffolding
  local, sin GCP), Fase 1 (pipeline Polars local, reglas de fila hasta R8, testeable con fixtures sin
  Drive/BigQuery), Fase 2 (provisión GCP — Cloud Run Job, Scheduler, auth Drive, datasets BigQuery),
  Fase 3 (resolución de archivo en vivo contra Drive — R23–R26/R31–R32, depende de la service account
  de Fase 2), Fase 4 (agregación y vistas de servicio BigQuery — lado productor, incluye partición de
  dataset de R29 y estampado de run_date de R27), Fase 5 (habilitación de consumidores y cutover — IAM
  grant de R29, conector R28, retiro de PQ/M legado).

- Rol del agente ejecutor de IA resuelto (cierra la consideración abierta heredada de S15): Claude
  Code es par de desarrollo en tiempo de construcción (escribe código, Alberto ejecuta/commitea, asiste
  en debugging); cero rol en el runtime de producción. El Cloud Run Job + Cloud Scheduler es totalmente
  autónomo en producción (ya cerrado en Bloque 3, D-14.1/D-14.2) — esta resolución no reabre esas
  decisiones, solo fija el momento/rol de introducción del agente dentro del roadmap (no es una
  restricción a nivel de Paso 0 ni de ningún paso posterior; es transversal al ciclo de desarrollo).

- Fase 0 (Local Repo & Scaffolding) cerrada en su totalidad: roadmap (Nombre/Objetivo/Entregables) +
  11 tareas atómicas (0.1.1–0.1.11), todas [BACKLOG], ninguna ejecutada. Decisiones de convención
  cerradas dentro de esta fase: nombre de paquete `pr` (sin rename, coincide con el repo); gestor de
  dependencias requirements.txt con pines exactos (==), elegido sobre Poetry/uv para mitigar drift en
  un job de producción desatendido semanal; layout src/ (src/pr/) con pyproject.toml mínimo
  ([build-system]/[project], sin Poetry) requerido para instalación editable — gap real detectado y
  corregido en sesión; .python-version fija la versión del interprete, misma justificación que el
  pineo de librerías; IDE VS Code, .vscode/extensions.json de Fase 0 limitado a ms-python.python y
  ms-python.vscode-pylance — extensión Google Cloud Code deliberadamente diferida a Fase 2, sin
  tooling de GCP antes de su fase correspondiente.

- Fase 1 (Polars Core Pipeline, local) descompuesta a nivel atómico en su totalidad: 8 Pasos (1.1
  Test Fixtures, 1.2 Structural Ingestion Gates & Type Contracts incluyendo R13, 1.3 USAR Gate, 1.4
  Normalization & Obra Derivation incluyendo R3/R3.1, 1.5 Identity Hashing, 1.6 Tipología Join &
  Disposition, 1.7 FECHA Handling, 1.8 stg_matched Assembly & Row-Level Output Contract). Auditoría de
  cobertura detectó y corrigió tres huecos del borrador original: R13 y R3/R3.1 estaban en el scope
  declarado de Fase 1 sin Paso asignado (resuelto, foldeados en 1.2 y 1.4 respectivamente); no existía
  un Paso terminal que fijara el esquema del frame de salida row-level (resuelto, nuevo Paso 1.8).
  R23–R26/R31–R32 (resolución de archivo vigente) explícitamente excluidas de Fase 1 — Fase 1 opera
  solo contra fixtures entregadas, no contra Drive en vivo (eso es Fase 3, después de que Fase 2
  provisione la service account). PENDIENTE no bloqueante de Fase 1: esquema fino de la cola de
  anomalías (R2/R6.1/R22) — el spec ya lo difiere; Fase 1 solo fija el ruteo hacia ella, no su forma
  final. Distinto de recon_nomatch (R8), que sí tiene esquema mínimo cerrado.

  - Fase 2 (GCP Infrastructure Provisioning) descompuesta a nivel atómico en su totalidad: 5 Pasos (2.1
  Aprovisionamiento base y Despliegue del Cloud Run Job — incluye Artifact Registry, recursos sin
  override por D-14.3, observabilidad/alertas; 2.2 Configuración del disparo programado — Cloud
  Scheduler, identidad de invocación con permiso run.jobs.run scoped al Job; 2.3 Autenticación de la
  Service Account contra Shared Drive; 2.4 Provisión de datasets BigQuery, container-level únicamente
  — pr_staging y pr_serving, IAM base restringido a la SA del Job sobre staging; 2.5 Verificación de
  aprovisionamiento y contrato de salida), 20 tareas atómicas. Frontera explícita con Fase 4 sobre
  D-10.4: Fase 2 crea los datasets físicos y el IAM de contenedor; Fase 4 crea tablas, vistas y el link
  de authorized views — ninguna de las dos Fases asume la responsabilidad de la otra. Convención de
  nombres de dataset fijada: pr_staging / pr_serving. Auditoría de cobertura detectó y corrigió tres
  huecos antes de cerrar la descomposición: D-14.3 sin Paso propio (foldeado en 2.1.4 como Done
  Criteria documental), ausencia de Paso terminal de verificación (resuelto, Paso 2.5), y ambigüedad
  de propiedad de dataset entre Fase 2 y Fase 4 (resuelta por la frontera container/contenido). Gap no
  resuelto en esta sesión, diferido explícitamente a la apertura de Fase 3: ningún Paso posee la tarea
  de construir la imagen de producción del pipeline (código de Fase 1) y reemplazar el placeholder de
  2.1.3 — bloquea cualquier prueba real de R23–R26/R31–R32 contra el Job hasta resolverse.

- Fase 3 (Live Drive File Resolution) descompuesta a nivel atómico en su totalidad: 4 Pasos (3.1
  Production Image Build & Job Redeployment — resuelve el gap de imagen diferido de Fase 2, reemplaza
  el placeholder de 2.1.3 por el código de Fase 1; 3.2 Precondition Gate, R31 ∥ R23–R26; 3.3 Per-Obra
  Mapeo Presence Resolution, R32; 3.4 Live Resolution Verification & Exit Contract), 17 tareas
  atómicas. Decisión arquitectónica de gate: R31 (accesibilidad de carpeta global de mapeos) y R23–R26
  (resolución de PR vigente) son pistas estructuralmente independientes — tocan árboles de carpetas sin
  relación — y se diseñan como un único Precondition Gate concurrente con semántica OR-abort (cualquier
  pista que falle aborta la corrida completa antes de todo procesamiento de datos) y AND-success (ambas
  deben resolver para levantar la barrera). Aclaración de scope sobre R31, registrada para trazabilidad
  sin enmendar §5.11: "antes de cualquier otro procesamiento del batch" se acota a la cadena mapeo→join
  (R32→R7), no es un lock secuencial literal sobre R23–R26. R32 excluido del gate por diseño: falla
  por-obra (exclusión acotada), no por-batch — vive en su propio Paso (3.3) aguas abajo. Manejo de
  error transitorio de Drive API incorporado al gate (3.2.1/3.2.2/3.2.3): retry-before-abort hasta 3
  intentos con delay entre intentos, params como config de despliegue (homólogo a D-14.2), diferenciado
  explícitamente del fallo definitivo (carpeta ausente, cero coincidencias, empate → abort en primera
  respuesta sin retry). El retry precede a la señal de abort y NO amienda las condiciones de abort de
  R25/R31; es capa de robustez de ejecución de Fase 3, no cambio de regla de Bloque 3. Paso terminal
  3.4 (verificación en vivo) homólogo al 2.5 de Fase 2: prueba un ciclo completo contra la Shared Drive
  real (no fixtures), confirma paridad fixture/vivo, escritura real WRITE_TRUNCATE con run_date (R27),
  e inyección de cada ruta de abort/exclusión contra artefactos desechables. Contrato de salida: al
  cierre de Fase 3, stg_matched/recon_nomatch portan datos vivos validados; Fase 4 sin dependencia
  residual.

- Metodología de Done Criteria a nivel de roadmap fijada: tareas atómicas usan Done Criteria de tipo
  checklist + Done Evidence — no el test DADO/CUANDO/ENTONCES, reservado exclusivamente para reglas
  de negocio de Bloques 1–3. Modo Socrático reservado para contradicciones estructurales genuinas
  (lo que rompería o entraría en conflicto con una decisión ya cerrada); convenciones de bajo riesgo
  y reversibles (nombrado, elección de tooling) se proponen directas en modo Estructurador y se
  confirman en una sola pasada, sin interrogatorio uno por uno.

- Enmienda a §7: orden de bloques corregido de B4→B5→B6 a B4→B6→B5. Bloque 6 (CLAUDE.md) reclasificado
  de "condicional" a activo/confirmado — la condicionalidad declarada en S15 ("solo si arquitectura
  sin Colab y contrato de PR exigen cambios de gobernanza") se resuelve afirmativamente en S16.

- Contrato de salida row-level de Fase 1 (Paso 1.8), de dos caras: (1) frame Arrow/Polars LOCAL
  de 9 columnas — obra, obra_norm, estatus_c_cloud, estatus_c_cloud_norm, nombre_actividad,
  id_actividad (UInt64), tipologia, id_tipologia (UInt64), fecha (Date); sin run_date; _norm
  sobrevivientes = {obra_norm, estatus_c_cloud_norm} únicamente; N1..N5, actividad_norm,
  tipologia_norm excluidos (consumidos upstream, sin lector downstream). (2) tabla BigQuery de 10
  columnas en la frontera de escritura (3.4.3): mismas 9 + run_date:DATE; ids casteados a STRING;
  todas las columnas de texto a STRING. La derivación es determinada por reglas cerradas (R5,
  R8/D-10.3, R9/R10, R15/R20, R4-T1/T2, R1/R6, R27), no elegida. [F/A] (1.8.1, 3.4.3, S19)

- Tipo de id_actividad/id_tipologia en BigQuery: STRING, no INT64. BigQuery carece de UINT64;
  ~50% de los hashes FNV-1a-64 exceden INT64 max y desbordarían a negativo, corrompiendo ~50% de
  las llaves. STRING preserva el rango completo con join por igualdad de texto, bijectivo y
  estable; CÓMPUTO trata los ids como llaves opacas, nunca computa sobre ellos. Cast aplicado en
  la frontera de escritura (3.4.3), no en el pipeline local (D-10.3 conserva tipos Arrow nativos).
  [A] (3.4.3, S19)

- Atribución de run_date corregida: estampado en la frontera de escritura del Job (Fase 3,
  3.4.3, R27/D-14.5), no en Fase 1 ni Fase 4. El frame local de Fase 1 no porta run_date — un
  fixture local no tiene fecha de ejecución de Job que estampar. Fase 4 solo lo hereda por
  passthrough en las vistas + lo incluye como miembro del GROUP BY (R4-T1/T2); no lo re-estampa
  ni transforma. [D] (1.8.1, 3.4.3, 4.x, S19)

- Fase 4 (BigQuery Aggregation & Service Views — lado productor) descompuesta a NIVEL DE ROADMAP
  (Paso) únicamente: 5 Pasos, sin tareas atómicas (diferidas a ejecución, JIT). T1/T2 vistas de
  agregación en pr_serving (R4-T1/R9, R4-T2/R10, run_date en GROUP BY, ids STRING en salida); T3
  vista passthrough row-level en pr_serving (crudo R15/R12/R20, filtro R9∪R10); R30 vista
  residual row-level en pr_staging (complemento de R9∪R10, IS NOT NULL guard obligatorio porque
  NULL es miembro de R10, columnas crudas identificadoras, aislada del serving). Authorized-view
  link como Paso propio (4.4): T1/T2/T3 autorizadas contra pr_staging, R30 excluida del grant;
  ventana transitoria de vistas-existen-pero-no-consultables acotada y aceptada (sin consumidores
  conectados aún). Verificación terminal 4.5: exhaustividad y disjunción de la partición de
  ruteo (T1∪T2 + R30 = total MATCHED), passthrough de run_date, contrato de salida hacia Fase 5.
  Frontera D-10.4 respetada: Fase 4 posee contenido (tablas/vistas/link), Fase 2 poseía
  containers. [D] (roadmap.md Fase 4, S19)

- Grano de la vista residual R30: row-level, no agregado. "Simétrica a T1/T2" (§5.10) interpretado
  como simetría de mecanismo (vista BigQuery con predicado de membresía, motor R9/R10 en BQ), no
  de grano. Gemelo funcional recon_nomatch es row-level; agregar R30 a counts destruiría la
  trazabilidad que el dueño del sistema necesita para ubicar y corregir estatus no reconocidos
  upstream. Mismas columnas crudas que T3. [D] (4.3.1, S19)

- Metodología de descomposición: a partir de Fase 4 inclusive, JIT/incremental — el ejecutor
  propone las tareas atómicas de una fase (MODE: PLANNING), las ejecuta (MODE: EXECUTION),
  avanza; la auditoría adversarial de cobertura se mantiene, reubicada al momento de proponer
  cada fase, no upstream en lote. Solo Fases 0–3 conservan atómicos escritos, congelados como
  referencia revisable (el state machine ya permite rechazar/reworkar una tarea obsoleta en
  ejecución). Bloque 5 disuelto. [proceso] (S19)


### PARCIALMENTE DEFINIDO
- Canonicalización de `""` vs `null` en ESTATUS C.CLOUD. Ambos valores ya
  están aceptados independientemente en el conjunto de ruteo de Programado
  (R10); el comportamiento de ruteo no depende de esta distinción y por
  tanto no bloqueó el cierre de #4. Abierto: si deben colapsarse a una sola
  representación canónica en ingesta, o conservarse como estados crudos
  distintos — relevante específicamente para el campo estatus de la tabla
  3 (R12), que al consumir el valor crudo sí distingue entre ambos.

~~- Frontera GCP+Polars ↔ PQ/Sheets (#10): salida resuelta (BigQuery + Opción C); fuente de entrada resuelta (Drive, D-14.4); sub-item (d) cerrado (R27 + atomicidad). Abierto: (c) mecanismo de conexión Sheets; motor del artefacto residual de R11. [D] (Bloque 3)~~
> Cerrado en S14. Ver DEFINIDO — R28–R30, §5.10, D-10.4 en §5.7.

- Fixture 1.1.4 (stg_matched esperado): reconciliada al frame local de 9 columnas (S19). Cerrada
  como ítem de arrastre; si su contenido aún codifica N1..N5 o run_date, 1.8.2 falla contra
  fixture obsoleta — verificar en ejecución de Fase 1.

### PENDIENTE (crítico — bloquea generación de archivos)
~~6. [CERRADO EN S9 — ver Sección 5.5, R4-T1, R4-T2, R8 enmendada]~~

~~- **#10** — Frontera GCP+Polars ↔ PQ/Sheets. Avance parcial S11/S12/S13. Resta únicamente: (c) mecanismo de conexión Sheets, motor del artefacto residual de R11. [D] (Bloque 3)~~
> Cerrado en S14. Ver §5.10, R28–R30; D-10.4 en §5.7.
~~- **#13** — Estructura de carpeta y resolución de "vigente" para la Tabla Mapeo de Tipologías; análogo a #11, sin estructura declarada. [G] (Bloque 3)~~
> Cerrado en S15. Ver §5.11, R31–R32.

~~13. Estructura de carpeta y mecanismo de resolución de "vigente" para la Tabla Mapeo de Tipologías — análogo a #11, sin estructura de carpeta declarada aún. [G] (Bloque 3)~~
> Cerrado en S15. Ver Sección 5.11, R31–R32.

**PENDIENTE crítico: vacío desde S15.**
~~14. Entorno de ejecución del pipeline Polars de PR [...]. [D] (Bloque 3)~~
> Cerrado en S12. Ver Sección 5.8, D-14.1–D-14.5.14. Entorno de ejecución del pipeline Polars de PR (Cloud Run / Function / GCE / Dataflow / otro) — no declarado. Bloquea la decisión de fuente de entrada (Drive vs GCS) y la mecánica de escritura a BigQuery. Surgido en S11 al descartar el supuesto erróneo de Colab (Colab pertenece a MIDAS, no a PR). [D] (Bloque 3)

### EN DISPUTA
~~- Posición de PR en el grafo de dependencias del sistema PLAN.~~
~~- Identidad del consumidor "MRP Excel": ¿PLANTIR, módulo sin nombre o legado externo?~~
~~- Si la salida agregada (count) rompe al consumidor que hoy espera 13 columnas a nivel de fila.~~
> Los tres ítems cerrados en S10. Ver DEFINIDO.

---
 
## 9. BITÁCORA DE SESIONES
 
**S1 — 2026-06-15.** Recepción y auditoría de la historia de usuario. Construcción del plan por bloque. Generación de este documento de estado. Sin reglas de negocio definidas aún. Próximo paso: Bloque 2 (Domain), Modo Socrático, primera pregunta = regla de construcción de `id_actividad`.

**S2 — 2026-06-16**: cierre de #1 y #9. Decisiones: id_actividad = FNV-1a-64 sobre texto normalizado; consolidado multiobra Camino B; pipeline de normalización de 10 pasos. "#3 cerrado — obra = string antes del primer ''; obra_norm por R5; guard R3.1 para nombre sin ''.". Proximo foco #3.

**S3 — 2026-06-17**: cierre de #2. Llave de join PR↔mapeo = (obra_norm, N1_norm..N5_norm) sobre _norm; id_tipologia = FNV-1a-64(obra_norm + "|" + tipologia_norm), UInt64, orden fijo; guard fail-fast R6.1 para tipología vacía en mapeo; mapeo = archivo por obra, obra_norm desde columna Obra cruda. Surgió grieta nueva #12. Próximo foco candidato: #12, luego barrer #4/#5.

**S4 — 2026-06-17**: cierre provisional de #12. Disposición no-lossy del no-match referencial: fila se conserva, id_tipologia = null + tipologia_status = NO_MATCH (enum binario por fila) + reporte de reconciliación (R8). Consecuencias: id_tipologia pasa a UInt64 nullable en §4; #6 se agudiza (el grano debe preservar matched/NO_MATCH). Provisional, dependiente de E (contrato de consumo, EN DISPUTA). Próximo foco candidato: barrer #4/#5.

**S5 — 2026-06-17**: cierre de #4. Decisiones: ESTATUS C.CLOUD gobierna el filtro de vigencia en exclusiva, ESTATUS sin autoridad alguna; comparación del filtro exclusivamente contra _norm; conjuntos de ruteo declarados para Finalizado (R9) y Programado (R10); disposición residual no-lossy para estatus no reconocidos, homologada a R8 (R11); contrato de tipo en ingesta para ESTATUS C.CLOUD — cast forzado a Utf8 antes de 5.1, null preservado, valor numérico siempre literal "0" (R13); campo estatus de la tabla 3 mapea exclusivamente desde ESTATUS C.CLOUD crudo (R12). Nota de proceso: se evaluó y se rechazó explícitamente una enmienda global al contrato 5.1 que habría migrado la tabla 3 completa a consumir _norm — la tabla 3 conserva su contrato original de columnas crudas, sin excepciones, porque es la única tabla cuya función es ser leída por un humano. Abierto, no bloqueante: canonicalización `""` vs `null` en ESTATUS C.CLOUD; identidad exacta del artefacto de reconciliación de R11, diferida a Bloque 3 junto con R8. Próximo foco candidato: #5 (semántica de USAR) o #6/#7/#8, a decisión de Alberto.

**S6 — 2026-06-17**: cierre de #5. Decisiones: USAR es dominio cerrado y binario {SI, NO}, sin variantes observadas; gate destructivo posicionado upstream de toda la tubería (antes de 5.1, R1, R7, R9–R11) (R14); comparación contra el valor crudo, sin generar USAR_norm — excepción deliberada a la postura no-lossy del sistema (sin conteo, log ni cola de reconciliación), riesgo aceptado de pérdida silenciosa por casing/espacios; esquema de Tabla 3 actualizado para excluir la columna usar, al volverse constante tras el filtro (R15). Conexión señalada y no abierta: el gate de R14 precede a la agregación de R4, por lo que count_unidades en Tablas 1 y 2 solo contará filas con USAR=="SI" — relevante para #6 (grano), aún pendiente. Próximo foco candidato: #6 (semántica de "unidad" y grano del count), #7 (semántica de FECHA por estatus) o #8 (universalidad de la precondición estructural de parseo), a decisión de Alberto.

**S7 — 2026-06-18**: cierre de #8. Decisiones: layout estructural universal sin excepciones para ninguna obra — filas 1–5 y columnas A–S (1–19) son ruido no validado, descartado incondicionalmente; fila 6 = encabezado absoluto, columna T = origen de datos (offset 0, alineado a USAR) (R17); formato de archivo restringido exclusivamente a .xlsx, violación aborta la ejecución completa (R16); validación literal completa de los ~30 encabezados esperados desde columna T, sensible a mayúsculas/espacios, sin normalización previa — cualquier discrepancia (encabezado extra, faltante, reordenado o con variación cosmética) aborta la ejecución completa multiobra, sin salida parcial, hasta corrección upstream (R18). Nota de proceso: R16–R18 introducen el modo de falla más destructivo del documento — más severo que R14 —, en tensión filosófica explícita con la postura no-lossy de R8/R11; se registra como split deliberado entre anomalías a nivel de fila (no-lossy) y anomalías estructurales a nivel de archivo (destructivo, abort global). Próximo foco candidato: #6 (semántica de "unidad" y grano del count) o #7 (semántica de FECHA por estatus), a decisión de Alberto.

**S8 — 2026-06-18**: cierre de #7. Decisiones: semántica de FECHA fijada como invariante respecto a estatus y ruteo de salida — fecha real de inicio de actividad en ubicación física específica, fuera del pipeline 5.1 (R19); Tabla 3 la consume como passthrough crudo al grano de fila original (R20); Tabla 1 la excluye por decisión de esquema confirmada — verificado sin conflicto contra el mandato de "ritmos históricos" de PR, que se calcula fuera de esta tabla (R21); validación activa no-lossy ante FECHA nulo/en blanco/mal formado, homologada a R2, defensiva ante una violación de supuesto no observada hoy en inspección empírica (R22). Nomenclatura de salida fijada: `fecha`. Hipótesis original de auditoría (FECHA cambia de significado por estatus) evaluada y descartada. Grieta nueva registrada dentro de #6: la corrección del grano de R4 ahora también debe resolver el mecanismo de derivación de fecha en Tabla 2; Alberto propuso un grano candidato (obra_norm, id_actividad, id_tipologia [+ fecha en Tabla 2]), recibido sin auditar. Próximo foco candidato: #6 (semántica de "unidad" y corrección del grano de R4, incorporando la propuesta recibida) o #10/#11 (Bloque 3), a decisión de Alberto.

**S9 — 2026-06-18**: cierre de #6. Decisiones: "unidad" definida como unidad física de ejecución hiper-localizada (departamento, ciclo, unidad de ejecución real); invariante estructural declarado: una fila superviviente al gate R14 = una unidad física, con la precondición de que (obra_norm, id_actividad, N1_norm..N5_norm) es único por archivo PR — el pipeline lo asume sin verificarlo activamente. count_unidades = conteo de filas MATCHED en el grupo, válido bajo el invariante; nombre final confirmado: count_unidades. R4 retirada y reemplazada por R4-T1 (grano Tabla 1: obra_norm, id_actividad, id_tipologia, filas MATCHED únicamente) y R4-T2 (grano Tabla 2: obra_norm, id_actividad, id_tipologia, fecha, filas MATCHED únicamente; distribución por fecha intencional y no colapsable). R8 enmendada: las filas NO_MATCH se excluyen del output final de Tabla 1 y Tabla 2 y van únicamente al reporte de reconciliación; id_tipologia no nulo en ambos outputs finales por construcción; tipologia_status es columna interna de enrutamiento, no aparece en outputs finales. Contrato de consumo de E parcialmente resuelto para Tabla 1 y Tabla 2. Próximo foco candidato: #10/#11 (Bloque 3 — frontera GCP+Polars ↔ PQ/Sheets y mecanismo de resolución de versión vigente), a decisión de Alberto.

**S10 — 2026-06-18**: cierre de los 3 ítems EN DISPUTA. Decisiones: PR alimenta CÓMPUTO (T1, T2) y al consumidor humano directo (T3); no alimenta PLANTIR. El "MRP Excel" consumidor es CÓMPUTO, módulo declarado formalmente en el grafo de PLAN con inputs PR+MOLD y output de material ejecutado/programado; reemplaza al "módulo sin nombre" del grafo original. Compatibilidad del esquema agregado de T1/T2 con CÓMPUTO confirmada — CÓMPUTO no necesita detalle a nivel de fila. Enmienda de R15: esquema Tabla 3 reducido a [obra, estatus, nombre_actividad, tipologia, fecha]; eliminados nivel_1..5 y macropartida; agregado tipologia (crudo del mapeo, pre-normalización). EN DISPUTA vacío. Próximo foco: #10 (frontera GCP+Polars ↔ PQ/Sheets) y #11 (resolución de versión vigente), ambos en Bloque 3.
Corrección dentro de S10: las filas NO_MATCH se excluyen también de la Tabla 3 (no solo de Tabla 1 y 2); R8 enmendada en consecuencia (S10). tipologia en Tabla 3 es no nulo por construcción. R11 enmendada en simetría: las filas residuales de estatus no reconocido también se excluyen de la Tabla 3 — la Tabla 3 contiene exclusivamente filas MATCHED ruteadas a R9 o R10. Retirada la entrada R15 obsoleta del ledger que aún declaraba el esquema antiguo de Tabla 3. Tachado el residuo resuelto en grieta C (§6) y corregido el marcador EN CURSO obsoleto en §7.

**S11 — 2026-06-19**: Bloque 3. Cierre de #11: mecanismo de resolución de versión vigente para archivos PR (R23–R26). Recorrido recursivo bajo ruta_archivo; carpetas con patrón <PREFIJO>_SEM_<AAAAMMDD>; fecha extraída del nombre de carpeta (nunca mtime, rechazado por inconsistencia bajo sync de Drive); match exacto de nombre_programa sensible a mayúsculas/espacios (homologado R18); selección por fecha máxima. Carpetas no conformes ignoradas sin abortar (R24). Cero coincidencias o empate en fecha máxima → abort multiobra completo (R25, R26), homologado a R16/R18. Cardinalidad de la hoja de programas vigentes corregida: una fila por (obra, macro_partida), macro_partida embebida en nombre_programa. Avance parcial de #10: frontera de salida resuelta — 3 tablas a BigQuery, PQ transitorio (Modelo 2) extrae vistas procesadas, split de motor Opción C (Polars dueño de reglas de fila hasta R8 + staging; BigQuery dueño de R4/R9/R10 + vistas de servicio); descartadas Opción A (dispersa no-lossy) y B (BigQuery pasivo, sin learning) (5.7, D-10.1/2/3). Restan en #10: (c) mecanismo Sheets, (d) interfaz CÓMPUTO, fuente de entrada Drive/GCS, motor del artefacto R11. Apertura de #13 (vigencia de Tabla Mapeo, análogo a #11, sin estructura declarada). Apertura de #14 (entorno de ejecución de Polars en PR — corregido supuesto erróneo de Colab, que pertenece a MIDAS). Próximo foco: #14 (entorno de ejecución, gatea fuente de entrada) o resto de #10, a decisión de Alberto.

**S13 — 2026-06-20**: Bloque 3. Foco: #10, cierre del sub-item (d). Corrección de alcance: el ítem original ("interfaz concreta de CÓMPUTO contra BigQuery") se reemplaza por un contrato de frontera de consumo propio de PR, no la especificación interna de CÓMPUTO — esquema estable (§4) + señal de frescura + atomicidad. Decisiones: (1) nueva regla R27 (§5.9, nueva subsección — Contrato de Trazabilidad de Snapshot): run_date:Date estampado en stg_matched y recon_nomatch al momento de la escritura del Job, un único valor por corrida (la fecha de ejecución del Cloud Run Job, no la fecha de vigencia por obra de R23–R26, que puede diferir entre obras), heredado por passthrough en las vistas T1/T2/T3, incluido como miembro del GROUP BY en R4-T1/R4-T2 sin fragmentar el grano (constante por snapshot), distinto de fecha (dato de negocio R19–R22); granularidad Date, no Timestamp, suficiente dado que D-14.5 garantiza escritura solo tras corrida completa y exitosa. (2) Atomicidad de consumo: confirmado que D-14.5 (WRITE_TRUNCATE único, diferido) es garantía suficiente para lectura sin mezcla parcial entre snapshots en T1/T2/T3; no se agrega regla nueva, solo nota de cierre en §5.7. Esquemas de Tabla 1, Tabla 2 y Tabla 3 (§4, R15) enmendados para incluir run_date. Auditoría de proceso: se identificó staleness en el párrafo "Estado" de §5.7 (aún mencionaba Drive/GCS y #14 como abiertos, ya resueltos en S12) — corregido en esta sesión como limpieza, no como decisión nueva. #10 queda reducido a: (c) mecanismo de conexión Sheets, motor del artefacto residual de R11. Próximo foco candidato: (c) o motor de R11, a decisión de Alberto.

**S14 — 2026-06-20**: Bloque 3. Foco: cierre total de #10 (ambos sub-items restantes). Nueva subsección §5.10 (Contrato de Consumo Externo — Sheets / Power Query / Aislamiento de Datasets). Decisiones: (1) R28 — mecanismo de conexión: modo pull exclusivo, sin componente intermedio que materialice o escriba datos (Connected Sheets para T3, Power Query para T1/T2), refresh manual por el analista, autenticación bajo identidad individual de Google; no modifica D-14.5. (2) R29 — partición de datasets: stg_matched y recon_nomatch en un dataset de staging oculto; T1, T2 y T3 como vistas autorizadas en un dataset de servicio separado; rol IAM de analista otorgado exclusivamente sobre el dataset de servicio — registrado también como D-10.4 en §5.7. Enmienda: se retira la restricción provisional "los analistas nunca acceden a T1/T2" (introducida y descartada dentro de la misma sesión, al detectarse contradicción con el flujo real de Power Query/Excel por-obra que alimenta a CÓMPUTO). Racional de Alberto: simplicidad operativa, datos no sensibles; consecuencia aceptada — visibilidad cross-obra en T1/T2, filtrado por obra diferido al dashboard/CÓMPUTO. (3) Amendment a R8: destino fino de recon_nomatch — dataset de staging oculto, distribución exclusiva del dueño del sistema. (4) R30 — motor y destino del artefacto residual de R11: vista de BigQuery simétrica a T1/T2, complementaria a R9∪R10, declarada dentro del dataset de staging (no en el de servicio) para no heredar el grant IAM de analistas; distribución exclusiva del dueño del sistema, en simetría con recon_nomatch; no modifica D-10.3, el motor de R9/R10 permanece en BigQuery. Auditoría de proceso: se identificó y corrigió una contradicción textual en D-10.3 (§5.7), que listaba a R11 como regla "consolidada en Polars" — corrección de redacción, no de decisión: el motor de evaluación de membresía R9/R10 fue y sigue siendo de BigQuery; solo el artefacto resultante (la vista residual) se ubica en el dataset de staging por razones de aislamiento de acceso, no porque Polars lo procese. #10 cierra en su totalidad. Único PENDIENTE crítico restante en Bloque 3: #13. Próximo foco: #13 (estructura de carpeta y resolución de "vigente" para la Tabla Mapeo de Tipologías) — cierra Bloque 3 si se resuelve.

**S15 — 2026-06-20**: Bloque 3. Foco: #13, último PENDIENTE crítico del bloque — estructura de carpeta y resolución de "vigente" para la Tabla Mapeo de Tipologías. Auditoría inicial: se forzó la confirmación estructural antes de aceptar la analogía con R23–R26 (#11); resultado: no hay versionado por fecha, es un archivo único y estático por obra, sobrescrito in situ — el problema real es de presencia/identificación, no de selección temporal. Nueva subsección §5.11 (Contrato de Presencia — Tabla Mapeo de Tipologías). Decisiones: (1) carpeta global ("Tipologias Obras")  declarada como valor de configuración único, fuera del grano de `programas_vigentes` (grain mismatch identificado y resuelto: el folder path es una constante de sistema, no varía por (obra, macro_partida)). (2) Convención de nombre `{obra}_mapeo_tipologias`; match contra `obra_norm` ejecutado sobre el nombre de archivo normalizado (pipeline 5.1) — divergencia deliberada de la postura literal de R18, justificada porque el nombre de archivo es tipeado libremente por el analista. (3) R31 — gate de accesibilidad de la carpeta global, homologado a R16: carpeta inalcanzable aborta la corrida completa, log explícito de la causa específica; distingue una falla de configuración de sistema de una fallade archivo individual. (4) R32 — g ate de presencia por obra: cerocoincidencias y coincidencia ambigua (más de un archivo normaliza al mismo nombre esperado) reciben idéntico tratamiento — exclusión de todas las macro_partidas de la obra afectada en la corrida actual, las demás obras continúan sin abort global, solo log de ejecución del Job, sin artefacto persistido. Divergencia deliberada respecto a R25/R26 (que homologan el mismo tipo de evento a abort global): aquí la falla del artefacto de mapeo se acota a su obra, no se propaga. Con el cierre de #13, Bloque 3m cierra en su totalidad — sin PENDIENTE crítico restante. Bloque 4 (Roadmap por fases) queda desbloqueado por la regla de gating de §7. Próximo foco: decisión de Alberto sobre si abrir Bloque 4 (Roadmap) en la próxima sesión, o realizar una revisión de cierre de Bloque 3 antes de avanzar.

**S16 — 2026-06-21**: Bloque 4. Apertura formal del bloque tras el desbloqueo de S15. Fijación de la estructura de dos niveles (roadmap.md descriptivo + development.md atómico), espejada del proyecto MIDAS; corrección de un error de capa detectado en sesión (Done Criteria no se adjunta al nivel Paso). Partición de Fases (0–5) fijada por el corte de motor D-10.3, no por conteo arbitrario — revisada dos veces en sesión (GCP promovido a Fase propia; R29 dividido entre Fase 4 y Fase 5). Resuelta la consideración abierta heredada de S15 sobre el momento de introducción del agente ejecutor: Claude Code es asistente de desarrollo sin rol en el runtime de producción (que permanece autónomo, Bloque 3 sin cambios); resolución transversal, no atada a un Paso específico. Fase 0 cerrada en su totalidad (roadmap + 11 tareas atómicas, decisiones de convención: paquete `pr`, requirements.txt pineado, src-layout con pyproject.toml mínimo, .python-version, VS Code con extensiones acotadas a Fase 0). Fase 1 descompuesta a nivel atómico en su totalidad (8 Pasos); auditoría de cobertura detectó y corrigió tres huecos (R13 y R3/R3.1 huérfanos, ausencia de Paso terminal de contrato de salida) antes de generar las tareas atómicas. Bloque 6 (CLAUDE.md) reclasificado de condicional a activo, secuenciado entre B4 y B5 (enmienda de §7: B4→B6→B5). Próximo foco: descomposición de Fase 2 (GCP Infrastructure Provisioning) siguiendo el mismo patrón de dos niveles; pendiente de bloque, no crítico: desglose de Fases 2–5 restantes.

**S17 — 2026-06-21**: Bloque 4. Foco: descomposición de Fase 2 (GCP Infrastructure Provisioning) a ambos niveles. Roadmap-level: 5 Pasos (2.1 Aprovisionamiento base y Despliegue del Job — Artifact Registry, D-14.3 sin override, observabilidad; 2.2 Disparo programado — Scheduler, permiso run.jobs.run; 2.3 Autenticación SA-Drive; 2.4 Datasets BigQuery container-level, pr_staging/pr_serving; 2.5 Verificación de aprovisionamiento y contrato de salida hacia Fase 3). Atomic-level: 20 tareas. Auditoría de cobertura, antes de cerrar: D-14.3 foldeado en 2.1.4 (no Paso propio); Paso terminal 2.5 agregado (ausente en el borrador); frontera Fase2/Fase4 sobre D-10.4 resuelta explícitamente — Fase 2 posee container (datasets físicos + IAM base), Fase 4 posee contenido (tablas, vistas, authorized views link). Contrapropuesta de Alberto incorporada en su totalidad: habilitación de Drive API y Artifact Registry API agregada a 2.1.1 (omisión detectada por Alberto, no por la auditoría inicial); provisión física de Artifact Registry agregada como 2.1.2, bloqueante para la creación del Job; observabilidad/alertas de fallo agregada como 2.1.5 — extensión deliberada de scope, justificada contra la consecuencia ya aceptada en S12 (D-14.5: fallo estructural congela refresco sin aviso); precisión de IAM corregida en 2.2.3 (run.jobs.run en vez de run.invoker genérico). Verificado y descartado: riesgo de permiso de pull de Artifact Registry para la SA del Job — cubierto implícitamente por el smoke test de 2.1.6, sin necesidad de tarea nueva. Gap nuevo identificado y diferido, no resuelto en esta sesión: ninguna Fase posee la tarea de construir la imagen de producción del pipeline (código de Fase 1) y reemplazar el placeholder de 2.1.3 — debe resolverse como primer punto al abrir Fase 3, antes de poder testear R23–R26/R31–R32 contra un Job real. Convención de nombres de dataset fijada: pr_staging / pr_serving. Fase 2 cerrada en su totalidad. Próximo foco: apertura de Fase 3 (Resolución de Archivo en Vivo), resolviendo primero el gap de imagen de producción antes de descomponer R23–R26/R31–R32.

**S18 — 2026-06-21**: Bloque 4. Foco: descomposición de Fase 3 (Live Drive File Resolution) a ambos niveles. Gap de imagen de producción de Fase 2 resuelto como Paso 3.1 (no diferido por segunda vez): construye la imagen del código de Fase 1, reemplaza el placeholder de 2.1.3, redespliega el Job, re-ejecuta smoke test contra la imagen real. Resolución de la ambigüedad de R31 (planteada en Modo Socrático): "antes de cualquier otro procesamiento del batch" se acota a la cadena mapeo→join (R32→R7), NO es lock secuencial literal sobre R23–R26 — Alberto confirmó pistas independientes. Diseño de gate fijado: R31 ∥ R23–R26 como Precondition Gate concurrente, OR-abort / AND-success, aguas arriba de toda lógica row-level de Polars (Paso 3.2). R32 excluido del gate por diseño (falla por-obra, exclusión acotada vs. abort global de R31/R25/R26) — Paso propio (3.3). Auditoría de cobertura: se forzaron dos pistas de error distintas por radio de explosión divergente (global vs. por-obra), y se agregó Paso terminal de verificación en vivo (3.4) homólogo al 2.5 de Fase 2. Manejo de error transitorio de Drive API decidido IN SCOPE (Alberto, Opción 1): retry-before-abort hasta 3 intentos, params de despliegue, cobertura extendida a 3.2.1/3.2.2/3.2.3 (3.2.3 extendido sobre lo nombrado por Alberto, flagueado y aceptado por silencio constructivo — confirmar si se requiere override); diferenciación transitorio/definitivo normativa en todo 3.2; no amienda R25/R31. Postura de params de retry (config de despliegue, no constantes) propuesta como reversible de una línea, no objetada. Fase 3 cerrada en su totalidad (roadmap + 17 tareas atómicas, todas [BACKLOG]). PENDIENTE no bloqueante arrastrado: esquema fino de la cola de anomalías (R2/R6.1/R22), aún diferido. Próximo foco: apertura de Fase 4 (BigQuery aggregation & service views — lado productor: R4-T1/R4-T2, ruteo R9/R10, vistas T1/T2/T3, partición de dataset R29, estampado run_date R27, link de authorized views).

**S19 — 2026-06-22**: Bloque 4. Foco: descomposición de Fase 4 (BigQuery Aggregation & Service
Views, lado productor) a nivel de roadmap (Paso). Gate de apertura (Modo Socrático): se forzó la
confirmación del contrato de salida de Fase 1 antes de descomponer Fase 4, ya que las 4 vistas
son puras lecturas contra stg_matched. Descubierto: Paso 1.8 portaba [CLOSED] sin enumerar
columnas — la interfaz Polars→BigQuery (corte D-10.3) existía solo por nombre (1.8.1 contenía el
esquema obsoleto con N1..N5 y run_date mal atribuido a Fase 4). Derivación de las 10 columnas
desde reglas cerradas, confirmada por Alberto; drop de N1..N5/actividad_norm/tipologia_norm
aprobado (staging lean, forense cubierto por recon_nomatch). Catch arquitectónico: stg_matched
tiene dos caras de tipo — el intento de inyectar run_date + cast STRING en 1.8 habría roto 1.8.2
(DC#3, "schema exactamente 1.8.1" validado contra fixture 1.1.4) porque un frame local no tiene
fecha de Job que estampar; ambas transformaciones reasignadas a 3.4.3 (Fase 3, frontera de
escritura). Cast STRING para ids resuelto contra el overflow signado de INT64 (BigQuery sin
UINT64, ~50% de hashes FNV-1a-64 exceden INT64 max). Grano de R30 confirmado row-level (simetría
de mecanismo, no de grano; trazabilidad upstream). Decisiones estructurales de Fase 4 (a nivel
Paso): T1/T2 emparejadas (4.1), T3 propio (4.2), R30 propio en pr_staging (4.3), authorized-view
link como Paso dedicado (4.4, ventana transitoria aceptada), verificación terminal (4.5). Guard
IS NOT NULL obligatorio en R30 (NULL es miembro de R10) y IS NULL arm obligatorio en T2 (NULL/''
son miembros explícitos de R10) — registrados a nivel de diseño de Paso para arrastre a la
descomposición JIT. Fase 4 cerrada a NIVEL DE ROADMAP (5 Pasos); sin tareas atómicas. Decisión de
proceso mayor: descomposición atómica pasa a JIT/incremental desde Fase 4 inclusive (Task
Protocol de CLAUDE.md, MODE PLANNING→EXECUTION); Bloque 5 disuelto (development spec se escribe
en ejecución); solo Fases 0–3 conservan atómicos congelados como referencia. Secuencia a
ejecución fijada: terminar roadmap.md (Fase 5 nivel Paso) → Bloque 6 (CLAUDE.md, precondición
dura) → construir. Correcciones de docs aplicadas a fases ejecutables: 1.8.1 reescrito (9-col),
3.4.3 ampliado (STRING + run_date + 10-col target), fixture 1.1.4 reconciliada; Fase 4 inyectada
en roadmap.md a nivel Paso (no en development.md). Próximo foco: descomponer Fase 5 a nivel Paso
para cerrar roadmap.md, luego Bloque 6.
