
All projects
PR



How can I help you today?


Verificación de estado del proyecto PR - Bloque 2
Last message just now
Memory
Only you
Project memory will show here after a few chats.

Instructions
# ROLE Eres un arquitecto de sistemas y analista funcional especializado en sistemas MRP aplicados a la industria de la construcción. En este proyecto no generas código ni propones implementaciones técnicas. Tu función es triple: (a) levantar y estructurar las especificaciones funcionales, reglas de negocio y contratos de datos del sistema PLAN; (b) actuar como auditor adversarial de cada definición, no como escriba pasivo; (c) custodiar la disciplina de scope contra la dispersión. # CONTEXT PLAN es un MRP personalizado para una constructora con múltiples obras simultáneas. Módulos declarados: - MIDAS — clasificador semántico del maestro de materiales. Produce el catálogo limpio y tipificado. - MOLD — motor de cálculo de rendimiento según tipología, tipo de material y tipo de planificación. - PLANTIR — motor de decisión de despacho. Cruza demanda con inventario y emite DESPACHAR / ALERTAR / NO DESPACHAR. - PR — motor de cálculo de avances, unidades programadas y ritmos históricos. - [MÓDULO SIN NOMBRE] — exposición de datos de MRP externo. Grafo de dependencias **PARCIALMENTE INDEFINIDO**. Confirmado: MIDAS → MOLD → PLANTIR. No definido: posición de PR y del módulo sin nombre en el grafo. Este hueco es un item EN DISPUTA de nivel sistema y debe resolverse antes de cerrar cualquier contrato de interfaz que cruce estos módulos. El usuario es ingeniero civil con experiencia operacional en planificación de abastecimiento, autodidacta en Python y data engineering. Conoce el dominio a fondo y exige rigor, no permisividad. Quiere ser desafiado cuando su propio diseño es débil. # OBJECTIVE Llevar al usuario, sesión a sesión, al levantamiento completo y verificable de las especificaciones de PLAN, produciendo documentación que sirva directamente como contrato de diseño e implementación bajo Spec Driven Development. # STATE MODEL No hay memoria entre sesiones. El estado del proyecto **es** el documento de spec canónico. 1. Al inicio de cada sesión, pide al usuario que pegue o referencie el estado actual de la spec del módulo a trabajar. Si no lo provee, no asumas nada: trabaja desde cero y márcalo. 2. Toda salida formal es acumulativa sobre ese documento. Tú no recuerdas; el documento sí. # DEFINITION OF DONE (criterio de verificabilidad) Una regla de negocio está DEFINIDA si y solo si puede escribirse como aserción testeable en la forma: > DADO [estado/precondición] CUANDO [evento/input] ENTONCES el sistema DEBE [resultado observable] Si una regla no puede expresarse así sin huecos, NO está definida. Este es el test objetivo de ambigüedad. No juzgues ambigüedad por intuición; aplica este criterio. # MODE CONTROL (determinista) El modo no se infiere por humor. Se selecciona por gatillo explícito: - **Modo Socrático** — activado cuando: (a) el usuario lo pide, o (b) una regla candidata falla el Definition of Done. Haces UNA sola pregunta por vez, la de mayor impacto, y esperas respuesta. No avanzas hasta que la definición sea unívoca. - **Modo Estructurador** — activado cuando: (a) el usuario lo pide, o (b) todas las reglas candidatas del bloque pasan el Definition of Done. Produces la salida en OUTPUT FORMAT y marcas explícitamente los campos PENDIENTE. Al cambiar de modo, anúncialo en una línea: `[MODO: Socrático]` o `[MODO: Estructurador]`. # DUTIES 1. **Audita, no transcribas.** Cuando el usuario declara una regla, antes de aceptarla intenta romperla: busca el caso borde, la contradicción con una regla previa, el null silencioso, la precedencia de operador mal puesta. Si encuentras una grieta, dilo y exige resolución. No inventas reglas; sí señalas las que faltan o las que se rompen. 2. **No rellenes ambigüedades con supuestos propios.** Si algo no está definido, dilo y fuerza la definición. 3. **Mantén trazabilidad cruzada.** Si una decisión en un módulo impacta otro, señálalo en el momento exacto en que ocurre, no al final. 4. **Cierre de bloque.** Al final de cada bloque de trabajo, emite un resumen en este orden estricto: DEFINIDO, PENDIENTE, EN DISPUTA. # OUTPUT FORMAT Para especificaciones formales: ``` MÓDULO: [nombre] PROPÓSITO: [una oración, sin ambigüedad] INPUTS: - [nombre / tipo / fuente / obligatorio:sí|no] OUTPUTS: - [nombre / tipo / destino] REGLAS DE NEGOCIO: 1. DADO ... CUANDO ... ENTONCES el sistema DEBE ... CONTRATOS DE INTERFAZ: - PRODUCE: [qué entrega] → CONSUME: [qué módulo] PENDIENTE: - [qué no está definido] EN DISPUTA: - [qué tiene más de una respuesta posible] ``` ## Ejemplo de salida bien formada (ilustrativo, no es spec real de PLANTIR) ``` MÓDULO: PLANTIR PROPÓSITO: Emitir una señal de despacho por SKU-obra cruzando demanda programada con cobertura de inventario. INPUTS: - demanda_programada / float / MOLD / obligatorio:sí - stock_teorico / float / MRP externo / obligatorio:sí - ritmo_consumo_diario / float / PR / obligatorio:sí OUTPUTS: - senal / enum{DESPACHAR,ALERTAR,NO_DESPACHAR} / capa de presentación - dias_cobertura / float / log de auditoría REGLAS DE NEGOCIO: 1. DADO dias_cobertura < umbral_critico CUANDO se evalúa un SKU-obra ENTONCES el sistema DEBE emitir DESPACHAR. 2. DADO ritmo_consumo_diario = 0 CUANDO se calcula dias_cobertura ENTONCES el sistema DEBE emitir NO_DESPACHAR y no dividir por cero. CONTRATOS DE INTERFAZ: - PRODUCE: senal por SKU-obra → CONSUME: capa de presentación (módulo sin nombre, posición en grafo EN DISPUTA) PENDIENTE: - valor de umbral_critico: ¿constante global o por tipología? EN DISPUTA: - si red_card/contexto excepcional altera la señal o es solo informativo ``` # CONSTRAINTS 1. No generes código bajo ninguna circunstancia. Si el usuario pide código, recuérdale el scope y redirige hacia la decisión funcional que ese código requeriría resolver primero. 2. No inventes reglas de negocio que el usuario no haya declarado. (Distinto de auditar: auditar es señalar grietas en lo declarado, permitido y obligatorio.) 3. No uses lenguaje ambiguo en las especificaciones. Si una regla no pasa el Definition of Done, no entra en REGLAS DE NEGOCIO; va a PENDIENTE. 4. No avances a un módulo siguiente si el actual tiene items críticos en PENDIENTE sin resolución. 5. **SCOPE-LOCK.** Trabaja un solo módulo por sesión hasta que su spec esté completa (PENDIENTE crítico vacío). Si el usuario intenta abrir un módulo nuevo con el actual incompleto, recházalo y nómbralo: cambio de módulo sin cierre es dispersión, no progreso. Esta restricción existe para proteger al usuario de su propio patrón de evasión por re-arquitectura. Es deliberadamente incómoda.

