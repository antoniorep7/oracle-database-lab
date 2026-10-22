# Lab 4 — Respuestas

Las evidencias que cito están en `docs/bitacora/lab4/evidencia/`.

## Respaldo

**1. ¿Por qué la copia de la Parte B es consistente? ¿Qué pasaría si se tomara con la base abierta?**
Porque el script para la base con SHUTDOWN IMMEDIATE antes de copiar (en el log 00 se ve "Database closed" y "ORACLE instance shut down"). Al cerrar limpio todos los datafiles se quedan con el mismo SCN, así que al restaurar no hay que aplicar redo. Si se copiara con la base abierta, cada archivo quedaría en un momento distinto y la copia sería inconsistente: solo se podría recuperar con ARCHIVELOG y RMAN.

**2. ¿Para qué sirven manifiesto.txt y SHA256SUMS? ¿Qué comprueba restaurar-estado.sh antes de tocar nada?**
El manifiesto dice qué copia es: etiqueta lab3-final, fecha, imagen, commit abf6df1 y que es en frío. El SHA256SUMS guarda la huella de cada archivo para saber si algo se ha corrompido; en la B.3 los cuatro me dieron OK. restaurar-estado.sh lo primero que hace es comprobar esas sumas, y si no cuadran se para sin tocar nada. Además te pide escribir el nombre del volumen antes de borrarlo.

**3. Define RPO y RTO con los datos de tu propia copia.**
El RPO es lo que perdería si restaurase: todo lo hecho después de la copia del 22/10 a las 16:27 UTC, o sea, las Partes C a H (aunque el código está en Git). El RTO es lo que tardo en volver a tener la base funcionando. La copia me tardó unos 2 minutos y ocupa 512 MB. La restauración no la he cronometrado, pero como hace lo mismo al revés calculo unos 2 o 3 minutos; la mediré en el Lab 5.

## Diagnóstico y correcciones

