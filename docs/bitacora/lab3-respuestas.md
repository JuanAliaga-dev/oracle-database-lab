# Laboratorio 3 - Respuestas de comprobacion

## Docker

### 1. ¿Que diferencia hay entre una imagen y un contenedor?
Una imagen es la plantilla a partir de la que se crean los contenedores. Por ejemplo, descargamos la imagen de Alpine y despues creamos contenedores a partir de ella. El contenedor es una instancia concreta de esa imagen, que puede estar ejecutandose, detenida o eliminada.

### 2. ¿Por que nota.txt desaparecio en G5 y en G6 no?
En G5 el archivo estaba dentro del sistema de archivos del propio contenedor, por lo que al eliminarlo se perdio. En G6 usamos un volumen de Docker, que existe independientemente del contenedor, y por eso los datos siguieron ahi aunque el contenedor cambiase.

### 3. Diferencia entre docker ps y docker ps -a. ¿Que significa Exited (0)?
`docker ps` muestra solo los contenedores que estan ejecutandose. `docker ps -a` tambien muestra los detenidos. `Exited (0)` significa que el proceso del contenedor termino correctamente y devolvio codigo de salida 0.

### 4. ¿Que significa -p 8181:8181?
El primer 8181 es el puerto de mi equipo y el segundo es el puerto del contenedor. Con `-p 80:8080`, accederia al puerto 80 de mi equipo y Docker redirigiria la conexion al puerto 8080 del contenedor.

### 5. ¿Por que Oracle sigue en marcha y hello-world termina?
Un contenedor sigue funcionando mientras su proceso principal siga vivo. Oracle mantiene el servidor de base de datos ejecutandose continuamente. `hello-world` solo imprime un mensaje y termina, por lo que su contenedor tambien termina.

### 6. ¿Que es el digest y por que lo registramos usando :latest?
El digest identifica exactamente el contenido de una imagen concreta. La etiqueta `latest` puede apuntar en el futuro a una version distinta, mientras que el digest permite saber exactamente que imagen se utilizo en la practica.

### 7. ¿Que comando borraria realmente los datos de Oracle?
Los datos estan en el volumen `oralab-26ai-data`, por lo que para borrarlos de verdad habria que eliminar ese volumen, por ejemplo con `docker volume rm oralab-26ai-data` cuando no este siendo usado. `docker rm oralab-26ai` solo elimina el contenedor y el volumen con los datos sigue existiendo.

## Git, organizacion y evidencia

### 8. ¿Por que hacemos el laboratorio dentro del repositorio con Issue, branch y Pull Request?
Porque asi todo el trabajo queda versionado y relacionado con una tarea concreta. La branch aisla los cambios, el Issue explica lo que se quiere hacer y el Pull Request permite revisar los scripts, las evidencias y el historial antes de integrarlo en main.

### 9. Diferencia entre source 00-config.sh y bash 00-config.sh
`bash 00-config.sh` lo ejecuta en otro proceso y las variables desaparecen al terminar. `source 00-config.sh` lo ejecuta en la shell actual, por lo que variables como CONT_NAME, EVID o SERVICE_PDB quedan disponibles para los comandos siguientes.

### 10. Explica 20260915T091230Z_02-docker.script.log
`20260915T091230Z` es la fecha y hora en UTC: 15 de septiembre de 2026 a las 09:12:30. `02` indica el numero de evidencia. `docker` describe lo que se esta verificando y `.script.log` indica que es un registro de terminal.

### 11. ¿Para que sirve .gitattributes?
Sirve para normalizar los finales de linea del repositorio. En esta practica fuerza LF para los scripts y evita problemas de CRLF procedentes de Windows, como el error `$'\r': command not found`.

### 12. ¿Por que usamos Create a merge commit y no Squash and merge?
Porque cada commit de esta practica representa una parte concreta y conserva informacion util sobre como se construyo y verifico el entorno. Con squash se perderia ese historial detallado al convertirlo todo en un unico commit.

## Seguridad

