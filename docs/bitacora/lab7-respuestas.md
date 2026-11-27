# Laboratorio 7 · Respuestas · E002

<!--
CÓMO SE RELLENA
- Copia este archivo como docs/bitacora/lab7-respuestas.md y pon tu código en el título.
- Responde con TUS números: cada respuesta cita el archivo de evidencia de donde sale (docs/bitacora/lab7/evidencia/...).
  Una cifra sin evidencia no puntúa. Las cifras del PPTX son de referencia: las tuyas serán parecidas, no iguales.
- 3 a 6 líneas por pregunta. No pegues salidas enteras: el archivo de evidencia ya las tiene.
- No borres los títulos: scripts/lab7/09-verificacion-final.sh comprueba que están todos.
-->

## Parte B · C01 · Más sesiones no es más rendimiento

1. ¿Cuántas llamadas por segundo y qué p95 tuviste con 1, 2 y 8 sesiones? ¿Qué número de sesiones recomiendas y por qué?

- 1 sesión: 4,6 llamadas/s y p95 de 301,4 ms.
- 2 sesiones: 8,2 llamadas/s y p95 de 347 ms.
- 8 sesiones: 7,4 llamadas/s y p95 de 1713,6 ms.

Recomiendo 2 sesiones porque se obtiene el mayor rendimiento y se reduce la latencia aproximadamente un 80 % respecto a las 8 sesiones.

Evidencias: 20261126T164753Z_C01_entender.txt y 20261126T165046Z_C01_validar.txt.

2. ¿Qué evento domina con 8 sesiones y qué significa en Oracle Free?

El evento dominante es resmgr:cpu quantum, con 244 muestras, que representan el 74,4 % del total. Las sesiones tienen que esperar su turno para utilizar la CPU porque Oracle Free está limitado a 2 CPU. El AAS de 7,8 confirma la saturación.

Evidencia: 20261126T164808Z_C01_diagnosticar.txt.

3. Comprueba la ley de Little (N = X · R) con tus tres medidas.

- 1 sesión: 4,6 x 0,2087 = 0,96.
- 2 sesiones: 8,2 x 0,2409 = 1,98.
- 8 sesiones: 7,4 x 1,0536 = 7,80.

Los resultados coinciden aproximadamente con el número de sesiones de cada prueba, por lo que se cumple la ley de Little.

Evidencia: 20261126T165102Z_C01_observar.txt.

## Parte C · C03 · Literales y análisis completos

1. Análisis completos y tiempo con literales, con variables de enlace y con CURSOR_SHARING = FORCE (tus cifras).

- LITERAL: 5047 hard parses y 4650 ms.
- BIND: 28 hard parses y 330 ms.
- FORCE: 28 hard parses y 540 ms.

Las variables de enlace reducen los análisis completos y permiten ejecutar la misma carga unas 14 veces más rápido.

Evidencia: 20261126T165647Z_C03_observar.txt.

2. ¿Por qué FORCE es un puente y no la solución? ¿Qué ve el DBA en V$SQL que lo delata (FORCE_MATCHING_SIGNATURE)?

FORCE sustituye automáticamente los literales por variables, pero no corrige el problema de la aplicación y puede afectar a la selectividad. La solución definitiva es utilizar variables de enlace.
En V$SQL aparecen 687 SQL_ID distintos con la misma FORCE_MATCHING_SIGNATURE y una ejecución por sentencia.

Evidencias: 20261126T165641Z_C03_diagnosticar.txt y 20261126T165647Z_C03_observar.txt.

## Parte C · C04 · El informe que se desborda a TEMP

1. Bloques de TEMP, tipo de pasada y tiempo del informe antes y después. ¿Qué hacía el disparador de logon?

- LEGADO: 5456 bloques escritos en TEMP, ONE PASS y 2810 ms.
- AUTO_COMPLETO: 0 bloques, OPTIMAL y 1180 ms.
- AUTO_TOP100: 0 bloques, OPTIMAL y 130 ms.

El disparador C04_LEGADO establecía WORKAREA_SIZE_POLICY = MANUAL y limitaba SORT_AREA_SIZE a 1 MB.

Evidencias: 20261126T165701Z_C04_diagnosticar.txt y 20261126T165707Z_C04_observar.txt.

2. ¿Por qué el TOP 100 cabe en memoria y el informe completo no?

El TOP 100 solo necesita mantener las 100 mejores filas durante la ordenación, mientras que el informe completo procesa un millón de filas. Con la configuración heredada, el informe necesitaba 48 MB y disponía de aproximadamente 1 MB, por lo que utilizaba TEMP. Con la gestión automática, ambos consiguen ejecutarse en memoria.

