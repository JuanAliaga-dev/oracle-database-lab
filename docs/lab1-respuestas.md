\# Laboratorio 1 - Preguntas de comprobación



Nombre: Juan Aliaga Navarro



\## 1. ¿Cuál es la diferencia entre Working Directory, Staging Area y Local Repository? Da un ejemplo de un archivo pasando por las tres.



El Working Directory es donde están los archivos que estoy editando actualmente en mi ordenador. La Staging Area es una zona intermedia donde selecciono qué cambios quiero incluir en el siguiente commit. El Local Repository es donde Git guarda definitivamente el historial de commits.



\## 2. Si modificas un archivo pero no haces git add, ¿aparece ese cambio en tu próximo commit? Explica por qué.



No. Un commit solo incluye los cambios que se encuentran en la Staging Area. Si modifico un archivo pero no ejecuto git add, el cambio sigue únicamente en el Working Directory y no se incluye en el siguiente commit.



\## 3. ¿Por qué git status no mostraba las carpetas vacías que creaste en la Parte C? ¿Qué truco usamos para solucionarlo?



Porque Git no versiona carpetas, sino archivos. Si una carpeta está completamente vacía, Git no tiene ningún archivo que registrar dentro de ella.



Para conservar esas carpetas añadimos archivos .gitkeep, que sirven como archivos placeholder para que Git pueda incluirlas en el repositorio.



\## 4. Explica con tus palabras qué es HEAD.



HEAD es el puntero que indica en qué posición del historial estoy trabajando actualmente. Normalmente apunta a la rama activa y, a través de ella, al último commit de esa rama.



\## 5. ¿Qué diferencia hay entre crear una branch con git switch -c y crear una carpeta nueva con mkdir? ¿Cómo lo comprobamos en la Parte G?



"git switch -c" crea una nueva rama en el historial de Git, mientras que "mkdir" crea físicamente una carpeta en el disco.



Una branch no es una carpeta ni duplica los archivos. Lo comprobamos creando "feature/customer-search" y ejecutando "ls -la": la estructura de carpetas seguía siendo la misma y no apareció ninguna carpeta llamada "feature".



También vimos que un archivo podía aparecer o desaparecer al cambiar de rama dependiendo de si su commit existía en esa rama.



\## 6. Durante el conflicto de la Parte H, ¿qué representaba el contenido entre <<<<<<< HEAD y =======? ¿Y entre ======= y >>>>>>>?



El contenido entre "<<<<<<< HEAD" y "=======" representaba la versión que ya existía en la rama actual, que en nuestro caso era main.



El contenido entre "=======" y ">>>>>>> fix/readme-subtitle" representaba la versión procedente de la rama que intentábamos fusionar.



Git mostraba ambas versiones porque no podía decidir automáticamente cuál debía conservar.



\## 7. ¿Por qué NO se debe hacer git commit --amend sobre un commit que ya se subió con git push?



Porque "git commit --amend" reescribe el último commit y genera un hash diferente. Si ese commit ya se ha compartido en GitHub, otros usuarios pueden tener la versión anterior y se producirían historiales diferentes y posibles conflictos.



Por eso "--amend" es adecuado para commits todavía locales, pero no es recomendable modificar de esta forma commits que ya se han compartido.



\## 8. Si borras por accidente la carpeta .git de tu proyecto, ¿qué se pierde exactamente? ¿Se pierde también el código fuente que está en el disco?



Se perdería la información interna del repositorio Git: commits, ramas, referencias, configuración local e historial.



Los archivos del proyecto que están físicamente en el Working Directory no se borrarían. Seguiría teniendo el código y los documentos, pero esa carpeta dejaría de funcionar como el repositorio Git que era anteriormente.



\## 9. Explica con tus propias palabras la diferencia entre Git y GitHub, sin usar la palabra "nube".



Git es un sistema de control de versiones que funciona en mi ordenador y permite crear commits, ramas, merges y consultar el historial incluso sin conexión a Internet.



GitHub es una plataforma que aloja repositorios Git en un servidor y permite compartirlos y trabajar con otras personas, además de ofrecer herramientas como Pull Requests, Issues y Code Review.



\## 10. ¿Por qué no se debe subir un archivo .env con contraseñas reales a un repositorio, aunque el repositorio sea privado?



Porque una contraseña o una clave subida al repositorio puede quedar almacenada en el historial de Git incluso aunque después borremos el archivo.



Además, un repositorio privado puede ser compartido accidentalmente, cambiar de permisos o ser accesible por otras personas. Las credenciales reales deben mantenerse fuera del repositorio y almacenarse de forma segura.



\## 11. Un compañero te dice: "hice push y ahora GitHub me rechaza el segundo push con non-fast-forward". ¿Qué ha ocurrido probablemente y qué comando ejecutarías primero?



Probablemente el repositorio remoto contiene commits que todavía no existen en la copia local del compañero. Esto puede ocurrir porque otra persona haya subido cambios o porque se haya modificado un archivo directamente desde GitHub.



Lo primero que ejecutaría sería:



"git pull"



De esta forma traería los cambios remotos y los integraría con los locales. Si aparece algún conflicto tendría que resolverlo antes de volver a ejecutar "git push".



\## 12. ¿Qué tipo de Conventional Commit usarías para: añadir un índice de rendimiento a una tabla, corregir una restricción mal definida y actualizar el README?



Para añadir un índice de rendimiento utilizaría "perf", porque el objetivo del cambio es mejorar el rendimiento.



Para corregir una restricción mal definida utilizaría "fix", porque estoy corrigiendo un error.



Para actualizar el README utilizaría "docs", porque se trata de un cambio únicamente de documentación.

