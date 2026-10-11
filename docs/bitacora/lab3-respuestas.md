# Lab 3 — Respuestas de autoevaluación

## Docker

**1. ¿Qué diferencia hay entre una imagen y un contenedor? (ejemplos G2 y G4)**
Una imagen es la plantilla (de solo lectura) y un contenedor es una instancia en marcha creada a partir de ella. En G2, `docker run hello-world` creó un contenedor a partir de la imagen hello-world; en G4 hice `docker run -it --name prueba alpine:3.20 sh` y me metí dentro del contenedor, que era una instancia de la imagen alpine. Vamos, la imagen sería el molde y el contenedor el objeto.

**2. En G5 el archivo nota.txt desapareció y en G6 no. ¿Por qué?**
Porque en G5 lo escribí en `/tmp`, que forma parte del sistema de archivos del propio contenedor; al hacer `docker rm prueba` se borró el contenedor y con él el archivo. En G6 usé un volumen con nombre (`datos-prueba`) montado en `/datos`, así que aunque los contenedores se borraban (con `--rm`), el volumen seguía existiendo y el segundo contenedor encontró el dato.

**3. Diferencia entre docker ps y docker ps -a. ¿Qué significa Exited (0)?**
`docker ps` solo lista los contenedores que están en marcha; `docker ps -a` los lista todos, incluidos los detenidos. `Exited (0)` significa que el contenedor terminó y lo hizo correctamente (código de salida 0). Si fuese `Exited (1)` o mas, habría terminado con error.

**4. En -p 8181:8181, ¿qué número es de mi equipo y cuál del contenedor? ¿Y -p 80:8080 en nginx?**
El primero (izquierda) es el puerto de mi equipo y el segundo el del contenedor. Con `-p 80:8080` sería "mi puerto 80 → puerto 8080 del contenedor", pero nginx escucha en el 80 de dentro, no en el 8080, así que la página no cargaría.

**5. ¿Por qué Oracle se queda en marcha y hello-world termina solo?**
Porque un contenedor vive exactamente lo que vive su proceso principal. En hello-world el proceso imprime el mensaje y acaba, y el contenedor termina. Oracle arranca el motor de base de datos y ese proceso no termina, así que el contenedor sigue `Up`.

**6. ¿Qué es el digest y por qué lo registramos si usamos :latest?**
El digest es la huella exacta (sha256) de la imagen. `:latest` cambia con el tiempo, así que por sí solo no me dice qué versión concreta tengo. Registrando el digest (en mi caso `sha256:f988b0c04c4c…`) cualquiera sabe exactamente qué imagen instalé.

**7. ¿Qué comando borraría los datos de Oracle? ¿Por qué docker rm oralab-26ai no?**
Los borraría `docker volume rm oralab-26ai-data`, porque ahí viven los datos. `docker rm oralab-26ai` borra el contenedor, pero los datos están en el volumen (montado en `/opt/oracle/oradata`), no dentro del contenedor, por eso sobreviven.

## Git, organización y evidencia

**8. ¿Por qué se hace dentro del repo con Issue, branch y PR?**
Porque así el trabajo queda trazable y reproducible: el Issue dice qué hay que hacer, la branch aísla los cambios y el PR permite revisarlos. Si lo hiciera en una carpeta suelta, nadie podría reconstruir el entorno ni ver cómo se hizo. Es el flujo que se sigue en la industria.

**9. ¿Qué diferencia hay entre source 00-config.sh y bash 00-config.sh?**
`bash 00-config.sh` lo ejecutaría en un proceso nuevo y las variables se perderían al terminar; con `source 00-config.sh` se ejecuta en mi shell actual y las constantes (`$CONT_NAME`, `$EVID`, la función `ts`, etc.) quedan disponibles. Por eso usamos source.

**10. Explica cada parte de 20260915T091230Z_02-docker.script.log.**
`20260915T091230Z` es la fecha y hora en UTC (la Z final indica UTC); `02` es el número de paso; `docker` describe qué se hizo; y `.script.log` indica que es una grabación hecha con `script`. Así las evidencias se ordenan solas y se ve cuándo se hizo cada paso.

**11. ¿Para qué sirve .gitattributes y qué error evita?**
Para normalizar los finales de línea (`eol=lf`) en `.sh`, `.sql` y `.md`. Evita el error `$'\r': command not found`, que aparece cuando un script trae finales de línea de Windows, y también que los diffs salgan llenos de ruido.