Evidencias: 20261126T165701Z_C04_diagnosticar.txt y 20261126T165707Z_C04_observar.txt.

## Parte D · C07 · EXPLAIN PLAN frente al plan real

1. Copia el predicado que delata el problema en el plan real y explica de dónde sale.

filter(TO_NUMBER("CODIGO_POSTAL")=:B1)

La aplicación enviaba la variable como NUMBER, mientras que CODIGO_POSTAL es de tipo texto. Oracle realizaba una conversión implícita sobre la columna, impidiendo utilizar el índice y provocando un TABLE ACCESS FULL.

Evidencia: 20261126T170133Z_C07_diagnosticar.txt.

2. Lecturas y p95 de V1 y V2. ¿Por qué EXPLAIN PLAN no lo vio?

- V1: 5227 lecturas y p95 de 156 ms.
- V2: 95 lecturas y p95 de 1,1 ms.

EXPLAIN PLAN no ejecuta la consulta ni conoce el tipo real de la variable. DISPLAY_CURSOR muestra el plan utilizado durante la ejecución. Al corregir el tipo de la variable, Oracle utiliza el índice IX_T6_PED_CP.

Evidencias: 20261126T170131Z_C07_entender.txt y 20261126T170136Z_C07_validar.txt.

## Parte D · C08 · E-Rows frente a A-Rows

1. Estimación y filas reales antes y después del grupo de columnas. ¿En qué línea del plan se separan por primera vez?

- Antes: 3 filas estimadas y 56 reales.
- Después: 50 filas estimadas y 56 reales.

La primera desviación importante aparece en la línea 3 del plan, TABLE ACCESS BY INDEX ROWID BATCHED sobre CLIENTE, donde se aplica el filtro de COMUNIDAD.

Evidencias: 20261126T170146Z_C08_diagnosticar.txt y 20261126T170148Z_C08_solucionar.txt.

2. ¿Por qué el optimizador multiplicaba mal las dos selectividades?

El optimizador consideraba CIUDAD y COMUNIDAD como columnas independientes, cuando realmente están correlacionadas, ya que cada ciudad pertenece a una comunidad. Al crear estadísticas extendidas sobre ambas columnas, Oracle tiene en cuenta sus combinaciones y mejora la estimación de 3 a 50 filas.

Evidencias: 20261126T170144Z_C08_entender.txt y 20261126T170150Z_C08_validar.txt.

## Parte E · C12 · Estadísticas obsoletas

1. Filas según las estadísticas y reales; método de unión y lecturas antes y después.

Antes: las estadísticas indicaban 100 filas, pero había 200100 filas reales. Oracle utilizaba NESTED LOOPS y realizaba 212313 lecturas lógicas.
Después de actualizar las estadísticas, Oracle estima 200100 filas, utiliza HASH JOIN y realiza 5907 lecturas lógicas.

Evidencias: 20261126T170514Z_C12_diagnosticar.txt y 20261126T170518Z_C12_validar.txt.

2. ¿Por qué la tarea automática no lo habría arreglado a tiempo? ¿Quién debe recoger y cuándo?

La tarea automática recoge las estadísticas durante las ventanas de mantenimiento. En mi entorno, la siguiente ventana comenzaba a las 22:00 y duraba 4 horas.
El propio proceso de carga debe recoger las estadísticas al finalizar mediante DBMS_STATS.GATHER_TABLE_STATS.

Evidencia: 20261126T170521Z_C12_observar.txt.

## Parte E · C13 · El ciclo de estadísticas con red

1. Describe el ciclo que seguiste (pendientes, probar, publicar, restaurar, bloquear) con la vista que comprueba cada paso.

- Pendientes: PUBLISH = FALSE y comprobación en DBA_TAB_PENDING_STATS.
- Probar: OPTIMIZER_USE_PENDING_STATISTICS = TRUE y comprobar PLAN_TABLE. La estimación pasó de 250000 a 185277 filas.
- Publicar: PUBLISH_PENDING_STATS y comprobar DBA_TAB_STATISTICS.
- Restaurar: RESTORE_TABLE_STATS, consultando DBA_TAB_STATS_HISTORY y comprobando DBA_TAB_COL_STATISTICS.
- Bloquear: LOCK_TABLE_STATS y comprobar STATTYPE_LOCKED = ALL en DBA_TAB_STATISTICS.

Evidencias: 20261126T170709Z_C13_entender.txt, 20261126T170717Z_C13_solucionar.txt y 20261126T170726Z_C13_observar.txt.

