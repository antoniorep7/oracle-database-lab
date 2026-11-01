# Lab 5 — Respuestas

Las evidencias que cito están en `docs/bitacora/lab5/evidencia/` (casi todas en `script/`, las de V004, V005 y el runbook en `spool/`). Pongo solo la parte del nombre que va después de la fecha.

Nota antes de empezar: el Lab 4 lo hice con una versión anterior del kit (mi verificación era `scripts/lab4/10-verificacion-final.sh`, no el 16). Para este lab copié del kit v3 `scripts/lab5/`, V004, V005 y `scripts/ops/grafico-indice.sh`. En la Parte P el bloque 5 del runbook me dio ORA-00942 porque a C##OPS_RO le faltaba el permiso sobre V_$DIAG_ALERT_EXT, que en la v3 se da en el Lab 4. Lo arreglé ejecutando `scripts/lab4/11-ampliar-ops-ro.sql` del kit y luego ya salió todo (por eso hay dos spools del 15).

## Punto de partida y banco de medida (A, B)

**1. Compara el paso 6 de tu copia lab4-final con el de la copia lab3-final. ¿Qué demuestra la diferencia?**
En mi caso no hay diferencia: las dos dicen "ninguno: el volumen contiene toda la base de datos" (`00-backup-lab4.script.log` de este lab y `00-backup-lab3.script.log` del Lab 4). Ya lo comenté en el Lab 4: en mi entorno los datafiles de V000 acabaron en `$ORACLE_HOME/dbs`, pero esa carpeta es un enlace al volumen, así que el backup ya los veía "dentro". Lo que sí demuestra la copia de ahora es que después de reubicar los datafiles todos están en `FREE/FREEPDB1/` (paso 3: academia01, clinica01, fintech01, logistica01 y retail01 salen ahí) y que la copia no depende de nada fuera del volumen. Si el enlace no hubiera existido, la del Lab 3 habría tenido cinco archivos fuera y esta ninguno.

**2. Calcula el factor de bloqueo (bfr) de CLIENTE y de PEDIDO con sus estadísticas. ¿Por qué PEDIDO cabe casi el triple de filas por bloque?**
Con `01-perfil-tiendasur.script.log`: CLIENTE tiene 100000 filas en 1517 bloques, bfr = 100000 / 1517 = 65,9, o sea 66 filas por bloque (en la guía salían 1467 y 68, a mí me salieron algunos bloques más). PEDIDO tiene 1000000 filas en 5320 bloques, bfr = 188. La diferencia es el tamaño de fila: un cliente ocupa 102 bytes de media (email, nombre, apellidos, teléfono...) y un pedido solo 32 (números, una fecha y códigos cortos). 102 / 32 es más o menos 3, y por eso caben casi el triple.
Hice también el reto con DBMS_ROWID agrupando PEDIDO por bloque: mínimo 2 filas, media 191,6 y máximo 200, en 5220 bloques con filas. La media sale más alta que 188 porque BLOCKS cuenta los 5320 bloques del segmento por debajo de la marca de agua, también los de cabecera y los que no tienen filas, y yo solo contaba los que tenían. El de 2 filas es el último bloque, que se quedó a medias al acabar la carga. No todos tienen las mismas filas porque las filas no miden todas lo mismo (importes y códigos postales con distinto número de dígitos) y cada bloque se llena hasta que la siguiente ya no cabe.

**3. ¿Por qué el banco de medida ejecuta cada sentencia dos veces y mide solo la segunda?**
Porque la primera ejecución hace trabajo que no es de la consulta: el análisis (hard parse), traer bloques a la caché y a veces limpiar bloques de cargas anteriores. Si se midiera la primera, comparar dos consultas dependería de cuál se ejecutó antes. Midiendo la segunda, en caliente, las lecturas lógicas son estables: la prueba_pk me dio 4 lecturas, igual que en la guía (`01-herramienta-medir.script.log`).

## Anatomía y tipos de índice (C, D)

