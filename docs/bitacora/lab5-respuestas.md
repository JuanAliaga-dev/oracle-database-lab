# Lab 5 — Respuestas, simulacro y retrospectiva

**Laboratorio:** Oracle DBA Workflow II · **Fecha:** 23/10/2026 · **Issue:** #5

Las cifras siguientes proceden de las ejecuciones de este laboratorio. Las evidencias están en `docs/bitacora/lab5/evidencia/{script,spool,config}/`, identificadas por el prefijo numérico `00`–`16`.

## 1. Preguntas de comprobación (1–22)

1. **Copia Lab 3 frente a Lab 4.** Según la revisión del laboratorio, en Lab 3 había cinco datafiles fuera del volumen persistente, frente a ninguno en la copia `lab4-final`. Demuestra que se corrigió la ubicación de los archivos y que la copia del volumen incluye toda la base. El backup `lab4-final` se validó íntegro (evidencia `00-backup-lab4`).
2. **Factor de bloqueo.** CLIENTE: 1.517 bloques para 100.000 filas, unas 66 filas/bloque; PEDIDO: 5.320 bloques para 1.000.000 de filas, unas 188 filas/bloque. PEDIDO tiene menor longitud media (32 B frente a 102 B), por eso caben aproximadamente 2,85 veces más filas por bloque. Evidencia `01-perfil`.
3. **Dos ejecuciones.** La primera puede provocar carga inicial en caché, trabajo de apertura o lectura física; medir la segunda reduce ese ruido y permite comparar los planes en caliente. Evidencias `01-herramienta`, `02-anatomia-indice`.
4. **Anatomía de IX_PED_CLIENTE.** Altura 3, 2.216 hojas y factor de ramificación real 370. Con mil millones de filas, la estimación del script es altura 4. Evidencia `02-anatomia-indice`.
5. **ROWID de PK_CLIENTE.** La primera entrada tiene clave `c1 02` (=1); el ROWID corto apunta al bloque 795 (`0000031B`) y ranura 0. En un bigfile tablespace, el identificador de archivo está implícito en el segmento y no aparece separado en esos seis bytes. Evidencia `02-volcado-indice`.
6. **Tres comparaciones de índices.** DNI: full scan 1.486 lecturas frente a 3 con `UNIQUE`. Q7: índice por cliente 15 frente a 3 con índice cubriente. Búsqueda de token: heap+PK 4 frente a clúster hash 1; este último penaliza los rangos. Evidencia `03-tipos-indice`.
7. **Clustering factor.** `IX_PED_FECHA` tiene CF 5.220 (cercano a los 5.320 bloques) porque la inserción mantiene proximidad física por fecha; `IX_PED_CLIENTE` tiene CF 999.041 (cercano al millón de filas) porque clientes consecutivos están dispersos. Reconstruir el índice no reordena la tabla, así que no corrige ese CF. Evidencia `04-agrupamiento-equilibrio`.
8. **Punto de equilibrio.** Con CF malo, Oracle pasó a full scan a partir del 0,2 % estimado; en lecturas medidas el índice malo supera 5.232 lecturas de full scan alrededor del 1 % (9.920). Con CF bueno el índice aún resulta ventajoso al 50 % (3.938 frente a 5.232), pero no al 100 % (7.875). Aproximación: 1/bfr = 1/188 ≈ 0,53 % en coste lógico, menor si las lecturas aleatorias son más caras. Evidencia `04-agrupamiento-equilibrio`.
9. **LIKE 'Gonz%'.** Es sargable porque el índice puede recorrer un rango por prefijo; sin embargo, accede a numerosas filas dispersas mediante ROWID. Se midieron 2.024 lecturas, por encima de las 1.523 del full scan de la otra consulta `LIKE '%ez'` (son predicados distintos, por lo que no es comparación idéntica). Evidencia `05-sargabilidad`.
10. **Escrituras e índices rechazados.** Con 4 índices, 10,1 bloques modificados/INSERT; con 9, 20,2; con V005 y 5, 12,1. Se descartan redundancias (`IX_PED_FEC_CLI` frente a fecha, `IX_PED_CLIENTE` cubierto por compuesto), el índice de canal por baja selectividad, el índice completo de estado a favor de parciales, y el de importe sin uso observado. El protocolo conservó fecha, código postal, compuesto cubriente y pendientes, además de PK. Evidencia `06-indices`.
11. **Tres patrones.** Aleatorio: 595 divisiones (ninguna 90/10), 75,4 % de ocupación. Creciente: 398 divisiones, todas 90/10, 99,9 % de ocupación. Cola: 285 hojas casi vacías a la izquierda y 89.903 entradas marcadas borradas. Evidencias `07a`, `07b` y gráficos.
12. **Remedio para cola.** `COALESCE` en línea: redujo la búsqueda del siguiente mensaje de 287 a 2 lecturas y eliminó entradas borradas; `SHRINK SPACE` además devolvió espacio (4 MB a 1,69 MB). A largo plazo, particionar por fecha y eliminar particiones completas. Evidencia `07c-remedios`.
13. **SCN en el arranque.** Oracle compara el SCN de checkpoint registrado en el control file con el de la cabecera del datafile. Antes, ambos eran 7.272.717; tras el checkpoint, 7.278.952. Evidencia `08-scn-checkpoint`.
14. **Multiplexación del control file.** Tres copias (`control01`, `control02`, `control03`) protegen contra pérdida de un archivo individual, pero aquí comparten volumen y no protegen contra la pérdida del disco entero. En producción se ubicarían en almacenamiento físicamente distinto y se mantendrían backups RMAN. Evidencia `09-multiplexar-controlfile`.
15. **Estados redo.** CURRENT recibe escrituras; ACTIVE ya no recibe nuevas escrituras, pero puede necesitarse para recuperar la instancia; INACTIVE ya no es necesario para esa recuperación. Se observaron secuencias 16–18 y seis miembros válidos. Oracle impidió borrar el grupo CURRENT (`ORA-01623`), su miembro (`ORA-01609`) y un grupo aún pendiente de archivado (`ORA-00350`). Evidencia `10-redo`.
16. **FRA llena.** Se redujo temporalmente a 2 MB, apareció aviso `ORA-19815`, luego `ORA-19809` y `ORA-16038` al no poder archivar la secuencia 20. Los cambios de log no pueden continuar si faltan grupos reutilizables; el servicio puede bloquearse. Al restaurar 4.096 MB se archivaron las secuencias 20 y 21. Evidencia `11-fra-llena`.
17. **Lectura consistente.** A actualizó la fila 101 sin COMMIT; B leyó `ACTIVO` desde undo sin bloquearse. A tenía bloqueo TM modo 3 y TX modo 6; B sí falló con `ORA-00054` al intentar `FOR UPDATE`. Tras COMMIT, B vio `EGRESADO` y Flashback recuperó `ACTIVO`. Evidencia `12-lectura-consistente`.
18. **Undo y redo.** Borrar obliga a guardar información para reconstruir la fila; INSERT requiere menos undo. Para 10.000 filas: INSERT 143 bloques undo y 4.873 KB redo; UPDATE 71 y 1.258 KB; DELETE 427 y 6.627 KB. Deshacer DELETE produjo otros 4.701 KB de redo: el rollback también modifica bloques que deben quedar protegidos. Evidencia `12-undo-y-redo`.
19. **Garantía de durabilidad.** Tras COMMIT, LGWR había hecho 3 escrituras y generado 1.779 KB de redo, mientras permanecían 231 bloques sucios de tabla; la garantía estaba en el redo persistente. Después de CHECKPOINT, DBWn escribió 346 bloques y los bloques sucios de tabla pasaron a cero. Evidencia `13-lgwr-dbwn`.
20. **Crash recovery.** En O.1, el registro confirmado de la fila 102 sobrevivió como `EGRESADO`; el cambio sin confirmar de la 103 se revirtió y quedó `ACTIVO`. El alert log mostró `Beginning crash recovery`, redo scan y redo application: primero roll forward y después rollback. Evidencia `14a-caida-instancia`.
21. **Simulacros C y D.** C restauró una copia completa: RTO 56 s y RPO potencial limitado al intervalo desde el backup (en la prueba, aproximadamente 1 s). D restauró un solo datafile con RMAN+redo: RTO 8 s y RPO 0. La recuperación física selectiva más los logs preserva cambios posteriores al backup. Evidencias `14c` y `14d`.
22. **Clasificación de errores del healthcheck.** `ORA-01110`, `ORA-01113`, `ORA-63999` son efectos del fallo de datafile simulado; `ORA-00202` y `ORA-00205` proceden del control file eliminado intencionadamente; `ORA-00800 [Set Priority Failed]` son advertencias del entorno y deben vigilarse si se repiten, pero aquí no impidieron abrir Oracle. Evidencia `15-healthcheck`.

