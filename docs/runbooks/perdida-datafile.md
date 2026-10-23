# Runbook: pérdida de un datafile Oracle

## Síntoma
Errores ORA-01116, ORA-01110, ORA-27041 y posiblemente ORA-63999.

## Diagnóstico
Comprobar ARCHIVELOG, el estado de FREEPDB1, el datafile afectado y las copias RMAN.

## Recuperación
1. Identificar el datafile afectado.
2. Si la PDB sigue abierta, poner el datafile OFFLINE.
3. Ejecutar RESTORE DATAFILE y RECOVER DATAFILE con RMAN.
4. Devolver el datafile ONLINE o abrir FREEPDB1 si estaba MOUNTED.
5. Verificar la recuperación de los datos.

## Evidencia del Lab 5
- Datafile afectado: 16 (academia01.dbf).
- Recuperación mediante RMAN y redo.
- RTO: 8 segundos.
- RPO: 0.
- FREEPDB1 recuperada en READ WRITE.
- Fila 104 restaurada a ACTIVO tras verificar la recuperación.

## Escalado
Informar al DBA responsable, al equipo de infraestructura y al responsable del servicio afectado.

No eliminar archivos manualmente durante la recuperación. Confirmar la existencia de backups y redo antes de proceder.
