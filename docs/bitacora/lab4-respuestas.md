# Lab 4 — Oracle DBA Workflow I

## Parte A — Diagnóstico inicial

1. **Riesgo medio:** cinco datafiles de negocio utilizan rutas no estándar bajo `$ORACLE_HOME/dbs`, aunque sus destinos reales están dentro del volumen persistente.
2. **Riesgo medio:** las migraciones originales contienen contraseñas de esquema; se corregirán mediante V003, sin modificar el historial ya aplicado.
3. **Riesgo medio:** el procedimiento de migraciones del Lab 3 utiliza credenciales de SYS en argumentos; se sustituirá por autenticación local segura.
4. **Riesgo bajo:** el puerto 8181 está publicado, pero debe comprobarse si existe un servicio que lo utilice.
5. **Prioridad:** asegurar credenciales y autenticación, normalizar las rutas de los datafiles y revisar los puertos y scripts de despliegue.

## Parte B — Observabilidad

## Parte C — Persistencia

## Parte D — Migraciones

## Parte E — Instancia

## Parte F — Parámetros

## Parte G — Memoria

## Parte H — Procesos

## Parte I — Contenedores

## Parte J — Archivos

## Parte K — Bloques

## Parte L — Capacidad

## Parte M — Filas migradas

## Parte N — Verificación final