**4. Con tu evidencia 02: altura, número de hojas y factor de ramificación (fan-out) real de IX_PED_CLIENTE. ¿Qué altura tendría con mil millones de filas?**
En `02-anatomia-indice.script.log`: altura 3, 2216 hojas y 6 ramas, con 1000000 entradas. El fan-out real es (2216 + 6 - 1) / 6 = 370 y el máximo 8028 / 14,8 = 543. Las hojas tienen 451 entradas de un máximo de 503, un 90 %, por el PCTFREE 10. Con mil millones de filas serían unas 2208969 hojas y la altura pasaría a 4, o sea, 5 lecturas para encontrar una fila. De un millón a mil millones solo un nivel más.

**5. En el volcado de la hoja de PK_CLIENTE, localiza una entrada y descompón el ROWID que guarda en bloque y fila. ¿Por qué no aparece el número de archivo?**
La primera entrada de la hoja es `row#0 ... data:(6): 00 00 03 1b 00 00` con la clave `c1 02` (el número 1). Los 4 primeros bytes son el bloque: 0x0000031B = 795, y los 2 últimos la fila dentro del bloque: 0. La consulta de comprobación lo confirma: el cliente 1 está en el bloque 795, ranura 0. No aparece el archivo porque TBS_TIENDASUR es BIGFILE: tiene un solo datafile, así que no hace falta guardar cuál es y los 4 bytes se usan enteros para el número de bloque. Tampoco va el número de objeto, porque el índice es de una tabla no particionada y todas sus filas son del mismo segmento.

**6. Elige tres pruebas de la Parte D y explica, con las lecturas lógicas medidas, por qué un tipo de índice gana al otro.**
Todo de `03-tipos-indice.script.log`:
- Único sobre DNI (Q13): sin índice 1550 lecturas con TABLE ACCESS FULL, con UX_CLI_DNI 3. Gana porque va directo a una hoja, y además la base rechaza el duplicado con ORA-00001, que antes dependía de la aplicación.
- Cubriente para el historial (Q7): con IX_PED_CLIENTE 15 lecturas y un SORT; con (id_cliente, fecha) 7 y sin SORT porque el índice ya da las filas ordenadas; con el cubriente 3, porque todas las columnas están en el índice y no tiene que ir a la tabla.
- Parcial para los pendientes (Q12): con (estado, fecha) 5008 lecturas; con IX_PED_PENDIENTES 239. El parcial solo tiene 20000 entradas (54 hojas) frente a 1000000 (2933 hojas), porque las filas que no son 'P' dan NULL en la columna virtual y no se guardan.
También el bitmap del almacén: 4073 lecturas sin bitmaps y 528 con los cuatro combinados con BITMAP AND, y el bitmap de región ocupa 114 hojas frente a 1954 del B-tree. Pero solo vale ahí, no en OLTP.

## Agrupamiento, equilibrio y sargabilidad (E, F)

**7. ¿Por qué IX_PED_FECHA tiene un clustering factor cercano al número de bloques e IX_PED_CLIENTE cercano al de filas? ¿Lo arreglaría un REBUILD?**
En `04-agrupamiento-equilibrio.script.log` IX_PED_FECHA tiene CF 5220 con 5320 bloques, e IX_PED_CLIENTE 999041 con 1000000 filas. V004 cargó los pedidos con ORDER BY, así que la fecha va en el mismo orden que las filas en disco: recorriendo el índice por fecha casi siempre sigues en el mismo bloque. El cliente se repartió al azar, y al recorrer el índice por cliente casi cada entrada salta a otro bloque. Lo calculé también a mano en el paso 2 y da exactamente lo mismo. Un REBUILD no lo arregla porque el CF depende de cómo están ordenadas las filas en la tabla, no del índice. Habría que reordenar la tabla por id_cliente, y entonces empeoraría el de la fecha.