**12. ¿Por qué elegimos Create a merge commit en lugar de Squash and merge?**
Porque cada commit del PR corresponde a una Parte y tiene valor por sí mismo; con merge commit se conserva ese historial paso a paso, que muestra cómo y cuándo se verificó cada cosa. Con squash quedaría todo en un único commit y se perdería.

## Seguridad

**13. Las cuatro capas de la estrategia de contraseñas. ¿Qué pasa si te saltas la primera?**
Capa 1: decirle a Git que ignore (`config/.env` y `backups/` en `.gitignore`). Capa 2: la plantilla `.env.example` versionada, sin valores reales. Capa 3: el archivo real `config/.env`, local y comprobado con `git check-ignore`. Capa 4: leer la contraseña desde la variable (`$ORACLE_PWD`) sin teclearla a mano. Si me salto la primera, el `.env` con la contraseña real podría acabar en un commit… y ya no se puede borrar del historial.

**14. ¿Por qué no escribimos la contraseña directamente en docker run?**
Porque todo lo que tecleo queda en el historial de la terminal (`~/.bash_history`) en texto plano. Usando `"$ORACLE_PWD"`, en el historial solo queda el nombre de la variable.

**15. Si la contraseña aparece en un commit publicado, ¿basta con borrarla en un commit nuevo?**
No. Aunque la borre en un commit nuevo, sigue en el historial y cualquiera puede recuperarla yendo hacia atrás. Lo correcto es rotarla (cambiar la contraseña) y avisar al docente: el secreto se da por comprometido.

## Oracle y herramientas

**16. ¿Por qué no usamos SPOOL ni @archivo.sql con sqlplus dentro del contenedor?**
Porque sqlplus corre dentro del contenedor: un SPOOL escribiría el archivo dentro del contenedor (no en mi repo) y `@archivo.sql` buscaría el .sql dentro del contenedor, donde no está (SP2-0310). En su lugar pasé el archivo por la entrada estándar (`< archivo.sql`) y capturé la salida con `tee` desde mi equipo.

**17. ¿Qué hace WHENEVER SQLERROR EXIT SQL.SQLCODE? ¿Y sin esa línea?**
Hace que, si una sentencia falla, el script se detenga en ese punto con el código de error. Sin esa línea, sqlplus seguiría ejecutando las siguientes sentencias sobre un estado a medias.

**18. ¿Qué es una migración y por qué no se editan una vez aplicadas?**
Una migración es un script SQL numerado que lleva la base de un estado al siguiente (V000 crea tablespaces y usuarios; V001, las tablas). No se editan tras aplicarlas porque ya están aplicadas en la base; si hay que corregir algo se crea una migración nueva, así todos reconstruyen lo mismo en el mismo orden.

**19. ¿Por qué en SQL Developer se usa FREEPDB1 y no FREE ni un SID?**
Porque FREEPDB1 es la base de trabajo (el PDB) donde están los esquemas. Con el SID o con el servicio FREE me conectaría al contenedor raíz (CDB), que no es donde quiero trabajar.

**20. ¿Qué aporta SQLcl frente a SQL*Plus? ¿Por qué dominar ambas?**
SQLcl es la versión moderna: autocompletado, historial, formato automático, conexiones guardadas e integración con Liquibase. SQL*Plus es más antiguo, pero está en cualquier servidor Oracle desde hace décadas. Conviene dominar las dos porque en un servidor de producción puede que solo tengas sqlplus, mientras que en el día a día usas SQLcl.

## Entorno de trabajo

**21. ¿Por qué pasamos de Git Bash a Ubuntu en WSL 2? Dos problemas concretos.**
Porque Git Bash es una emulación y se nota. Por ejemplo, convertía rutas `/opt/...` a rutas de Windows y rompía argumentos de Docker, y `docker run -it` fallaba con "not a TTY" (había que tirar de winpty o PowerShell). Además no trae utilidades de administración como `free`, `ss` o `htop`. En WSL 2 es Linux de verdad y eso desaparece.

**22. ¿Por qué clonamos en ~/oracle-database-lab y no en /mnt/c? ¿Y por qué bash frente a zsh?**
Porque trabajar sobre `/mnt/c` (el disco de Windows) va lento y da problemas de permisos y de finales de línea; en `/home` (dentro de Linux) todo va como en un servidor. Y bash porque es el shell estándar de los servidores y el que usan los scripts del curso; zsh es cómodo para uso personal, pero no es el estándar para scripts de administración.