### 13. Las cuatro capas de la estrategia de contraseñas
Primero configuramos `.gitignore` para que Git ignore `config/.env`. Segundo mantenemos una plantilla `config/.env.example` sin secretos reales. Tercero guardamos las contraseñas reales solo en el archivo local `config/.env`. Cuarto cargamos los valores mediante variables de entorno en lugar de escribir las contraseñas directamente en los comandos. Si se omite la primera capa, se podria añadir por error el archivo real de secretos a Git.

### 14. ¿Por que no ponemos la contraseña directamente en docker run?
Porque los comandos escritos directamente en la terminal quedan guardados en `~/.bash_history` en texto plano. Al utilizar una variable como `$ORACLE_PWD`, el historial guarda el nombre de la variable y no la contraseña real.

### 15. ¿Que hacer si una contraseña aparece en un commit publicado?
No basta con borrarla en otro commit porque sigue existiendo en el historial anterior. Hay que considerar esa contraseña comprometida, cambiarla y limpiar el historial correspondiente. Si ya se publico, tambien hay que avisar para gestionar correctamente la exposicion.

## Oracle y herramientas

### 16. ¿Por que no usamos SPOOL ni @archivo.sql con sqlplus dentro del contenedor?
Porque el SQL*Plus del contenedor trabaja con su propio sistema de archivos y no directamente con el repositorio de Ubuntu. En su lugar enviamos el archivo SQL desde Ubuntu por la entrada estandar de `docker exec -i` y usamos `tee` para guardar la salida como evidencia en el repositorio.

### 17. ¿Que hace WHENEVER SQLERROR EXIT SQL.SQLCODE?
Hace que SQL*Plus termine con un codigo de error si alguna sentencia SQL falla. Asi el script de migraciones puede detectar el fallo y detenerse. Sin esa linea podria continuar despues de un error y dejar la base de datos en un estado incompleto.

### 18. ¿Que es una migracion y por que V000 y V001 no se editan despues?
Una migracion es un cambio versionado de la estructura o configuracion de la base de datos que se aplica en un orden concreto. Una vez aplicada no se debe modificar porque distintos entornos podrian haber ejecutado versiones diferentes del mismo archivo. Los cambios posteriores deben hacerse con una migracion nueva.

### 19. ¿Por que SQL Developer usa FREEPDB1?
Porque `FREEPDB1` es la base de datos pluggable donde estamos trabajando. `FREE` corresponde al contenedor raiz y no es donde hemos creado nuestros esquemas. Usar el nombre de servicio `FREEPDB1` permite conectarnos directamente a la PDB correcta.

### 20. ¿Que aporta SQLcl frente a SQL*Plus?
SQLcl es una herramienta mas moderna, con mejor formato de resultados, historial, autocompletado, conexiones guardadas e integracion con herramientas como Liquibase. SQL*Plus es mas basico pero esta disponible practicamente en cualquier instalacion Oracle, por lo que un DBA debe conocer los dos.

## Entorno de trabajo

### 21. ¿Por que pasamos de Git Bash a Ubuntu en WSL 2?
Porque WSL 2 proporciona un entorno Linux real, mucho mas parecido a los servidores de produccion. Git Bash puede transformar rutas Linux en rutas de Windows, necesita soluciones como `winpty` para algunos contenedores interactivos y no incluye muchas herramientas de administracion como `free`, `ss` o `htop`. En Ubuntu esos problemas desaparecen.

### 22. ¿Por que clonamos en ~/oracle-database-lab y usamos bash?
Trabajar directamente en `/mnt/c` cruza continuamente entre el sistema de archivos de Windows y Linux, lo que hace Git y Docker mas lentos y puede producir problemas de permisos y finales de linea. Por eso trabajamos dentro de `/home/juan`. Usamos bash porque es la shell habitual en servidores Linux y los scripts se comportan de la misma forma en los distintos entornos. zsh es util de forma interactiva, pero tiene algunas diferencias de sintaxis y no siempre esta instalado en los servidores.