2. ¿Qué error obtuviste al recoger sobre estadísticas bloqueadas y cómo se recoge si de verdad hace falta?

Obtuve el error ORA-20005: object statistics are locked (stattype = ALL).
Si es necesario recoger las estadísticas, se puede utilizar GATHER_TABLE_STATS con force => TRUE o desbloquearlas previamente mediante UNLOCK_TABLE_STATS.

Evidencia: 20261126T170726Z_C13_observar.txt.

## Parte E · C14 · El índice invisible

1. Lecturas por apertura del panel antes y después. ¿Cómo probaste el índice sin que lo viera nadie más?

Antes: 5265 lecturas lógicas y 31,3 ms.
Después: 118 lecturas lógicas y 0,58 ms.
Creé el índice IX_T6_C14_PEND como INVISIBLE y activé OPTIMIZER_USE_INVISIBLE_INDEXES = TRUE únicamente en mi sesión. Tras comprobar la mejora, lo hice VISIBLE para todas las sesiones.

Evidencias: 20261126T170732Z_C14_provocar.txt, 20261126T170740Z_C14_solucionar.txt y 20261126T170741Z_C14_validar.txt.

2. Redo con el índice INVISIBLE y VISIBLE: ¿qué demuestra?

- Sin mantener el índice: 9007 KB.
- Índice INVISIBLE: 10402 KB.
- Índice VISIBLE: 10402 KB.

Esto demuestra que un índice invisible tiene el mismo coste de mantenimiento durante las operaciones DML que uno visible. La diferencia es que el optimizador no lo utiliza por defecto.

Evidencia: 20261126T170745Z_C14_observar.txt.

## Parte F · C17 · INC-604 de principio a fin

1. SLI y SLO; p95 antes y después; instantáneas AWR que usaste.

- SLI: latencia p95 de las búsquedas.
- SLO: p95 inferior a 20 ms.
- Antes: 63,3 ms.
- Después: 1,07 ms.
- Intervalo AWR inicial: snapshots 21 y 22.

Evidencias: 20261126T172120Z_C17_entender.txt y 20261126T172342Z_C17_validar.txt.

2. ¿Qué dijeron ASH, AWR y ADDM, y qué no podían decirte? ¿Cuál fue el ÚNICO cambio?

- ASH: identificó el SQL 0dbx7196jdajy, con 421 muestras ON CPU y 52 esperando CPU.
- AWR: registró 4321 ejecuciones y 5039 lecturas por ejecución.
- ADDM: identificó las sentencias SQL principales como un problema con un impacto del 90,63 %.
- El plan real permitió identificar la causa: TO_NUMBER sobre CODIGO_POSTAL.
- Único cambio: sustituir la V1 (NUMBER) por la V2 (VARCHAR2), permitiendo utilizar el índice IX_T6_PED_CP.

Evidencias: 20261126T172122Z_C17_diagnosticar.txt y 20261126T172123Z_C17_solucionar.txt.

## Parte F · C18 · SQL Plan Management

1. PLAN_HASH_VALUE y lecturas del lunes, del martes y con la línea base.

- Lunes: plan 1767728868 y 195 lecturas por ejecución.
- Martes: plan 2571571058 y 5227 lecturas por ejecución.
- Con SPM: plan 1767728868 y 192 lecturas por ejecución.

Evidencias: 20261126T171517Z_C18_entender.txt y 20261126T171525Z_C18_validar.txt.

2. ¿Qué hizo la evolución con el plan nuevo y por qué? ¿Qué queda pendiente para que no vuelva a pasar?

- La evolución rechazó el nuevo plan porque no superó el criterio de beneficio.
- El plan anterior permanece aceptado y activo.
- El nuevo permanece como ACCEPTED = NO.
- Queda pendiente corregir las preferencias de recogida de estadísticas para conservar el histograma de ESTADO.

Evidencia: 20261126T171528Z_C18_observar.txt.

## Parte G · Incidente 1 · INC-605

- **Síntoma (con números):** El informe de cestas abandonadas tarda más tras una carga de 152000 filas nuevas. Las estadísticas indican 100 filas frente a 152100 reales.

- **Playbook y pasos seguidos (herramientas y vistas):** PB-3 y RB-2. Utilicé t6 estad CESTA, t6 top 10 elapsed, DBA_TAB_STATISTICS y SELECT COUNT(*) FROM t6.cesta.

- **Causa raíz:** Estadísticas obsoletas (STALE_STATS = YES) y bloqueadas (STATTYPE_LOCKED = ALL), provocando una estimación incorrecta de la cardinalidad.

