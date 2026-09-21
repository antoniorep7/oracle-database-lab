\# Respuestas del Laboratorio 1



\## 1. ¿Cuál es la diferencia entre Working Directory, Staging Area y Local Repository? Da un ejemplo de un

archivo pasando por las tres.



El Working Directory es la carpeta donde se editan los archivos. La Staging Area tiene los cambios preparados para el próximo commit. El Local Repository contiene el historial guardado en git.



\## 2. Si modificas un archivo pero no haces git add, ¿aparece ese cambio en tu próximo commit? Explica por qué.



No aparecen en el commit porque el git commit solo confirma los cambios que están en la Staging Area.



\## 3. ¿Por qué git status no mostraba las carpetas vacías que creaste en la Parte C? ¿Qué truco usamos para solucionarlo?



Porque git no versiona carpetas directamente, solo archivos. Para conservar una carpeta vacía usamos un archivo gitkeep.



\## 4. Explica con tus palabras qué es HEAD



HEAD es el puntero que indica la rama y el commit en ese momento.



\## 5. ¿Qué diferencia hay entre crear una branch con git switch -c y crear una carpeta nueva con mkdir? ¿Cómo lo comprobamos en la Parte G?

Porque Git switch -c crea una nueva línea de evolución en el historial mientras que mkdir una carpeta física. Lo comprobamos usando `ls -la`.



\## 6. Durante el conflicto de la Parte H, ¿qué representaba el contenido entre <<<<<<< HEAD y =======? ¿Y entre ======= y >>>>>>>?



Entre <<<<<<< HEAD y ======= estaba la versión de la rama actual. Entre ======= y >>>>>>> estaba la versión de la rama que se fusionaba.



\## 7. ¿Por qué NO se debe hacer git commit --amend sobre un commit que ya se subió con git push?



No se debe usar sobre commits subidos porque modifica el historial y cambia el hash del commit.



\## 8. Si borras por accidente la carpeta .git de tu proyecto, ¿qué se pierde exactamente? ¿Se pierde también el código fuente que está en el disco?



Se perdería el historial (creo), los commits, las ramas y la configuración de Git, pero no los archivos del proyecto.



\## 9. Explica con tus propias palabras la diferencia entre Git y GitHub, sin usar la palabra "nube".



Git es el programa local que controla versiones. GitHub es el servicio web que contiene los repositorios Git y facilita la colaboración.



\## 10. ¿Por qué no se debe subir un archivo .env con contraseñas reales a un repositorio, aunque el repositorio sea privado?



No se deberia subir un .env con contraseñas porque el contenido podría quedar expuesto incluso dentro del historial.



\## 11. Un compañero te dice: "hice push y ahora GitHub me rechaza el segundo push con 'non-fast-forward'". ¿Qué ha ocurrido probablemente y qué comando ejecutarías primero?

Significa que el remoto tiene commits que no tengo en el local. Ejecutaría primero un git pull y después git push.



\## 12. ¿Qué tipo de Conventional Commit (feat, fix, docs, test…) usarías para: añadir un índice de rendimiento a una tabla, corregir una restricción mal definida, y actualizar el README?



Para añadir un índice usaría perf(db): add performance index



Para corregir una restricción usaría fix(db): correct constraint



Para actualizar el README usaría docs: update README

