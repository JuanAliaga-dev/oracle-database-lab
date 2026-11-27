# Laboratorio 6 — Respuestas de comprobación

**Alumno:** E002
**Práctica:** Oracle DBA Operations
**Fecha:** 19/11/2026

## Partes A-C. Rol operativo, parámetros y estados

### 1. Errores ORA- del alert log

En la Parte B se registraron 8 errores ORA- durante las últimas 24 horas, como aparece en la evidencia `01-turno.script.log`.

Ese número por sí solo no significa que exista una incidencia. El alert log también puede contener errores generados durante pruebas o situaciones que ya se han solucionado. Para saber si realmente hay un problema, habría que revisar los códigos, cuándo ocurrieron y si siguen afectando al funcionamiento de la base de datos.

En mi comprobación, la instancia estaba abierta, `FREEPDB1` funcionaba en modo `READ WRITE`, no había sesiones bloqueadas ni alertas pendientes. Por tanto, el contador no indicaba por sí mismo una incidencia activa.

**Evidencia:** `01-turno.script.log`, 8 errores ORA- en 24 horas.

### 2. Ventana de mantenimiento del miércoles

La ventana `WEDNESDAY_WINDOW` comienza a las 22:00 y tiene una duración de 4 horas. En mi instalación estaba habilitada.

Las tareas automáticas activas eran la recopilación de estadísticas del optimizador (`auto optimizer stats collection`), el asesor de espacio (`auto space advisor`) y el asesor de optimización SQL (`sql tuning advisor`).

Estas tareas permiten mantener las estadísticas actualizadas, analizar el uso del almacenamiento y detectar posibles mejoras en las consultas.

**Evidencia:** `01-automatizacion.spool.log`, `WEDNESDAY_WINDOW`, 22:00 y 4 horas.

### 3. Diferencia entre V$SPPARAMETER y V$PARAMETER

Al modificar `db_files` utilizando `SCOPE=SPFILE`, Oracle aceptó el nuevo valor de 250, pero el parámetro que estaba utilizando la instancia continuó siendo 200.

Esto ocurre porque `db_files` es un parámetro estático. `V$SPPARAMETER` permite consultar la configuración persistente del SPFILE, mientras que `V$PARAMETER` muestra la configuración utilizada en ese momento.

Si hubiera reiniciado la instancia sin deshacer el cambio, Oracle habría utilizado 250 en el siguiente arranque, aunque durante la ejecución anterior siguiera mostrando 200.

**Evidencia:** `02-parametros.script.log`, SPFILE = 250, memoria = 200.

### 4. Por qué se utilizó RESET para db_files

Antes de modificar `db_files`, comprobé que no tenía un valor establecido expresamente en el SPFILE. Por eso, la forma correcta de volver al estado anterior era utilizar `RESET`, eliminando el valor añadido durante la prueba.

No habría sido correcto aplicar automáticamente el mismo procedimiento a `processes`, porque primero habría que comprobar si tenía un valor propio en la configuración original. Si lo tuviera, eliminarlo no equivaldría a recuperar el estado previo.

La comprobación posterior confirmó que `db_files` volvía a aparecer con `ISSPECIFIED = FALSE`.

**Evidencia:** `02-parametros.script.log` y `02-init_desde_spfile.ora.txt`.

### 5. Estado de la PDB después de reiniciar

Durante las pruebas comprobé que el estado de la instancia y el de una PDB no son lo mismo. La instancia podía indicar que la base de datos estaba abierta mientras `FREEPDB1` no estaba disponible para ejecutar consultas normales.

En las pruebas de estados aparecieron errores relacionados con el acceso y la apertura de la PDB, entre ellos `ORA-01109`, porque no estaba abierta en el estado necesario.

Para solucionar este problema, se abrió `FREEPDB1` y se guardó su estado mediante `SAVE STATE`, de modo que pudiera recuperar ese estado tras futuros reinicios.

**Evidencia:** `02-estados-pdb.script.log` y comprobación con `validar-arranque.sql`.

## Partes D-E. Servicios, sesiones y diccionario

### 6. Identificación de la sesión de la aplicación

La sesión de la aplicación tenía `SID = 66`, `SERIAL# = 29801` y `SPID = 2367`. El evento registrado era `PL/SQL lock timer`, porque estaba ejecutando una espera controlada.

El usuario de conexión era `APP_ACADEMIA`, pero los campos adicionales aportaban información más concreta:

- `MODULE`: `matriculas`.
- `ACTION`: `cierre-periodo`.
- `CLIENT_IDENTIFIER`: `ana.garcia@ucam`.

Estos datos permiten identificar qué funcionalidad se estaba ejecutando y a qué cliente correspondía, algo que no podríamos saber únicamente mirando `USERNAME`, ya que diferentes usuarios pueden utilizar la misma cuenta de aplicación.

**Evidencia:** `03-sesiones.script.log`, SID 66 y SPID 2367.

### 7. Diferencia entre CANCEL SQL y KILL SESSION

Después de ejecutar `CANCEL SQL`, la aplicación todavía tenía una transacción abierta. El resultado mostraba `TRANSACCIONES_ABIERTAS_DE_LA_APP = 1`.

Esto demuestra que cancelar una sentencia no implica cerrar la sesión ni deshacer automáticamente una transacción que ya estaba abierta.

En cambio, al ejecutar `KILL SESSION`, la sesión terminó y el cambio que estaba pendiente se revirtió. La comprobación mostró 0 sesiones de aplicación vivas y el estudiante 101 conservó el estado `ACTIVO`.

**Evidencia:** `03-sesiones.script.log`, 1 transacción tras CANCEL SQL, 0 sesiones tras KILL SESSION y fila 101 en ACTIVO.

### 8. Valor de common_user_prefix

En mi instalación, `common_user_prefix` tenía el valor `C##`.

Esto significa que las cuentas comunes creadas normalmente desde la raíz del CDB necesitan utilizar ese prefijo. Por ejemplo, `C##OPS_RO` es una cuenta común.

En cambio, `APP_ACADEMIA` se creó como usuario local dentro de `FREEPDB1`, por lo que no necesitaba utilizar `C##`. La comprobación mostró `COMMON = NO`.

**Evidencia:** `03-cuenta-app.script.log`, prefijo C## y APP_ACADEMIA local.

### 9. Comparación de las tres miradas al diccionario

Los resultados dependieron tanto del usuario utilizado como del contenedor desde el que se realizó la conexión.

| Consulta | C##OPS_RO en CDB$ROOT | C##OPS_RO en FREEPDB1 | APP_ACADEMIA |
|---|---:|---:|---|
| ALL_TABLES de ADMIN_ACADEMIA | 0 | 0 | 1 |
| DBA_USERS | 37 | 47 | Sin permiso |
| CDB_USERS | 37 en CON_ID 1 y 47 en CON_ID 3 | 47 en CON_ID 3 | Sin permiso |
| V$DATAFILE | 17 | 10 | Sin permiso |

Una diferencia importante es que `C##OPS_RO`, al conectarse desde la raíz y con `CONTAINER_DATA` configurado, podía consultar información de varios contenedores.

Por otro lado, `APP_ACADEMIA` podía acceder a objetos concretos de la aplicación, pero recibió `ORA-00942` al intentar consultar vistas administrativas para las que no tenía permisos.

**Evidencia:** `04-tres-miradas.script.log`.

### 10. Lecturas lógicas y tiempo de base de datos

Durante la carga se realizaron diez recorridos completos sobre la tabla `PEDIDO`.

El resultado fue de 53.771 lecturas lógicas en total, aproximadamente 5.377 por recorrido. El tiempo de base de datos (`DB time`) fue de 0,42 segundos y el tiempo de CPU (`DB CPU`) de 0,36 segundos.

Como ambos tiempos son bastante parecidos, se puede deducir que durante esta medición la mayor parte del tiempo se dedicó a ejecutar trabajo en CPU, en lugar de esperar por recursos externos.

Además, este ejercicio demuestra por qué es más útil comparar dos mediciones mediante deltas que consultar únicamente los contadores acumulados desde el arranque.

**Evidencia:** `04-deltas.script.log`, 5.377 lecturas por recorrido, DB time de 0,42 s y DB CPU de 0,36 s.

## Partes F-G. Espacio y control de acceso

### 11. Espacio de TBS_ACADEMIA_HIST

Antes de realizar la carga, el tablespace `TBS_ACADEMIA_HIST` tenía 10 MB asignados, 3,9 MB libres dentro de ese espacio y un máximo de crecimiento de 40 MB.

Estas tres cantidades no representan lo mismo. El espacio asignado es lo que Oracle ya ha reservado, el espacio libre indica lo que todavía puede utilizar dentro de esa asignación y el máximo marca hasta dónde puede crecer.