**8. Según tu curva de la Parte E, ¿a partir de qué porcentaje deja de compensar cada índice? Compáralo con la fórmula de la Unidad 3.**
La tabla entera cuesta 5232 lecturas. Con el índice malo (CF de cliente) al 0,5 % llevo 4875 y al 1 % ya 9920, así que el punto de equilibrio está sobre el 0,53 %. La fórmula da justo eso: k/r < 1/bfr = 1/188 = 0,53 %, porque con CF malo cada fila es un bloque distinto. Con el índice bueno al 50 % todavía lee 3938 y solo pierde al 100 % (7875), así que compensa hasta más o menos dos tercios de la tabla. El optimizador fue más prudente: ya eligió el recorrido completo a partir del 0,2 %, que se acerca a lo que da la fórmula en disco (0,11 %), donde la lectura aleatoria cuesta más que la secuencial.

**9. Explica por qué Q4b ('Gonz%') usa el índice y aun así lee más bloques que un recorrido completo.**
En `05-sargabilidad.script.log` Q4b devuelve 2056 filas con 2024 lecturas por IX_CLI_APELLIDOS, y el recorrido completo de CLIENTE (Q4a) cuesta 1523. 'Gonz%' es sargable porque es un rango por el principio de la cadena, así que el índice se puede usar, pero los Gonzálezes están repartidos por toda la tabla: casi cada fila es un bloque distinto. Son el 2,1 % de los clientes, y con CF malo el índice compensa por debajo de 1/66 = 1,5 %. Sargable no quiere decir que el índice vaya a ser más barato.

## Coste de escritura y mantenimiento (G, H)

**10. Con tu evidencia 06: bloques modificados por INSERT con 4, 9 y 5 índices. ¿Qué índices de la propuesta del compañero rechazaste y con qué dato?**
En `06-indices.script.log`, por fila: con 4 índices 10,1 bloques y 1314 bytes de redo; con los 9 del compañero 20,3 bloques y 2666 bytes (el doble); con V005, 5 índices, 12,1 bloques y 1609 bytes. Rechacé:
- IX_PED_CANAL y IX_PED_ESTADO: no son viables, cada valor es el 33,3 % y el 25 % de la tabla, muy por encima del 0,5 %, y MONITORING USAGE dijo NO en los dos.
- IX_PED_FEC_CLI: no se usó (USA = NO) y hacía redundante a IX_PED_FECHA, que sí usan Q2, Q8 y Q9.
- IX_PED_CLI_FEC: lo cambié por el cubriente IX_PED_CLI_FEC_CUB (Q7 de 7 a 3 lecturas en la Parte D), que además empieza por id_cliente y cubre la clave ajena.
De la propuesta me quedé IX_PED_CP, que sí se usó y es viable (0,009 % por valor). Además V005 quita IX_PED_CLIENTE (es prefijo del cubriente) e IX_PED_IMPORTE (no se usó, y con él INVISIBLE Q2 seguía en 272 lecturas). Un precio que se ve: el UPDATE de estado pasa de 2 a 8,1 bloques, porque estado está en el cubriente y en la columna virtual del parcial. Lo acepto porque Q7 es la consulta más frecuente.

**11. Distingue, con tus gráficos de la Parte H, un índice con divisiones 50/50, uno con divisiones 90/10 y uno con patrón cola.**
- 50/50, IX_AZAR (`07a-grafico-IX_AZAR.script.log`): 595 divisiones, ninguna 90/10, ocupación 75,4 % y el histograma repartido entre el 40 y el 99 %. El mapa es una mezcla de `=` y `-` por todas partes, porque las claves al azar parten hojas del medio y las dejan a medias.
- 90/10, IX_CRECIENTE (`07a-grafico-IX_CRECIENTE.script.log`): 398 divisiones y las 398 son 90/10, 99,9 % de ocupación, 398 hojas en el tramo 90-99 % y el mapa todo `#`. Cada hoja se queda llena y se abre una nueva a la derecha.
- Cola, IX_COLA (`07b-grafico-IX_COLA-antes.script.log`): 487 hojas pero 286 vacías (0-9 %), 90403 entradas borradas (47 %) y el mapa vacío a la izquierda y `#` a la derecha. Buscar el siguiente mensaje costaba 288 lecturas frente a 2 de un índice sano.