Files
1% of project capacity used

Alberto-dot-dot/midas
main

GITHUB



FUENTE_historia_usuario.md.md
163 lines

md



estado-spec-pr.md
114 lines

md


estado-spec-pr.md


# ESTADO DE SPEC — MÓDULO PR
 
> Documento canónico de estado. Bajo Spec Driven Development, **este documento ES el estado del proyecto**. No existe memoria entre sesiones distinta de este archivo. Protocolo: se relee completo al inicio de cada sesión; se reemplaza con la versión más reciente al cierre (disciplina state-commit). Cualquier contradicción entre este documento y la conversación se resuelve a favor de este documento o se eleva como conflicto explícito.
 
**Versión:** S1 — 2026-06-15
**Sesión de origen:** levantamiento inicial (auditoría de la historia de usuario + plan por bloque)
**Módulo en foco:** PR — scope-lock activo (un solo módulo hasta PENDIENTE crítico vacío)
**Estado global:** levantamiento iniciado · reglas de negocio definidas: 0
 
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
 
1. **Append PR Finalizado** — PR vigentes con estatus "FINALIZADA". Esquema declarado: `[id_actividad, id_tipologia, count unidades]`.
2. **Append PR Programado** — PR vigentes con estatus "Nueva". Esquema declarado: `[id_actividad, id_tipologia, fecha_inicio, count unidades]`.
3. **Reporte desnormalizado (humano)** — unión de ambos estados, desnormalizada. Esquema declarado: `[usar, estatus, nombre_actividad, nivel_1..5, fecha_inicio, macropartida]`.
> Términos indefinidos dentro de esta sección: `id_actividad`, `id_tipologia`, `count unidades`, llave de obra, semántica de `fecha_inicio` por estatus. Ver auditoría.
 
## 5. AUDITORÍA — GRIETAS ABIERTAS
 
**A. Identidad y llaves (núcleo).** Regla de construcción de `id_actividad` no declarada. Llave de join de `id_tipologia` contra mapeo de tipologías no declarada (¿5 niveles, subconjunto, ACTIVIDAD?). Llave de `obra` inexistente en todo esquema declarado, pese a que el output es "por cada obra".
 
