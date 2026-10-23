# Lab 4 — Oracle DBA Workflow I

**Entorno:** Oracle Database Free (versión 23.26.3.0.0) en Docker, CDB `FREE` y PDB `FREEPDB1`. Trabajo realizado en la rama `chore/3-oracle-dba-workflow-i`, asociada al Issue #3.

> Esta bitácora recoge las comprobaciones y conclusiones de los experimentos realizados. Las cifras de tiempos, procesos, tamaños y esperas corresponden a esta ejecución del laboratorio; pueden variar en otra sesión. Los archivos de `docs/bitacora/lab4/evidencia/` contienen las salidas originales.

## Parte A — Diagnóstico inicial

### 1. ¿Qué riesgos se detectaron antes de intervenir?
Se identificaron cinco datafiles de negocio referenciados desde rutas no estándar bajo `$ORACLE_HOME/dbs`, aunque sus destinos reales estaban dentro del volumen persistente; contraseñas de esquemas incorporadas a migraciones originales; paso de credenciales de SYS como argumentos; y publicación del puerto 8181 sin necesidad demostrada. El riesgo principal era operativo y de seguridad, no una pérdida de datos ya constatada.

### 2. ¿Cómo se priorizaron las correcciones?
Primero se aseguró la autenticación y el manejo de credenciales; después se normalizaron las rutas físicas de los datafiles y el despliegue del contenedor. Antes de cambios potencialmente destructivos se prepararon copias de seguridad. Se mantuvo el historial de migraciones ya aplicadas, introduciendo cambios mediante migraciones nuevas.

### 3. ¿Cómo se verificó el respaldo?
Se comprobó la integridad del respaldo del Lab 3 mediante `sha256sum -c --quiet SHA256SUMS`. El verificador final confirmó que existía un backup íntegro y que las copias de seguridad no estaban versionadas en Git.

## Parte B — Observabilidad

### 4. ¿Para qué se creó `C##OPS_RO`?
Para consultar el estado de Oracle sin trabajar habitualmente con SYSDBA. Es una cuenta común de observabilidad con permisos de lectura controlados, siguiendo el principio de mínimo privilegio.

### 5. ¿Qué diferencia hay entre observar y administrar?
Observar significa consultar vistas del diccionario y vistas dinámicas, sin realizar DML ni cambios de configuración. Administrar como SYSDBA permite ejecutar operaciones de mayor impacto. El verificador comprobó que `C##OPS_RO` no tenía privilegios `ANY` de escritura y que podía leer la vista de diagnóstico autorizada.

## Parte C — Persistencia

### 6. ¿Qué se corrigió en los datafiles y por qué?
Los cinco archivos de tablespaces de negocio quedaron ubicados en `/opt/oracle/oradata/FREE/FREEPDB1/`, dentro del volumen persistente de Docker. Sus tablespaces son `TBS_ACADEMIA`, `TBS_CLINICA`, `TBS_FINTECH`, `TBS_LOGISTICA` y `TBS_RETAIL`.

### 7. ¿Cómo se demostró la persistencia?
Se recreó el contenedor conservando el volumen y se comprobó que la PDB, los tablespaces y sus datos seguían accesibles. La verificación final devolvió cero datafiles de negocio fuera de la carpeta esperada y cero rutas reales que salieran del volumen.

### 8. ¿Qué ocurrió con el puerto 8181?
Se corrigió el despliegue para dejar de publicar ese puerto innecesario. La comprobación automática confirmó que Docker ya no lo exponía.

## Parte D — Migraciones

### 9. ¿Cómo se corrigió la gestión de contraseñas?
La migración V003 cambió las cuentas `ADMIN_*` a `NO AUTHENTICATION`, adecuado para esquemas propietarios de objetos sin inicio de sesión directo. Se verificó `AUTHENTICATION_TYPE = NONE` y que ninguna cuenta `ADMIN_*` conservara autenticación por contraseña.

### 10. ¿Qué datos incorporó V002?
La carga de demostración generó 20.000 estudiantes, 40 asignaturas y 60.000 matrículas en Academia. Estos tres recuentos se volvieron a comprobar en la verificación final.