**12. ¿Qué remedio elegirías para el patrón cola en producción, en horario de servicio, y por qué?**
COALESCE. En `07c-remedios.script.log` bajó de 487 a 201 hojas, dejó 0 borradas y el siguiente mensaje pasó de 288 a 2 lecturas. Va en línea y no bloquea la tabla. SHRINK SPACE da lo mismo y además devuelve espacio (de 4 a 1,69 MB), así que lo usaría si el espacio importase. REBUILD ONLINE no, porque necesita el doble de espacio y genera mucho redo para conseguir lo mismo (223 hojas y 2 lecturas). A largo plazo lo bueno sería particionar por fecha y vaciar particiones enteras.

## SCN, control files, redo y FRA (I a L)

**13. Con tu evidencia 08: ¿qué valores compara Oracle al abrir para decidir si hace falta recuperación?**
Compara el SCN de checkpoint de cada datafile que tiene apuntado el control file (v$datafile) con el que hay en la cabecera del propio archivo (v$datafile_header). En `08-scn-checkpoint.script.log` coincidían: 6300360 en los dos para system01, sysaux01 y academia01, y después del CHECKPOINT los dos pasaron a 6307007. Los de PDB$SEED se quedaron en 2098101 porque está en solo lectura. Si al abrir una cabecera tuviera un SCN más viejo que el control file (por ejemplo, un archivo restaurado de una copia), Oracle pediría recuperación, que es justo el ORA-01113 que vi en el simulacro D.

**14. ¿Qué protege la multiplexación del control file en el mismo volumen y qué no? ¿Cómo se haría en producción?**
Protege de perder o estropear un archivo suelto: en el simulacro B se borró control03.ctl y lo arreglé copiando control01.ctl (`14b-perdida-controlfile.script.log`). No protege de perder el disco o el volumen, porque las tres copias (`09-multiplexar-controlfile.script.log`, todas en /opt/oracle/oradata/FREE) se irían juntas, ni de un error que se escriba en las tres a la vez. En producción cada copia iría en un disco y una controladora distintos, o en grupos de ASM con redundancia, y con CONFIGURE CONTROLFILE AUTOBACKUP ON en RMAN.

**15. Explica los estados CURRENT, ACTIVE e INACTIVE con tu evidencia 10 y por qué Oracle rechaza los tres borrados del paso 4.**
En `10-redo.script.log` al empezar el grupo 2 (seq 14) era CURRENT y el 1 y el 3 INACTIVE. CURRENT es el que está escribiendo LGWR. ACTIVE es uno que ya no se escribe pero todavía haría falta para recuperar la instancia, porque el checkpoint no ha llegado a sus cambios: después de los dos cambios de log tenía el 1 CURRENT (seq 16) y el 2 y el 3 ACTIVE. INACTIVE ya no hace falta y se puede reutilizar. Yo tengo 3 grupos y no 2 como en la guía, por eso después de dos cambios redo02b.log seguía INVALID: LGWR todavía no había vuelto al grupo 2.
Los borrados: 4a da ORA-01623 porque no se puede borrar el grupo CURRENT; 4b da ORA-01609 porque tampoco se puede quitar un miembro del CURRENT. El 4c a mí no me dio ORA-01567 sino ORA-00350 ("log 2 needs to be archived"): como tengo 3 grupos, borrar uno no me dejaba con menos de dos, pero la base está en ARCHIVELOG y el grupo 2 aún no estaba archivado, y Oracle no deja borrar un grupo con redo sin archivar.

**16. Describe la cadena de causas del escenario de la FRA llena, desde ORA-19815 hasta que la base deja de aceptar cambios.**
Con `11-fra-llena.script.log`: la FRA se bajó a 2 MB y ya tenía 2,8 MB usados. Al hacer ARCHIVE LOG CURRENT, ARCn intentó archivar el grupo 3 (seq 18), no había sitio y dio ORA-19809, y por eso ORA-16038 (no se puede archivar). En el alert log quedó ORA-19815 avisando de que la FRA estaba al 100 %. El grupo 3 se quedó ACTIVE con ARC = NO, y el 1 (seq 19) pasó a CURRENT sin archivar. Cuando LGWR diera la vuelta y necesitara reutilizar un grupo, no podría, porque ninguno está archivado: las sesiones se quedan esperando "log file switch (archiving needed)" y las conexiones nuevas reciben ORA-00257. Al devolverle los 4 GB, ARCn reintentó solo y el grupo 3 pasó a YES.

