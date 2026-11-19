# Laboratorio 6 — Guardia operativa

**Alumno:** E002
**Fecha:** 19/11/2026
**Base de datos:** Oracle FREE, PDB FREEPDB1

## INC-601 — Bloqueo al guardar una ficha de estudiante

**Detección:** A las 16:46 UTC, secretaría comunicó que la aplicación se quedaba esperando al guardar la ficha del estudiante 101.

**Impacto:** La operación de actualización de esa ficha quedó bloqueada mientras otra sesión mantenía abierta una transacción. La espera se observó durante el diagnóstico, pero no se dispone de una medición exacta de la duración total percibida por el usuario.

**Diagnóstico:** Utilicé `08-diagnostico.sql` para consultar las sesiones y bloqueos. La sesión 226, correspondiente a `ANA_GARCIA`, estaba esperando con el evento `enq: TX - row lock contention`. La sesión bloqueadora era la 217, de `APP_ACADEMIA`, asociada al módulo `matriculas`. La transacción bloqueadora tenía 1 bloque y 1 registro de undo utilizados.

**Causa raíz:** Una transacción de la aplicación mantenía un bloqueo sobre una fila de `ACA_ESTUDIANTES` mientras otra sesión intentaba modificarla. El problema era de concurrencia entre transacciones, no una caída de Oracle.

**Intervención:** Primero comprobé que la sesión bloqueadora seguía teniendo `SID = 217` y `SERIAL# = 39233`. Después ejecuté:

`ALTER SYSTEM KILL SESSION '217,39233' IMMEDIATE;`

Se finalizó únicamente la sesión responsable del bloqueo, asumiendo que su transacción sin confirmar sería revertida.

**Validación:** Repetí `08-diagnostico.sql` y comprobé que ya no aparecían sesiones bloqueadas ni bloqueos TX pendientes. ASH conservaba los registros históricos de la espera. La verificación final también confirmó que la fila 101 tenía el estado `ACTIVO`.

**Acción preventiva:** Revisar la gestión de transacciones de la aplicación para evitar mantener bloqueos durante esperas innecesarias. También sería conveniente monitorizar las sesiones bloqueadoras y registrar `MODULE`, `ACTION` y `CLIENT_IDENTIFIER` para localizar más rápidamente el origen de futuros problemas.

## INC-602 — La aplicación no podía conectarse

**Detección:** A las 16:49 UTC, el equipo de matrículas notificó que las instancias de la aplicación no podían conectarse y recibían el error `ORA-12514`.

**Impacto:** Las conexiones que utilizaban el servicio `academia_app` dejaron de funcionar mientras dicho servicio estaba detenido. La PDB seguía disponible por su servicio habitual.

**Diagnóstico:** Consulté la sección de servicios de `08-diagnostico.sql`. El resultado mostraba `FREEPDB1` con `ARRANCADO = SI`, pero `academia_app` con `ARRANCADO = NO`.

**Causa raíz:** El servicio de aplicación estaba definido, pero no se encontraba arrancado. Por eso el listener no podía atender las conexiones dirigidas a ese servicio. No era necesario reiniciar toda la base de datos.

**Intervención:** Inicié el servicio desde `FREEPDB1` mediante:

`BEGIN DBMS_SERVICE.START_SERVICE('academia_app'); END;`

Esta intervención era más limitada que reiniciar la instancia o la PDB, ya que únicamente actuaba sobre el servicio afectado.

**Validación:** Repetí el diagnóstico y ambos servicios aparecieron con `ARRANCADO = SI`. La comprobación confirmó que `academia_app` volvía a estar activo.

**Acción preventiva:** Comprobar el estado de los servicios después de reinicios y despliegues, y mantener guardada la configuración de arranque correspondiente. También sería útil configurar una comprobación automática de conectividad mediante el servicio utilizado realmente por la aplicación.

## INC-603 — Cuenta de aplicación bloqueada

**Detección:** A las 16:50 UTC, el equipo de matrículas informó de que la aplicación devolvía `ORA-28000` tras un despliegue. La nota del turno anterior indicaba que se había ejecutado el script trimestral de bajas.

**Impacto:** La aplicación no podía autenticarse con la cuenta `APP_ACADEMIA`, por lo que las operaciones que dependían de ella quedaban inaccesibles.

**Diagnóstico:** Consulté `08-diagnostico.sql` y encontré `APP_ACADEMIA` con estado `LOCKED`, perfil `DEFAULT` y 10 intentos fallidos permitidos. La diferencia entre `LOCKED` y `LOCKED(TIMED)` era importante: el primero era compatible con un bloqueo administrativo, mientras que el segundo indicaría un bloqueo temporal por intentos fallidos.

**Causa raíz:** La evidencia apuntaba a que el procedimiento de bajas había incluido una cuenta de servicio que debía seguir activa. El diagnóstico confirmó el bloqueo administrativo, aunque no se examinó el código del script de bajas para demostrar exactamente cómo seleccionó esa cuenta.

**Intervención:** Después de identificar que `APP_ACADEMIA` era la cuenta legítima de la aplicación, ejecuté:

`ALTER USER APP_ACADEMIA ACCOUNT UNLOCK;`

No fue necesario cambiar la contraseña ni modificar sus roles o privilegios.

**Validación:** Repetí el diagnóstico y la cuenta ya no aparecía entre los usuarios bloqueados. Además, ejecuté `06-app-sigue.sql` con la cuenta de aplicación: pudo conectarse, consultar 17.600 estudiantes activos y realizar una actualización que posteriormente se revirtió. El intento de insertar en el histórico siguió rechazándose con `ORA-00942`, como estaba previsto por los permisos configurados.

**Acción preventiva:** Revisar el script de bajas para que distinga entre cuentas nominales de personas y cuentas técnicas de aplicaciones. La selección de usuarios que se van a bloquear debería verificarse antes de ejecutar el cambio, especialmente en cuentas utilizadas por servicios de producción.

## Comprobación después de la guardia

Una vez resueltos los tres incidentes, la verificación final del Laboratorio 6 confirmó que el servicio `academia_app` estaba arrancado, la cuenta `APP_ACADEMIA` permanecía abierta, no existían sesiones bloqueadas y la fila 101 conservaba el estado `ACTIVO`.

Las tres intervenciones se realizaron sobre el recurso concreto que provocaba cada incidencia, sin necesidad de reiniciar toda la instancia Oracle.