### 11. ¿Por qué usar autenticación local para SYSDBA?
Evita proporcionar la contraseña administrativa como argumento del comando, donde podría exponerse en el historial o en la lista de procesos. Los scripts administrativos pasaron a usar autenticación del sistema operativo desde el contenedor cuando correspondía.

## Parte E — Instancia

### 12. ¿En qué se diferencia una instancia de una base de datos?
La instancia reúne memoria y procesos de Oracle. La base de datos incluye sus archivos persistentes: datafiles, control files y redo logs. Parar la instancia no equivale a eliminar esos archivos.

### 13. ¿Qué fases de arranque se comprobaron?
Se ensayaron las transiciones `SHUTDOWN IMMEDIATE`, `NOMOUNT`, `MOUNT` y `OPEN`. En `NOMOUNT` ya hay instancia, pero todavía no está montada la base; en `MOUNT` se conocen sus archivos de control y en `OPEN` los datos quedan disponibles. Algunas consultas producen errores esperados si se ejecutan antes de la fase adecuada.

### 14. ¿Cuál fue el estado final?
`FREE` quedó `OPEN` y `ACTIVE`, con base de datos `READ WRITE`. `FREEPDB1` quedó también `READ WRITE`; `PDB$SEED` permaneció `READ ONLY`.

## Parte F — Parámetros

### 15. ¿Qué diferencia hay entre cambiar un parámetro en memoria y en SPFILE?
`SCOPE=MEMORY` afecta a la instancia en ejecución; `SCOPE=SPFILE` prepara el cambio para el siguiente arranque; `SCOPE=BOTH` afecta a los dos ámbitos cuando el parámetro lo permite. No todos los parámetros admiten todos los ámbitos.

### 16. ¿Se restauró la configuración inicial?
Sí. `open_cursors` volvió a 300; `db_files` quedó sin una entrada explícita pendiente en el SPFILE. Se guardaron PFILE de evidencia para documentar la configuración y la versión en memoria.

## Parte G — Memoria

### 17. ¿Qué funciones tienen la SGA y la PGA?
La SGA es memoria compartida que utiliza la instancia para la caché de bloques, el shared pool y otras estructuras. La PGA es memoria privada de los procesos, utilizada entre otras operaciones para ordenaciones. En la autopsia final se observaron `sga_target` de 1.536 MB y objetivo PGA de 512 MB.

### 18. ¿Qué demostró el experimento de caché?
La primera lectura de la muestra requirió lecturas físicas, mientras que una repetición con bloques en caché redujo esas lecturas a cero en la prueba registrada. Se demuestra por qué el estado de la caché afecta al coste real de una consulta.

### 19. ¿Qué se observó al comparar literales y variables vinculadas?
Las sentencias construidas con literales distintos causaron muchas más operaciones de análisis duro (*hard parse*) que las ejecuciones equivalentes mediante variables vinculadas (*bind variables*). En la prueba se registraron aproximadamente 1.008 frente a 1; reutilizar SQL favorece el aprovechamiento del shared pool.

### 20. ¿Qué sucede si se reduce demasiado la memoria para ordenar?
Una ordenación puede dejar de resolverse íntegramente en memoria y utilizar espacio temporal en disco, con un incremento del tiempo de ejecución. En el experimento se observaron unos 20 ms sin uso de disco y unos 170 ms al forzar una memoria de ordenación muy pequeña, con una pasada a disco.

## Parte H — Procesos

### 21. ¿Qué procesos de Oracle se identificaron?
Se relacionaron procesos de segundo plano como `PMON`, `SMON`, `DBW0`, `LGWR`, `CKPT` y `ARC0`–`ARC3` con los procesos visibles desde Linux. Sus funciones incluyen recuperación y limpieza, escritura de bloques, escritura de redo, checkpoints y archivado.

### 22. ¿Por qué los recuentos de procesos pueden cambiar?
Oracle crea y finaliza procesos según el trabajo y el momento de observación. Por eso las cantidades no son constantes. En la autopsia final aparecieron 84 procesos de segundo plano y 9 procesos de servidor u otros.