Para configurar las alertas es importante tener en cuenta el límite máximo alcanzable, evitando que el tablespace llegue a quedarse sin capacidad. En el ejercicio establecí un aviso al 80 % y un nivel crítico al 95 %.

**Evidencia:** `05-almacenamiento.script.log`, 10 MB asignados, 3,9 MB libres y 40 MB máximos.

### 12. Cuotas y errores ORA-01950 y ORA-01536

En la prueba, la operación de inserción se realizó utilizando la cuenta de aplicación, pero el segmento pertenecía al esquema `CARGA_HIST`. Por tanto, quien necesitaba disponer de cuota era el propietario del segmento.

Inicialmente apareció `ORA-01950` porque `CARGA_HIST` no tenía cuota suficiente. Después, al asignarle una cuota de 2 MB y realizar una carga mayor, apareció `ORA-01536` por superar ese límite.

Ampliar el tamaño del tablespace no habría solucionado el problema, porque todavía existía espacio disponible. Lo que impedía continuar era la cuota individual del propietario.

**Evidencia:** `05-almacenamiento.script.log`, errores ORA-01950 y ORA-01536, cuota inicial de 2 MB.

### 13. Operación reanudable

La aplicación inició la carga aproximadamente a las 16:26:57 y terminó a las 16:27:11, por lo que transcurrieron unos 14 segundos entre ambos momentos.

Durante ese intervalo, Oracle suspendió la sentencia porque se había superado la cuota del tablespace. La sesión apareció con estado `SUSPENDED` y un tiempo máximo de espera configurado de 180 segundos.

El problema se resolvió ampliando la cuota de `CARGA_HIST` de 2 MB a 30 MB, sin tener que volver a lanzar la carga. Oracle registró en el alert log tanto la suspensión como la reanudación de la sentencia.

Finalmente, se insertaron 20.000 filas y el espacio consumido fue de 23 MB.

**Evidencia:** `05-resumable.script.log`, suspensión por ORA-01536, reanudación y 20.000 filas cargadas.

### 14. Diferencia entre DELETE y SHRINK SPACE

Antes del borrado, la tabla tenía 20.000 filas, 2.944 bloques y ocupaba 23 MB.

Después de eliminar el 90 % de los registros quedaron 2.000 filas, pero el segmento seguía utilizando los mismos 2.944 bloques y 23 MB.

Esto ocurre porque `DELETE` elimina las filas, pero no devuelve automáticamente al tablespace el espacio que ya estaba asignado al segmento.

Tras habilitar `ROW MOVEMENT` y ejecutar `SHRINK SPACE`, el segmento pasó a utilizar 304 bloques y 2,4 MB. Por tanto, se recuperaron aproximadamente 20,6 MB.

**Evidencia:** `05-hwm-shrink.script.log`, reducción de 2.944 a 304 bloques.

### 15. Perfil DEFAULT y PRF_PERSONAS

El perfil `DEFAULT` permitía 10 intentos de inicio de sesión fallidos, establecía una duración de contraseña de 180 días, un periodo de gracia de 7 días y un tiempo de bloqueo de 1 día. Además, `PASSWORD_VERIFY_FUNCTION` estaba configurado como `NULL`.

Para la cuenta nominal de secretaría se utilizó `PRF_PERSONAS`, que limitaba a 3 los intentos fallidos. Esto permite reducir los intentos repetidos de acceso con contraseñas incorrectas.

También se configuró un bloqueo de 1 minuto para las pruebas del laboratorio. Es un periodo corto que facilita comprobar el funcionamiento sin mantener la cuenta inutilizada durante demasiado tiempo.

La función de verificación de contraseña también rechazó una contraseña que no cumplía los requisitos de complejidad.

**Evidencia:** `06-gobierno-acceso.script.log`, DEFAULT con 10 intentos y PRF_PERSONAS con 3.

### 16. Permisos de ANA_GARCIA

Ana pudo consultar `ACA_MATRICULAS` aunque no se le hubiera concedido individualmente un permiso sobre esa tabla.

Esto fue posible porque tenía asignado `ROL_SECRETARIA_ACADEMIA`, que heredaba `ROL_LECTURA_ACADEMIA`. Este último disponía del privilegio `SELECT ANY TABLE` limitado al esquema `ADMIN_ACADEMIA`.

En cambio, sus permisos de modificación estaban más restringidos. Podía actualizar la columna `ESTADO` de `ACA_ESTUDIANTES`, pero no otras columnas.