- **Arreglo aplicado (órdenes exactas) y por qué ese:**
  EXEC DBMS_STATS.UNLOCK_TABLE_STATS('T6', 'CESTA');
  EXEC DBMS_STATS.GATHER_TABLE_STATS('T6', 'CESTA', no_invalidate => FALSE);
  Desbloqueé y actualicé las estadísticas para corregir la estimación del optimizador.

- **Verificación (archivo de evidencia con todo OK):** 152100 filas estimadas y reales. El informe utiliza HASH JOIN. Todas las comprobaciones dan OK.
  Evidencia: 20261126T173214Z_INC-605_verificar.txt.

- **Prevención (qué evitaría que se repita):** Mantener las estadísticas desbloqueadas y recogerlas después de cada carga masiva. La tarea automática no podía actualizarlas debido al bloqueo.

## Parte G · Incidente 2 · INC-606

- **Síntoma (con números):** La búsqueda por código postal de la versión V2 realizaba 5311 lecturas lógicas por búsqueda.

- **Playbook y pasos seguidos (herramientas y vistas):** PB-2 y RB-3. Utilicé t6 indices PEDIDO, t6 top 5 gets t6_cp, t6 plan 53mx77snb43uw y consulté DBA_INDEXES.

- **Causa raíz:** El índice IX_T6_PED_CP estaba en estado UNUSABLE aunque era VISIBLE. Esto provocaba un TABLE ACCESS FULL sobre PEDIDO, a pesar de que la aplicación utilizaba correctamente VARCHAR2.

- **Arreglo aplicado (órdenes exactas) y por qué ese:**
  ALTER INDEX T6.IX_T6_PED_CP REBUILD;
  Reconstruí el índice existente para recuperar su estado VALID sin modificar la aplicación ni crear otro índice.

- **Verificación (archivo de evidencia con todo OK):** El índice está VALID y VISIBLE. La V2 vuelve a utilizarlo y las lecturas por búsqueda se reducen de 5311 a 95.
  Evidencia: 20261126T173616Z_INC-606_verificar.txt.

- **Prevención (qué evitaría que se repita):** Monitorizar el estado de los índices mediante DBA_INDEXES y comprobar su utilización y rendimiento después de operaciones de mantenimiento o cargas masivas.

## Parte G · Incidente 3 · INC-612

- **Síntoma (con números):** La búsqueda de correo electrónico para recuperar la contraseña realizaba 1426 lecturas lógicas por búsqueda.

- **Playbook y pasos seguidos (herramientas y vistas):** PB-5 y RB-3. Utilicé t6 indices CLIENTE, t6 top 5 cpu y consulté las vistas DBA_INDEXES y DBA_IND_EXPRESSIONS.

- **Causa raíz:** Se había eliminado el índice basado en función sobre UPPER(email). El índice convencional sobre EMAIL no era adecuado para la consulta que aplica UPPER a la columna.

- **Arreglo aplicado (órdenes exactas) y por qué ese:**
  CREATE INDEX T6.IX_T6_CLI_EMAIL_UPPER
  ON T6.CLIENTE (UPPER(email))
  TABLESPACE TBS_T6;
  Creé un índice basado en función para optimizar la consulta sin modificar el código de la aplicación.

- **Verificación (archivo de evidencia con todo OK):** El índice es FUNCTION-BASED NORMAL, VALID y VISIBLE. La búsqueda vuelve a utilizar un índice y las lecturas se reducen de 1426 a 110.
  Evidencia: 20261126T173922Z_INC-612_verificar.txt.

- **Prevención (qué evitaría que se repita):** Antes de eliminar un índice, establecerlo como INVISIBLE y monitorizar su uso mediante DBA_INDEX_USAGE durante varias semanas. Comprobar que su retirada no perjudica al rendimiento antes de ejecutar DROP INDEX.
## Parte H · Reflexión

1. De todo el laboratorio, ¿qué cambio dio más mejora por menos riesgo? Justifícalo con tus números.

El cambio de NUMBER a VARCHAR2 en el caso C17.
La latencia p95 pasó de 63,3 ms a 1,07 ms y las
lecturas por ejecución de 5039 a 94.
Con la misma carga se pasó de 4412 a 33264 búsquedas.
El riesgo fue reducido porque no fue necesario modificar
las tablas ni crear índices adicionales.

2. Un caso no evaluable (sin ★) que hayas hecho y qué aprendiste de él (opcional, cuenta para la certificación).

No realizado. He completado los diez casos obligatorios
y los tres incidentes personalizados.