## Parte I — Contenedores

### 23. ¿Qué son CDB$ROOT, PDB$SEED y FREEPDB1?
`CDB$ROOT` es el contenedor raíz, `PDB$SEED` la plantilla de solo lectura para crear nuevas PDB y `FREEPDB1` la PDB de trabajo. Cada una tiene su propio contexto de objetos y privilegios, aunque comparten una instancia.

### 24. ¿Cómo se probó el ciclo de vida de una PDB?
Se creó `PDB_LAB4`, se abrió, se cerró y se eliminó con `INCLUDING DATAFILES`. Después se comprobó que no permanecía registrada ni conservaba sus archivos. El error `ORA-65025` observado durante el ensayo correspondió al intento de borrado cuando no se cumplía el estado requerido.

### 25. ¿Qué se comprobó sobre las cuentas comunes y locales?
Se verificó qué vistas y objetos estaban disponibles desde distintos contenedores y con distintos privilegios. Los usuarios temporales de ensayo se eliminaron, y el verificador confirmó que no quedaban cuentas de prueba `*LAB4*`.

## Parte J — Archivos

### 26. ¿Cuáles son los archivos principales de Oracle?
En esta ejecución se inventariaron 16 datafiles (4.004 MB), 3 tempfiles (60 MB), 2 control files (36 MB) y 3 miembros de redo online (600 MB). También se registraron 5 redo logs archivados (aproximadamente 249 MB).

### 27. ¿Son equivalentes redo online, redo archivado, SPFILE y ADR?
No. El redo online recoge cambios para recuperación; el archivado conserva copias de redo ya completado; el SPFILE guarda parámetros de arranque; y ADR guarda trazas y diagnósticos. Se confirmó `ARCHIVELOG`, sin FRA configurada todavía, y que `C##OPS_RO` podía consultar `V_$DIAG_ALERT_EXT`.

### 28. ¿Qué riesgo de persistencia se descubrió en el ADR?
El directorio `/opt/oracle/diag` estaba fuera del volumen persistente. Si se elimina y recrea el contenedor, los registros de diagnóstico de su capa de archivos pueden perderse. Conviene exportarlos o persistirlos en un entorno estable.

## Parte K — Bloques

### 29. ¿Cómo se relacionan tablespace, segmento, extensión, bloque y fila?
Un tablespace organiza el espacio lógico y sus archivos; un segmento corresponde al almacenamiento de un objeto, dividido en extensiones, y cada extensión contiene bloques. Las filas se almacenan en los bloques de la tabla. El tamaño del bloque utilizado fue de 8.192 bytes.

### 30. ¿Dónde estaba físicamente el estudiante 101?
Su `ROWID` correspondía al datafile 16, bloque 779, ranura 5, en `/opt/oracle/oradata/FREE/FREEPDB1/academia01.dbf`. El volcado físico mostró `tab 0, row 5`, los valores de sus columnas y los metadatos de la cabecera del bloque.

### 31. ¿Qué se observó sobre el espacio?
El segmento `ACA_ESTUDIANTES` tenía 17 extensiones, 256 bloques asignados y 2 MB de tamaño. `DBMS_SPACE.SPACE_USAGE` indicó 230 bloques llenos, uno con entre 25 % y 50 % libre y 13 con entre 75 % y 100 % libre en ese momento.

## Parte L — Capacidad

### 32. ¿Qué es el factor de bloqueo?
Es el número de filas que caben en un bloque. Depende del tamaño del bloque, del tamaño de las filas, de la sobrecarga interna y de la reserva `PCTFREE`. La tabla de estudiantes contenía 20.000 filas distribuidas en 231 bloques con filas, con una media observada de 86,6 filas por bloque.

### 33. ¿Coincidieron las fórmulas con la medida real?
No exactamente. La fórmula de la unidad estimó 95 filas por bloque y 211 bloques; el ajuste aproximado de Oracle estimó 93 filas por bloque y 216 bloques. Las estadísticas registraron 244 bloques bajo la marca de agua alta (*HWM*). Las cabeceras, la ocupación desigual y el espacio reservado explican parte de esa diferencia.