## Undo, WAL y simulacros (M a P)

**17. Explica, con los pasos 1 a 4 de tu evidencia 12, la frase "lectores no bloquean a escritores".**
En `12-lectura-consistente.script.log` la sesión A cambió la fila 101 a EGRESADO sin hacer COMMIT. En el paso 1, B la lee y ve ACTIVO sin esperar: Oracle le reconstruye la versión confirmada con el undo. En el paso 2 se ve que A ya usa 1 bloque y 1 registro de undo en el segmento 8, y en el paso 3 sus bloqueos: TM en modo 3 sobre la tabla y TX en modo 6 sobre la fila. Ni A bloqueó la lectura de B ni la lectura de B molestó a A. Lo único que choca es escritor contra escritor: en el paso 4, B intenta un SELECT FOR UPDATE de la misma fila y a los 3 segundos da ORA-00054.

**18. ¿Por qué un DELETE genera más undo que un INSERT? ¿Por qué el ROLLBACK también genera redo?**
Para deshacer un INSERT basta con apuntar "borra este ROWID". Para deshacer un DELETE hay que guardar la fila entera y sus entradas de índice para volver a meterlas. En `12-undo-y-redo.script.log`, con 10000 filas, el INSERT dejó 142 bloques de undo (1136 KB) y el DELETE 428 (3424 KB). El ROLLBACK genera redo porque deshacer es volver a cambiar los bloques de datos, y todo cambio en un bloque se protege con redo por si la instancia cae a mitad del ROLLBACK. Deshacer el DELETE generó 4701 KB de redo, y deshacer el INSERT 355 KB.

**19. Con tu evidencia 13: ¿dónde está la garantía de durabilidad justo después del COMMIT? ¿Y después del checkpoint?**
En `13-lgwr-dbwn.script.log`, después del UPDATE y el COMMIT se generaron 1779 KB de redo con 2 escrituras de LGWR, pero DBWn había escrito 0 bloques y la tabla tenía 231 bloques sucios en memoria. O sea, después del COMMIT el dato nuevo solo está en disco en el redo log online, y esa es la garantía. Después del CHECKPOINT DBWn escribió 346 bloques y quedaron 0 sucios: ya está también en los datafiles y ese redo deja de hacer falta para una recuperación de instancia.

**20. Simulacro A: explica roll forward y roll back con las filas 102 y 103 y las líneas del alert log.**
Antes del SHUTDOWN ABORT la 102 se cambió a EGRESADO con COMMIT y la 103 a INACTIVO sin COMMIT (`14a-caida-instancia.script.log`). Al arrancar, el alert log dice "Beginning crash recovery of 1 threads", "Recovery starting at checkpoint rba (logseq 20 block 42320), scn 6311809" y "read 10 KB redo, 6 data blocks need recovery". Ese es el roll forward: desde el último checkpoint se rehacen todos los cambios del redo, el de la 102, el de la 103 y también el de su undo. Después, con ese undo, se hace el roll back de lo que no estaba confirmado. Resultado: la 102 en EGRESADO (durabilidad) y la 103 otra vez en ACTIVO (atomicidad). Nadie hizo nada, bastó con un STARTUP.

**21. Simulacros C y D: tu RTO y tu RPO en cada uno. ¿Por qué son tan distintos?**
En el C (`14c-simulacro-restauracion.script.log`) RTO 63 s y RPO 1 s; en el D (`14d-perdida-datafile.script.log`) RTO 8 s y RPO 0. En el C, por una sola tabla tuve que restaurar el volumen entero (la copia ocupa 1,3 GB), así que el RTO depende del tamaño de la base, y todo vuelve al momento de la copia. El RPO de 1 s es mentira, porque la copia se hizo justo antes; en la vida real sería el tiempo desde la última copia en frío. En el D solo se restauró un archivo (una pieza de 38 MB) y RECOVER aplicó el redo archivado y el online hasta ese momento, así que no se perdió nada: la fila 104 seguía en EGRESADO aunque el cambio era posterior a la copia.

