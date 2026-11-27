He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 7 · E002

## Entrada 1
- **Herramienta:** ChatGPT.
- **Prompt (literal, copiado tal cual):**
```text
tengo que hacer esta tarea de lab 7 me puedes ayudar con ella
```
- **Código recibido:** no aplica para la orientación inicial.
- **Dónde lo usé:** Parte A.
- **Propósito:** Comprender la organización del laboratorio.
- **Qué hace:** Me orientó sobre la preparación del entorno y las fases de la práctica.
- **Evidencia de ejecución:** no aplica.

## Entrada 2
- **Herramienta:** ChatGPT.
- **Prompt:** Consulta sobre los resultados del INC-605. Adjunté la salida de la terminal.
- **Código recibido:**
```sql
EXEC DBMS_STATS.UNLOCK_TABLE_STATS('T6', 'CESTA');
EXEC DBMS_STATS.GATHER_TABLE_STATS('T6', 'CESTA', no_invalidate => FALSE);
```
- **Dónde lo usé:** Parte G, INC-605.
- **Propósito:** Resolver el problema de las estadísticas bloqueadas.
- **Qué hace:** Desbloquea las estadísticas de CESTA y actualiza la información utilizada por el optimizador.
- **Evidencia de ejecución:** docs/bitacora/lab7/evidencia/incidentes/20261126T173214Z_INC-605_verificar.txt.

## Entrada 3
- **Herramienta:** ChatGPT.
- **Prompt:** Consulta sobre los resultados del INC-612. Adjunté la salida de la terminal.
- **Código recibido:**
```sql
CREATE INDEX T6.IX_T6_CLI_EMAIL_UPPER
ON T6.CLIENTE (UPPER(email))
TABLESPACE TBS_T6;
```
- **Dónde lo usé:** Parte G, INC-612.
- **Propósito:** Recuperar el acceso eficiente por correo electrónico.
- **Qué hace:** Crea un índice basado en función para las búsquedas que utilizan UPPER(email).
- **Evidencia de ejecución:** docs/bitacora/lab7/evidencia/incidentes/20261126T173922Z_INC-612_verificar.txt.