### 34. ¿Qué diferencias de rendimiento se midieron?
En la tabla de pruebas de 400.000 filas, una lectura lógica costó aproximadamente 1,5 microsegundos por lectura. El ensayo aleatorio estimó 9 microsegundos por lectura física. El recorrido secuencial leyó 4.595 bloques en unas 53 peticiones, con un tiempo de 20,5 ms y rendimiento calculado de 1.749 MB/s. No se trata de latencias de disco aislado: la caché de páginas de Linux puede intervenir aunque se vacíe la caché de Oracle.

## Parte M — Filas migradas

### 35. ¿Qué diferencia hay entre migración y encadenamiento de filas?
Una fila migra cuando crece y deja de caber en el bloque donde se insertó; se conserva allí una referencia a su nueva ubicación. Una fila se encadena cuando su tamaño no permite almacenarla entera en un único bloque y debe dividirse entre varios.

### 36. ¿Cómo se provocó y detectó la migración?
Se creó `MIGRA` con 5.000 filas y `PCTFREE 0`. Al ampliar una de cada cinco filas, `CHAIN_CNT` pasó de cero a 1.000 (20 %). `ANALYZE ... LIST CHAINED ROWS` identificó también 1.000 filas. El volcado físico de una fila mostró una cabecera `fb: --H-----` y el puntero `nrid`.

### 37. ¿Qué coste y qué solución se comprobaron?
En 1.000 búsquedas se registraron 200 accesos a filas continuadas y 3.200 lecturas lógicas. `ALTER TABLE ... MOVE PCTFREE 20` reorganizó el segmento y eliminó la migración registrada (`CHAIN_CNT = 0`), pero dejó `PK_MIGRA` en estado `UNUSABLE`; `ALTER INDEX ... REBUILD` lo devolvió a `VALID`.

### 38. ¿Se puede reparar igual una fila encadenada por exceso de tamaño?
No necesariamente: una fila de aproximadamente 12 KB no cabe completa en un bloque de 8 KB. En la tabla `ANCHA` se registró una fila encadenada. Para evitar ese caso se requiere rediseñar la estructura o evaluar bloques de mayor tamaño, no solo reorganizar la tabla.

### 39. ¿Se limpiaron los recursos de prueba?
Sí. El script eliminó `LAB4_PRUEBAS` y `TBS_LAB4` con su datafile; la consulta final devolvió `TBS_LAB4_RESTANTES = 0`.

## Parte N — Verificación final

### 40. ¿Qué examinó la autopsia de la instancia?
Identidad de CDB/PDB, memoria SGA/PGA, procesos, archivos, almacenamiento, índices, usuarios, servicios, parámetros y alertas. Todos los índices de negocio examinados figuraron como válidos. Se observaron algunos índices sin estadísticas completas en varios esquemas de negocio, como oportunidad de mejora de optimización.

### 41. ¿Qué errores quedaron en el alert log?
Se registraron 6 mensajes con códigos ORA- en las últimas 24 horas de la autopsia: cuatro `ORA-00800`, uno `ORA-65025` y uno `ORA-00959` (presentado como `ORA-959` en el resumen). No se debe atribuir una causa precisa a este último sin revisar su mensaje completo. El experimento acabó con la instancia operativa.

### 42. ¿Qué garantiza la verificación automática?
El script `16-verificacion-final.sh` comprobó la integridad del backup, rutas persistentes, cuentas, número de registros, estado de la CDB y PDB, restauración de parámetros, eliminación de recursos temporales y presencia de evidencias. Todas las comprobaciones dieron `[OK]` y terminó con `RESULTADO: TODO OK. Lab 4 listo para entregar.`

### 43. ¿Qué queda para el cierre del trabajo?
Comprobar el contenido de esta bitácora frente al enunciado oficial, incorporarla al repositorio, verificar el estado de Git, subir la rama y abrir el Pull Request asociado al Issue #3. Los resultados técnicos ya se encuentran registrados en los archivos de evidencia.
