He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 6 · E002

## Entrada 1

- **Herramienta:** ChatGPT
- **Prompt literal:** «a ver, necesito que me vayas diciendo que voy haciendo para ir haciendo esta practica entera.»
- **Código recibido:** 
docker ps -a
bash scripts/lab5/16-verificacion-final.sh | tail -5
rsync -avh --ignore-existing "$KIT/" ./
source scripts/deployment/env/00-config.sh
mkdir -p "$EVID_LAB6"/{script,spool,config}
chmod +x scripts/lab6/*.sh scripts/ops/*.sh
bash scripts/ops/alumno.sh

- **Dónde lo usé:** Parte A, preparación del entorno.
- **Propósito:** Recibir orientación para configurar el laboratorio y comprobar los resultados.
- **Qué hace:** Comprueba el estado del contenedor, incorpora los archivos nuevos y prepara las evidencias identificadas con mi código E002.
- **Evidencia de ejecución:** Salidas de terminal de la Parte A; pendiente de registrar la evidencia 00.