**22. Clasifica cada error ORA- del bloque 5 de tu runbook: incidente real, efecto del laboratorio o ruido conocido.**
Del `15-healthcheck.spool.log` bueno (el segundo):
- ORA-63999 y ORA-01110 del archivo 16 (17:18): efecto del laboratorio, es el simulacro D. El proceso CKPT se encontró el archivo borrado y Oracle cerró FREEPDB1.
- ORA-01113 "file 21 needs media recovery" (17:18, dos veces): efecto del laboratorio. El 21 es tiendasur01.dbf. Como la PDB se cerró de golpe por el ORA-63999, al abrirla otra vez sus archivos estaban marcados para recuperar y Oracle los recuperó solo ("Media Recovery Complete") antes de abrir.
- ORA-00800 Set Priority Failed de VKTM y LGWR (17:15 y 17:16): ruido conocido. Salen en cada arranque del contenedor, que no deja subir la prioridad de los procesos; ya los tenía en el Lab 4.
- ORA-00202 de control03.ctl y ORA-205 al montar (17:11): efecto del laboratorio, el simulacro B.
Ninguno es un incidente real.

## Simulacros: RTO y RPO

| Simulacro | Fallo | RTO | RPO | De dónde sale |
|---|---|---|---|---|
| A | SHUTDOWN ABORT | unos 15 s (ABORT a las 17:09:53, abierta a las 17:10:08) | 0 | alert log y `14a-caida-instancia` |
| B | Se borra control03.ctl | unos 13 s de arranque; desde el ORA-205 hasta abrir, 2 s | 0 | alert log y `14b-perdida-controlfile` |
| C | DROP TABLE ... PURGE | 63 s | 1 s (en la realidad, desde la última copia) | `14c-simulacro-restauracion` |
| D | Se pierde academia01.dbf | 8 s (FREEPDB1 cerrada de 17:18:17 a 17:18:25) | 0 | `14d-perdida-datafile` |

Los scripts A y B no cronometran, así que esos dos los saqué de las horas del alert log.

## Retrospectivas

### Game day (simulacros A-D)

- Qué salió bien: los cuatro terminaron sin perder nada confirmado y la verificación final dio TODO OK.
- Qué me sorprendió: en el D Oracle cerró la PDB entera (ORA-63999) en vez de dejar solo el tablespace caído, y al abrirla tuvo que recuperar también tiendasur01.dbf. También que el C, que es "solo una tabla", fue el más lento (63 s) porque restaura toda la base, y el D, que es un archivo de datos, tardó 8 s.
- Qué cambiaría: activaría CONFIGURE CONTROLFILE AUTOBACKUP ON y haría copias RMAN periódicas, porque con ARCHIVELOG el borrado de la tabla se podría haber arreglado con RMAN RECOVER TABLE sin devolver toda la base al pasado. Pondría las copias del control file y de los redo en otro disco, y añadiría cronómetro a los scripts A y B para no tener que sacar el RTO del alert log.

### Incidente: DML colgado por FRA llena (entorno de laboratorio, caso de la Parte L.3)

A mí no me llegó a pasar sin querer: la FRA está al 2,1 % (86 de 4096 MB en el runbook). Lo he vivido de forma controlada en la L.2 y la plantilla la relleno para el caso de la guía:

- Impacto: sesiones esperando "log file switch (archiving needed)"; ninguna pérdida de datos.
- Detección: una carga no terminaba y v$session mostró el evento de espera. En mi L.2 se veía antes: ORA-19815 en el alert log y grupos con ARC = NO.
- Causa raíz: repetir varias veces la carga de un millón de filas de TiendaSur con ARCHIVELOG activo y una FRA de 4 GB.
- Mitigación: RMAN DELETE NOPROMPT ARCHIVELOG ALL (solo vale en laboratorio; en producción BACKUP ARCHIVELOG ALL DELETE INPUT). En la L.2 bastó con devolverle el tamaño a la FRA.
- Qué cambia: el runbook ya enseña el % de la FRA en el bloque 3 y habría que alertar al 80 %. Antes de repetir cargas, mirar v$recovery_area_usage.
- Lección: el aviso (ORA-19815) llega antes que el bloqueo, pero si nadie mira el alert log no sirve de nada.

## Simulacro de examen 1Z0-182

| P | Respuesta | Parte | Justificación |
|---|---|---|---|
| S1 | **B** | E | CF cerca de las filas = casi un bloque distinto por fila, como IX_PED_CLIENTE (999041). |
| S2 | **A, C** | D, F | El índice funcional (1486 a 4 lecturas) o aplicar la función al dato (1523 a 4). |
| S3 | **B** | F | En el plan salía filter(TO_NUMBER("CODIGO_POSTAL")=30001). |
| S4 | **B** | D | Pocos valores y se consulta sin escrituras concurrentes: el almacén de Q14. |
| S5 | **B** | D | Q6 solo por ciudad hizo INDEX SKIP SCAN sobre IX_CLI_PAIS_CIUDAD. |
| S6 | **B** | G | INVISIBLE se deshace con un VISIBLE y no hay que reconstruir nada. |
| S7 | **A, C** | H | COALESCE y SHRINK SPACE van en línea; los dos dejaron IX_COLA en 201 hojas. |
| S8 | **B** | H | El rango con REVERSE hizo TABLE ACCESS FULL (878 lecturas frente a 4). |
| S9 | **C** | G | El UPDATE de una columna sin índice se queda en 2 bloques (estado con 4 índices). |
| S10 | **B** | C | La altura sale del fan-out: 1 + ceil(log_543(2216)) = 3. |
| S11 | **A, C** | D | La IOT leyó 3 bloques frente a 4 del montículo + PK: no hay tabla aparte. |
| S12 | **B** | I | Control file frente a cabecera: 6300360 y 6300360, sincronizado. |
| S13 | **C** | I | CKPT apunta el checkpoint; DBWn es el que escribe los bloques. |
| S14 | **B, E** | O | En el simulacro B, en NOMOUNT, v$instance respondió STARTED; v$parameter sale del SPFILE ya cargado. |
| S15 | **B** | L | El cambio de modo va con la base montada, no abierta. |
| S16 | **B** | K | Un grupo INACTIVE y archivado no hace falta para nada: CLEAR LOGFILE lo vuelve a crear. |
| S17 | **A, C** | M | B leyó ACTIVO con el undo y la consulta AS OF de hace 40 s también. |
| S18 | **B** | M | El undo está en bloques que también se pierden si cae la instancia: hay que rehacerlos antes de deshacer. |
| S19 | **A** | N | Tras el COMMIT DBWn llevaba 0 bloques; solo estaba el redo escrito por LGWR. |
| S20 | **A** | I | Con un MTTR más bajo Oracle hace checkpoints incrementales más a menudo y hay menos redo que aplicar. |
| S21 | **B** | L | En la L.2 los grupos se quedaron con ARC = NO al llenarse la FRA. |
| S22 | **B** | O | Poner el archivo OFFLINE aísla el daño y el resto sigue en servicio. |
| S23 | **B** | O | Con SYSTEM no se puede seguir abierta: MOUNT, RESTORE, RECOVER y OPEN. |
| S24 | **B** | O | Sin redo archivado no hay RECOVER: solo queda la copia en frío, como en el simulacro C. |
| S25 | **B, D** | M | Más retención y espacio de undo, y GUARANTEE si la consulta no puede fallar. |
| S26 | **A** | L | V$RECOVERY_AREA_USAGE da el uso por tipo de archivo (ARCHIVED LOG 0,07 % en la L.1). |