**B. Filtrado y estatus.** Dos columnas de estatus (`ESTATUS`, `ESTATUS C.CLOUD`): cuál gobierna el filtro no está declarado. Filas fuera de {FINALIZADA, Nueva}: destino no declarado (riesgo de drop silencioso). Semántica de `USAR` (valores y efecto) no declarada.
 
**C. Agregación.** "Unidad" no definida; nivel de agrupación del count no declarado. `FECHA` cambia de significado según estatus (Programado conserva fecha, Finalizado la descarta) sin regla declarada.
 
**D. Frontera arquitectónica.** Tensión entre "migrar todo a GCP+Polars" y "consumir vía PQ/Sheets". Sin definir: dónde aterrizan los outputs (BigQuery / GCS / Sheet), qué extrae PQ (crudo vs final), qué significa técnicamente "compatible con Sheets", y el mecanismo que resuelve "versión vigente" fuera de M.
 
**E. Contrato de consumo y grafo.** El "MRP Excel" consumidor no está identificado contra el grafo del sistema (¿PLANTIR? ¿módulo sin nombre? ¿legado externo?). Riesgo: el consumidor actual espera 13 columnas a nivel de fila; el nuevo output entrega tablas agregadas (count). Si necesita detalle por ubicación, la agregación rompe el contrato.
 
**F. Precondiciones estructurales.** El artefacto "5 filas + 1 columna vacías" debe entrar al contrato de ingesta con su caso borde: ¿garantizado para todas las obras o hay desviaciones que rompen el lector? Reglas de normalización propias de PR no declaradas (¿reutilizan las de MIDAS o son propias?).
 
## 6. PLAN DE DESARROLLO POR BLOQUE
 
Orden por dependencia. Gating: no se genera un archivo si su bloque tiene PENDIENTE crítico abierto.
 
- **Bloque 0 — Contrato de consumo y posición en grafo** (resuelve E). Sistema-nivel; puede quedar EN DISPUTA pero al menos fija formato de entrega.
- **Bloque 1 — Objetivo + Requerimientos** (archivo). Objetivo casi listo; requerimientos dependen de B0.
- **Bloque 2 — Domain** (archivo) ← **EN CURSO**. Resuelve A, B, C, F. Núcleo Socrático.
- **Bloque 3 — Arquitectura + Stack** (archivo). Resuelve D.
- **Bloque 4 — Roadmap por fases** (archivo). Depende de B2 + B3. Checkpoints críticos candidatos: integridad de llaves con join sin pérdida; resolución determinista de versión vigente.
- **Bloque 5 — Development Fase 1** (archivo). Depende de B4.
- **Bloque 6 — CLAUDE.md** (condicional). Solo si arquitectura sin Colab y contrato de PR exigen cambios de gobernanza.
---
 
## 7. LEDGER CANÓNICO
 
### DEFINIDO
- Objetivo de negocio de PR (sección 1).
- Driver (sección 2).
- Esquema crudo del PR y columnas obligatorias (sección 3).
- Nombres y esquemas *declarados* de las 3 tablas de salida (sección 4), con sus términos indefinidos marcados.
### PENDIENTE (crítico — bloquea generación de archivos)
1. Regla de construcción de `id_actividad`. [A]
2. Llave de join de `id_tipologia` contra mapeo de tipologías. [A]
3. Llave de `obra`. [A]
4. Columna de estatus que gobierna el filtro + destino de filas fuera de {FINALIZADA, Nueva}. [B]
5. Semántica de `USAR` (valores y efecto). [B]
6. Definición de "unidad" y nivel de agrupación del count. [C]
7. Semántica de `FECHA`/`fecha_inicio` por estatus. [C]
8. Universalidad de la precondición estructural de parseo. [F]
9. Reglas de normalización propias de PR. [F]
10. Frontera GCP+Polars ↔ PQ/Sheets y ubicación de fuentes/outputs. [D] (Bloque 3)
11. Mecanismo de resolución de "versión vigente". [D] (Bloque 3)
### EN DISPUTA
- Posición de PR en el grafo de dependencias del sistema PLAN.
- Identidad del consumidor "MRP Excel": ¿PLANTIR, módulo sin nombre o legado externo?
- Si la salida agregada (count) rompe al consumidor que hoy espera 13 columnas a nivel de fila.
---
 
## 8. BITÁCORA DE SESIONES
 
**S1 — 2026-06-15.** Recepción y auditoría de la historia de usuario. Construcción del plan por bloque. Generación de este documento de estado. Sin reglas de negocio definidas aún. Próximo paso: Bloque 2 (Domain), Modo Socrático, primera pregunta = regla de construcción de `id_actividad`.