Al intentar cambiar `NOMBRES`, Oracle devolvió `ORA-41900`, indicando que faltaba el privilegio `UPDATE` correspondiente.

**Evidencia:** `06-probar-acceso.script.log`, lectura de 60.000 matrículas y error ORA-41900.

## Partes H-I. Objetos, diagnóstico y guardia

### 17. Invalidación de objetos y dependencias

El ejercicio comenzó con una tabla de prueba, dos vistas y una función.

Al añadir una columna nueva que ninguno de esos objetos utilizaba, no se invalidó nada. Esto ocurre porque Oracle controla las dependencias con suficiente detalle como para distinguir qué columnas afectan realmente a cada objeto.

Al ampliar `EMAIL_INSTITUCIONAL`, se invalidó `V_CONTACTO_LAB6`, porque esa vista utilizaba la columna.

Después, al ampliar `NOMBRES`, también se invalidaron `V_ACTIVOS_LAB6` y `F_TOTAL_ACTIVOS_LAB6`, ya que la función dependía de esa vista. `V_CONTACTO_LAB6` seguía inválida desde el cambio anterior.

Cuando se llamó a la función, Oracle recompiló automáticamente los objetos necesarios. Posteriormente, el DBA recompiló el resto y la comprobación final mostró 0 objetos de negocio inválidos.

**Evidencia:** `07-objetos.script.log`.

### 18. Advisors de SGA y PGA

El asesor de SGA mostraba que con el tamaño de referencia de 1.536 MB el factor estimado de DB time era 1.

Al duplicar la SGA hasta 3.072 MB, ese factor seguía siendo 1, por lo que el asesor no predecía una mejora apreciable del rendimiento.

Respecto a la PGA, el objetivo configurado era de 512 MB. Con tamaños estimados de 128 MB y 256 MB aparecían 3 sobreasignaciones, mientras que con 384 MB o más el valor estimado era 0.

Por tanto, no tendría sentido aumentar la memoria sin una justificación basada en las métricas y los asesores.

**Evidencia:** `07-memoria.spool.log`, factor SGA = 1 y sobreasignaciones PGA = 0 a partir de 384 MB en los valores analizados.

### 19. Lecturas lógicas según la traza

La consulta de la aplicación contaba los estudiantes con estado `ACTIVO`.

Según el informe generado con `TKPROF`, necesitó 247 lecturas lógicas. Este valor aparecía en la columna `query`, correspondiente a la operación `Fetch`.

En la traza original también se podían identificar mediante la línea `FETCH`, utilizando el campo `cr=247`, que indica lecturas consistentes.

La consulta devolvió una fila con el resultado del recuento, sin lecturas físicas registradas en el informe de esa ejecución.

**Evidencia:** `07-adr-trazas.script.log`, 247 lecturas lógicas.

### 20. Incidentes de la guardia

**INC-601 — Bloqueo de una ficha.** La sesión 226 de `ANA_GARCIA` estaba esperando por la sesión 217 de `APP_ACADEMIA`, con el evento `enq: TX - row lock contention`. Tras comprobar el identificador de la sesión bloqueadora, se ejecutó `ALTER SYSTEM KILL SESSION '217,39233' IMMEDIATE`. Después dejaron de aparecer bloqueos activos y la fila 101 quedó en estado `ACTIVO`.

**INC-602 — Servicio de aplicación detenido.** La aplicación recibía `ORA-12514`. El diagnóstico mostró que `FREEPDB1` estaba arrancado, pero `academia_app` tenía `ARRANCADO = NO`. Se inició el servicio con `DBMS_SERVICE.START_SERVICE('academia_app')` y posteriormente ambos servicios aparecieron como arrancados.

**INC-603 — Cuenta de aplicación bloqueada.** La aplicación recibía `ORA-28000`. El diagnóstico mostró `APP_ACADEMIA` con estado `LOCKED`, lo que apuntaba a un bloqueo administrativo. Se ejecutó `ALTER USER APP_ACADEMIA ACCOUNT UNLOCK` y se verificó que la cuenta volvía a conectarse, consultar y actualizar los datos autorizados.

**Evidencias:** registros `08-guardia-incidente1.script.log`, `08-guardia-incidente2.script.log`, `08-guardia-incidente3.script.log`, salidas de `08-diagnostico.sql` y verificación final `09-verificacion-final.script.log`.