## 2. Simulacro de examen S1–S26

| Pregunta | Respuesta | Justificación y parte |
|---|---|---|
| S1 | B | CF próximo a filas implica accesos dispersos a bloques (E). |
| S2 | A, C | Índice funcional o eliminar función de la columna (D, F). |
| S3 | B | Conversión implícita `TO_NUMBER` sobre columna VARCHAR2 (F). |
| S4 | B | Bitmap para baja cardinalidad en sistemas analíticos con poca escritura (D). |
| S5 | B | El prefijo izquierdo omitido puede dar lugar a `INDEX SKIP SCAN` (D). |
| S6 | B | INVISIBLE permite comprobar impacto sin eliminar el índice (G). |
| S7 | A, C | COALESCE fusiona hojas; SHRINK además libera espacio (H). |
| S8 | B | REVERSE distribuye inserciones pero inutiliza rangos ordinarios (H). |
| S9 | C | UPDATE de columna no indexada no mantiene las claves de esos índices (G). |
| S10 | B | El fan-out de las ramas determina el crecimiento logarítmico de altura (C). |
| S11 | A, C | La IOT almacena filas en el índice de PK y evita salto a heap (D). |
| S12 | B | Comparación de SCN de cabecera y control file (I). |
| S13 | C | CKPT coordina y registra los checkpoints (I). |
| S14 | B, E | En NOMOUNT existen instancia y parámetros, no diccionario/control file montado (I/J). |
| S15 | B | SHUTDOWN, STARTUP MOUNT, ARCHIVELOG y OPEN (L). |
| S16 | B | Se puede limpiar y recrear un grupo INACTIVE archivado mediante CLEAR LOGFILE (K). |
| S17 | A, C | Undo permite lectura consistente y Flashback Query (M). |
| S18 | B | El propio undo requiere redo para resistir caídas (M). |
| S19 | A | COMMIT confirma tras persistir redo online, no datafiles (N). |
| S20 | A | Menor objetivo MTTR conduce a checkpoints más agresivos (I). |
| S21 | B | ARCn bloqueado por destino de archivado lleno (L). |
| S22 | B | Datafile de aplicación OFFLINE para aislar el fallo cuando es posible (O). |
| S23 | B | SYSTEM exige recuperación con base montada (O). |
| S24 | B | En NOARCHIVELOG se vuelve a una copia fría y se pierde lo posterior (L, O). |
| S25 | B, D | Más undo/retención y RETENTION GUARANTEE cuando proceda (M). |
| S26 | A | `V$RECOVERY_AREA_USAGE` muestra uso de FRA por tipo (L). |