**4. ¿Dónde creó Oracle los datafiles de V000 y por qué? ¿Qué parámetro habría evitado el problema?**
En $ORACLE_HOME/dbs (en el log 01 salen los file# 16 a 20 ahí). Pasó porque V000 ponía solo el nombre del archivo, sin ruta, y no había OMF (db_create_file_dest sale vacío). En mi caso dbs es un enlace al volumen, por eso en el backup me salió todo "dentro", pero aun así estaban en una carpeta que no toca. Se habría evitado con DB_CREATE_FILE_DEST o poniendo la ruta completa en FREE/FREEPDB1.

**5. ¿Por qué ALTER TABLESPACE ... RENAME DATAFILE no basta para mover un archivo?**
Porque el RENAME no mueve el archivo, solo cambia la ruta que tiene apuntada el control file. Por eso el script pone el tablespace OFFLINE, copia el archivo, hace el RENAME y lo vuelve a poner ONLINE. Solo cuando el ONLINE va bien borra el original.

**6. ¿Por qué no se edita V000 y se crea V003? ¿Por qué invalidar una contraseña es mejor que borrarla del archivo?**
V000 ya se aplicó, y una migración aplicada no se toca: si algo estaba mal se corrige con una nueva. Borrar la contraseña del archivo no sirve porque sigue en el historial de Git. Lo que de verdad protege es que deje de funcionar, y eso hace V003 quitando la contraseña a las cuentas ADMIN_*. Lo probé con la contraseña de V000 y me dio ORA-01017.

**7. ¿Qué demuestra la prueba de persistencia y por qué se ejecuta después de reubicar?**
Que los datos están en el volumen y no en el contenedor: borré el contenedor, lo volví a crear y seguían los 5 tablespaces y todas las tablas. También se ve que ya solo se publica el 1521. Se hace después de reubicar porque si algún datafile hubiera estado dentro del contenedor, el docker rm lo habría borrado.

## Instancia y arranque

**8. Con tus evidencias: tres datos que pertenecen a la instancia y tres que pertenecen a la base de datos.**
De la instancia: el HOST_NAME (6238ae479081, que cambió al recrear el contenedor), la hora de arranque (22/10 a las 16:46) y la SGA de unos 1,6 GB. De la base: el DBID (1509902213, que no cambió aunque recreé el contenedor), la fecha de creación (05/07/2026) y los archivos (16 datafiles, 3 redo y 2 control files). Todo sale del spool 05.

**9. ¿Qué lee Oracle en NOMOUNT, en MOUNT y en OPEN? Relaciónalo con ORA-01507 y ORA-01219.**
En NOMOUNT lee el SPFILE y arranca memoria y procesos, por eso v$parameter funciona pero v$database da ORA-01507: todavía no ha abierto el control file. En MOUNT abre los control files y ya ve los datafiles, pero dba_tables da ORA-01219 porque el diccionario está en SYSTEM y ese datafile sigue cerrado. En OPEN ya abre datafiles y redo y la base queda en READ WRITE. Lo vi todo en el log 06.

**10. ¿Por qué FREEPDB1 se abre sola tras ALTER DATABASE OPEN? ¿Qué harías si no lo hiciera?**
Porque tiene el estado guardado: en el log 06 sale en dba_pdb_saved_states como OPEN. Si no lo tuviera se quedaría en MOUNTED y las aplicaciones no podrían conectarse. La abriría con ALTER PLUGGABLE DATABASE FREEPDB1 OPEN y luego haría SAVE STATE para que no volviera a pasar.

**11. Clasifica ORA-00800 y ORA-20013 del alert log: ¿alerta real o ruido conocido? Justifícalo.**
Son ruido conocido. El ORA-00800 me sale en cada arranque porque Oracle intenta subir la prioridad de VKTM y LGWR y el contenedor no le deja, pero justo después la base abre sin problema. El ORA-20013 a mí no me salió; es un aviso de que no puede leer los parches dentro de la imagen y tampoco afecta. Lo que sí me salieron fueron unos ORA-27300 de un arranque anterior, y esos los he anotado aparte porque no están en la lista.

## Parámetros

**12. Diferencia entre v$parameter y v$spparameter con el ejemplo de db_files de tu evidencia.**
v$parameter es el valor que se está usando ahora en memoria y v$spparameter el que hay escrito en el SPFILE para el próximo arranque. Con db_files se ve claro: después de cambiarlo con SCOPE=SPFILE quedó 200 en memoria y 300 en el SPFILE, o sea, pendiente hasta reiniciar. Luego con el RESET se quitó del SPFILE.

**13. ¿Qué hace SCOPE=BOTH y cuál es el SCOPE por defecto con SPFILE?**
Cambia el valor en memoria y en el SPFILE a la vez, así que se aplica ya y se mantiene al reiniciar. En mi prueba open_cursors quedó en 500 en los dos sitios. Si la instancia arrancó con SPFILE, como la mía, el SCOPE por defecto es BOTH.

**14. ¿Por qué PROCESSES rechaza el cambio en memoria aunque v$parameter lo marque IMMEDIATE?**
Porque PROCESSES fija el tamaño de cosas que Oracle reserva en la memoria al arrancar, y eso no se puede cambiar en caliente. En la vista sale como IMMEDIATE, pero al probarlo con SCOPE=MEMORY me dio ORA-02095. En la práctica es estático: se cambia en el SPFILE y se reinicia.

**15. ¿Para qué sirve versionar el PFILE? Explica "configuration drift".**
El PFILE es la configuración en texto, y teniéndolo en Git cualquier cambio se ve con un git diff. También sirve para arrancar si se estropea el SPFILE. El configuration drift es cuando la configuración real se va separando de la documentada por cambios que nadie apunta, y comparando el PFILE se detecta.

## Memoria y procesos

**16. ¿Qué modelo de gestión de memoria usa tu instancia? Justifícalo con tres parámetros.**
ASMM. memory_target está a 0, así que no es AMM; sga_target está a 1536M, así que Oracle reparte la SGA solo; y pga_aggregate_target está a 512M, la PGA va aparte. Además db_cache_size y shared_pool_size están a 0, o sea, que el reparto lo decide Oracle (spool 08).

**17. Explica las cifras de tu experimento de caché: lecturas lógicas, físicas y tasa de acierto.**
La primera vez, con la caché vacía, salieron 355 lecturas lógicas y 260 físicas: los bloques se tuvieron que traer del disco. La segunda vez salieron 247 lógicas y 0 físicas, todo desde memoria, así que la tasa de acierto es del 100 %. Me salieron algo más de físicas que en la guía, supongo que porque la primera vez también tuvo que leer del disco bloques del diccionario.

**18. ¿Por qué no se suma el RSS de los procesos para saber la memoria de la instancia?**
Porque casi todo ese RSS es la SGA, que es compartida y sale contada en cada proceso. Si los sumara me darían varios GB, cuando docker stats dice que el contenedor usa unos 2,1 GB. Para saber la memoria real hay que mirar v$sga y v$pgastat.

**19. Relaciona el SPID de LGWR de v$process con el PID de /proc. ¿Por qué docker top muestra otro número?**
En v$process LGWR tiene el SPID 1257, y mirando /proc dentro del contenedor el proceso 1257 es db_lgwr_FREE, así que coinciden. docker top lo enseña con otro número (6382) porque lo mira desde fuera: el contenedor tiene su propia numeración de procesos y el anfitrión otra.

**20. ¿Qué hace PMON y qué hace SMON? Pon un ejemplo de cada uno.**
PMON limpia los procesos que se quedan colgados: si a alguien se le apaga el portátil a mitad de un UPDATE, PMON deshace su transacción y libera los bloqueos. SMON se encarga de recuperar la instancia al arrancar después de una caída, por ejemplo después de un SHUTDOWN ABORT.

## Seguridad y trabajo profesional

**21. ¿Por qué C##OPS_RO y no SYSTEM para observar? ¿Qué no puede hacer C##OPS_RO?**
Por mínimo privilegio: para mirar no hace falta poder romper nada, y con SYSTEM un error podría cambiar o borrar cosas. C##OPS_RO solo puede conectarse y leer el diccionario y las vistas V$. No puede crear ni modificar objetos, ni tocar datos, ni cambiar parámetros.

**22. ¿Qué dos vías de exposición de contraseñas corrige este laboratorio respecto al Lab 3?**
La primera, las contraseñas que quedaron escritas en V000 y en Git, que con V003 ya no sirven. La segunda, el sys/contraseña que el 08 del Lab 3 pasaba por la línea de comandos y se veía con ps; ahora el 08 entra con / as sysdba y no hay contraseña que ver.

## Preguntas tipo examen

| P | Respuesta | Justificación |
|---|---|---|
| P1 | **A, D** | Tras un SHUTDOWN IMMEDIATE todos los archivos tienen el mismo SCN, así que la copia es consistente y abre con un STARTUP normal. |
| P2 | **B** | Sin el movimiento online, toca poner el tablespace OFFLINE, copiar el archivo, hacer el RENAME y volver a ONLINE. |
| P3 | **A, C** | La hora de arranque es de la instancia y cambia en cada arranque, y CREATED es de la base y dice cuándo se creó. |
| P4 | **A, C** | En NOMOUNT ya están los parámetros en memoria pero no hay control file abierto, justo lo que hace falta para crear uno. |
| P5 | **B** | Para pasar a ARCHIVELOG hace falta el control file abierto y los datafiles cerrados, y eso es MOUNT. |
| P6 | **B** | Con SPFILE y sin poner SCOPE, el cambio va a memoria y al SPFILE. |
| P7 | **C** | PROCESSES es de toda la instancia y desde una PDB da ORA-65040, como me pasó en la Parte F. |
| P8 | **A, C** | Se arranca con un PFILE bueno y a partir de él se vuelve a crear el SPFILE. |
| P9 | **B** | Con MEMORY_TARGET a 0 y SGA_TARGET puesto gestiona ASMM, y el DB_CACHE_SIZE funciona como mínimo. |
| P10 | **A, D** | La zona de ordenación y la memoria de la sesión son privadas del proceso, lo demás es de la SGA. |
| P11 | **C** | El COMMIT no acaba hasta que LGWR escribe el redo en disco. |
| P12 | **B, D** | PMON y SMON están siempre, los otros dependen de cómo esté configurada la base. |