## 3. Tabla RTO/RPO: game days

| Simulacro | Fallo provocado | RTO | RPO | Evidencia |
|---|---|---|---|---|
| A (O.1) | Caída brusca de instancia | No cronometrado en la salida | 0 para transacciones confirmadas | `14a-caida-instancia` |
| B (O.2) | Pérdida de `control03.ctl` | No cronometrado en la salida | 0 | `14b-perdida-controlfile` |
| C (O.3) | `DROP TABLE ... PURGE` y restauración total | **56 s** | Ventana de aprox. 1 s desde el backup; los cambios posteriores a la copia no se conservan | `14c-simulacro-restauracion` |
| D (O.4) | Pérdida de `academia01.dbf` | **8 s** | **0** gracias a backup y redo | `14d-perdida-datafile` |

## 4. Retrospectiva de simulacros

- **A:** sorprendió la recuperación automática al abrir: el redo reconstruyó el estado y el undo revirtió lo no confirmado. Mejora: cronometrar automáticamente inicio del arranque y estado READ WRITE.
- **B:** una sola copia perdida impidió montar la base, incluso existiendo otras dos. Mejora: distribuciones de control files entre volúmenes independientes y comprobaciones periódicas.
- **C:** restaurar toda la base por una sola tabla demoró 56 s y revierte todos los cambios posteriores al backup. Mejora: priorizar Flashback cuando es aplicable; disponer de RMAN RECOVER TABLE y política de backups.
- **D:** RMAN restauró un único archivo en 8 s y el redo conservó la modificación posterior a la copia. Oracle dejó la PDB MOUNTED durante la reparación. Mejora: documentar ambos caminos (PDB abierta o montada), mantener redo archivado y practicar restauraciones.

### Incidente FRA llena — retrospectiva sin culpables

**Impacto:** al fijar la FRA en 2 MB se detuvo el archivado de redo; apareció `ORA-16038` y no se perdió información confirmada. **Detección:** `ORA-19815` y `ORA-19809` en alert log; grupo 20 pendiente de archivado. **Causa:** límite de recuperación insuficiente para el volumen de redo generado. **Mitigación:** devolver el límite a 4.096 MB; ARCn pudo archivar de nuevo. **Acciones preventivas:** alertar al 80 % de FRA, comprobar `V$RECOVERY_AREA_USAGE`, política RMAN de retención y limpieza respaldada, nunca borrar `.arc` indiscriminadamente en producción. Evidencia `11-fra-llena`.

## 5. Cierre y verificación

- `16-verificacion-final.sh`: **RESULTADO: TODO OK. Lab 5 listo para entregar.**
- TiendaSur: 100.000 clientes y 1.000.000 de pedidos; migración V005 aplicada.
- Tres control files, dos miembros por grupo de redo, ARCHIVELOG, FRA 4 GB y backups RMAN.
- Filas 101, 102 y 104 restauradas a `ACTIVO`; `ACA_MATRICULAS` restaurada con 60.000 registros.
- Evidencias `00` a `16` presentes. Los backups y secretos no deben versionarse.

**Nota:** las duraciones de A y B no se midieron expresamente en las salidas capturadas y por eso no se inventan cifras. La comparación con Lab 3 depende de conservar la evidencia de aquella copia.
