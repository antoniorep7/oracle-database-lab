# Lab 5 — Respuestas

Las evidencias que cito están en `docs/bitacora/lab5/evidencia/`, casi todas en `script/`. Para no alargar pongo solo la parte del nombre que va después de la fecha.

Una cosa antes de empezar: el Lab 4 lo hice con la versión anterior del kit (mi verificación era el 10, no el 16), así que para este lab copié del kit v3 la carpeta scripts/lab5, V004, V005 y grafico-indice.sh. En la Parte P el bloque 5 del runbook me dio ORA-00942 porque a C##OPS_RO le faltaba el permiso sobre V_$DIAG_ALERT_EXT, que en la v3 se da en el Lab 4. Ejecuté el 11-ampliar-ops-ro.sql del kit y ya salió bien. Por eso hay dos spools del 15.

## Punto de partida y banco de medida

**1. Compara el paso 6 de tu copia lab4-final con el de la copia lab3-final. ¿Qué demuestra la diferencia?**
A mí no me sale ninguna diferencia, las dos dicen "ninguno: el volumen contiene toda la base de datos" (`00-backup-lab4` y, en el Lab 4, `00-backup-lab3`). Como ya puse en el Lab 4, en mi caso los datafiles de V000 estaban en $ORACLE_HOME/dbs, pero esa carpeta es un enlace al volumen y por eso el backup ya los contaba como dentro. Lo que sí se ve ahora es que después de reubicarlos están todos en FREE/FREEPDB1 (en el paso 3 salen academia01, clinica01, fintech01, logistica01 y retail01) y que la copia no depende de nada que esté fuera del volumen. Sin el enlace, la del Lab 3 habría tenido cinco archivos fuera.

**2. Calcula el factor de bloqueo (bfr) de CLIENTE y de PEDIDO con sus estadísticas. ¿Por qué PEDIDO cabe casi el triple de filas por bloque?**
En el `01-perfil-tiendasur` CLIENTE tiene 100000 filas en 1517 bloques, 100000 / 1517 = 65,9, o sea unas 66 filas por bloque (en la guía salían 1467 bloques y 68, a mí me salieron algunos más). PEDIDO tiene 1000000 filas en 5320 bloques, bfr 188. Es por el tamaño de la fila: un cliente ocupa 102 bytes de media porque tiene email, nombre, apellidos, teléfono... y un pedido solo 32, que son casi todo números y códigos cortos. 102 entre 32 da más o menos 3.
También hice el reto con DBMS_ROWID, agrupando los pedidos por bloque. Me salió mínimo 2, media 191,6 y máximo 200 filas, en 5220 bloques. La media es más alta que 188 porque BLOCKS cuenta los 5320 bloques del segmento, también los que no tienen filas (cabeceras y demás), y yo solo contaba los que tenían alguna. El de 2 filas es el último, que se quedó a medias al terminar la carga. No todos tienen las mismas filas porque no todas miden lo mismo (importes y códigos postales con más o menos cifras) y cada bloque se llena hasta que la siguiente ya no cabe.

**3. ¿Por qué el banco de medida ejecuta cada sentencia dos veces y mide solo la segunda?**
Porque la primera vez hace cosas que no son de la consulta en sí: el parse, cargar los bloques en la caché y a veces limpiar bloques que dejaron cargas anteriores. Si se midiera la primera, el resultado dependería de qué se ejecutó antes. La segunda ya va en caliente y da siempre lo mismo, por ejemplo la prueba_pk me dio 4 lecturas lógicas igual que en la guía (`01-herramienta-medir`).

## Anatomía y tipos de índice

**4. Con tu evidencia 02: altura, número de hojas y factor de ramificación (fan-out) real de IX_PED_CLIENTE. ¿Qué altura tendría con mil millones de filas?**
En `02-anatomia-indice` sale altura 3, 2216 hojas y 6 ramas para 1000000 de entradas. El fan-out real es (2216 + 6 - 1) / 6 = 370 y el máximo 8028 / 14,8 = 543. Cada hoja tiene 451 entradas de 503 posibles, un 90 %, porque al crearlo deja el 10 % libre (PCTFREE). Con mil millones de filas tendría unas 2208969 hojas y altura 4, o sea, 5 lecturas para encontrar una fila. Solo un nivel más que ahora.

**5. En el volcado de la hoja de PK_CLIENTE, localiza una entrada y descompón el ROWID que guarda en bloque y fila. ¿Por qué no aparece el número de archivo?**
He cogido la primera entrada de la hoja: `row#0 ... data:(6): 00 00 03 1b 00 00`, con clave `c1 02`, que es el 1. Los 4 primeros bytes son el bloque, 0x31B = 795, y los 2 últimos la fila dentro del bloque, la 0. La consulta que hace el script después lo confirma, el cliente 1 está en el bloque 795, ranura 0. No sale el número de archivo porque TBS_TIENDASUR es un tablespace BIGFILE, solo tiene un datafile, así que no hace falta decir cuál es y los 4 bytes se usan para el bloque. El número de objeto tampoco va porque la tabla no está particionada y todas las filas son del mismo segmento.

**6. Elige tres pruebas de la Parte D y explica, con las lecturas lógicas medidas, por qué un tipo de índice gana al otro.**
Todo sale del `03-tipos-indice`. La primera, el índice único del DNI (Q13): sin índice hacía un TABLE ACCESS FULL de 1550 lecturas y con UX_CLI_DNI 3, porque va directo a la hoja. Además así es la base la que impide meter un DNI repetido (me dio el ORA-00001), no la aplicación.
La segunda, el historial de pedidos (Q7). Con IX_PED_CLIENTE eran 15 lecturas más un SORT, con el compuesto (id_cliente, fecha) 7 y sin SORT porque el índice ya devuelve las filas ordenadas, y con el cubriente 3, porque todas las columnas que pide la consulta están en el índice y no hace falta ir a la tabla.
La tercera, los pedidos pendientes (Q12). Con el índice normal sobre (estado, fecha) leía 5008 bloques y con el parcial IX_PED_PENDIENTES 239. El parcial solo guarda los pedidos en 'P', 20000 entradas en 54 hojas, frente al millón de entradas y 2933 hojas del otro, porque las filas que no están pendientes dan NULL y no entran en el índice.

## Agrupamiento, equilibrio y sargabilidad

**7. ¿Por qué IX_PED_FECHA tiene un clustering factor cercano al número de bloques e IX_PED_CLIENTE cercano al de filas? ¿Lo arreglaría un REBUILD?**
En el `04-agrupamiento-equilibrio` IX_PED_FECHA tiene CF 5220 (la tabla tiene 5320 bloques) e IX_PED_CLIENTE 999041 (y hay 1000000 filas). V004 cargó los pedidos en orden, así que las fechas van en el mismo orden que las filas en el disco y recorriendo el índice casi siempre sigues en el mismo bloque. El cliente en cambio está repartido al azar y casi cada entrada te manda a otro bloque. En el paso 2 lo calcula a mano y da exactamente lo mismo. Un REBUILD no lo arreglaría porque el CF depende de cómo están colocadas las filas de la tabla, no del índice. Habría que reordenar la tabla por cliente, y entonces sería el de la fecha el que empeoraría.

**8. Según tu curva de la Parte E, ¿a partir de qué porcentaje deja de compensar cada índice? Compáralo con la fórmula de la Unidad 3.**
Leer la tabla entera cuesta 5232 lecturas. Con el índice de CF malo al 0,5 % ya van 4875 y al 1 % 9920, así que deja de compensar sobre el 0,53 %, que es justo lo que da la fórmula: 1/bfr = 1/188 = 0,53 %. El índice de CF bueno aguanta mucho más, al 50 % todavía lee 3938 y solo pierde al 100 % (7875), o sea que compensa hasta más o menos dos tercios de la tabla. El optimizador fue más prudente que eso y ya cambió al recorrido completo desde el 0,2 %, que se parece más a lo que da la fórmula pensando en disco (0,11 %), donde una lectura aleatoria cuesta bastante más que una secuencial.

**9. Explica por qué Q4b ('Gonz%') usa el índice y aun así lee más bloques que un recorrido completo.**
En `05-sargabilidad` la Q4b devuelve 2056 filas con 2024 lecturas usando IX_CLI_APELLIDOS, mientras que el recorrido completo de CLIENTE (Q4a) son 1523. 'Gonz%' sí es sargable porque es un rango desde el principio de la cadena, por eso usa el índice. El problema es que los clientes que se apellidan así están repartidos por toda la tabla y casi cada fila está en un bloque distinto. Son el 2,1 % de los clientes y con un CF malo el índice solo compensa por debajo de 1/66, un 1,5 %.

## Coste de escritura y mantenimiento

**10. Con tu evidencia 06: bloques modificados por INSERT con 4, 9 y 5 índices. ¿Qué índices de la propuesta del compañero rechazaste y con qué dato?**
En el `06-indices`, por cada fila insertada: con los 4 índices de hoy 10,1 bloques y 1314 bytes de redo, con los 9 del compañero 20,3 bloques y 2666 bytes (el doble), y con los 5 de V005 12,1 bloques y 1609 bytes.
Rechacé IX_PED_CANAL e IX_PED_ESTADO porque cada valor es el 33 % y el 25 % de la tabla, muy por encima del 0,5 %, y además en el MONITORING USAGE salen como no usados. IX_PED_FEC_CLI tampoco se usó y dejaba a IX_PED_FECHA como redundante, cuando IX_PED_FECHA lo usan Q2, Q8 y Q9. IX_PED_CLI_FEC lo cambié por el cubriente IX_PED_CLI_FEC_CUB, que en la Parte D bajaba Q7 de 7 a 3 lecturas y como empieza por id_cliente sigue sirviendo para la clave ajena. De la propuesta solo me quedé con IX_PED_CP, que sí se usó y es muy selectivo (0,009 % por valor). Aparte, V005 quita IX_PED_CLIENTE, porque es el prefijo del cubriente, e IX_PED_IMPORTE, que no lo usó nadie y con él en INVISIBLE Q2 seguía en 272 lecturas.
Lo malo es que el UPDATE de estado pasa de 2 a 8,1 bloques, porque estado ahora está en el cubriente y en la columna virtual del parcial. Me compensa porque Q7 es la consulta que más se ejecuta.

**11. Distingue, con tus gráficos de la Parte H, un índice con divisiones 50/50, uno con divisiones 90/10 y uno con patrón cola.**
El de divisiones 50/50 es IX_AZAR (`07a-grafico-IX_AZAR`): 595 divisiones, ninguna 90/10, ocupación del 75,4 % y en el histograma hay hojas desde el 40 hasta el 99 %. En el mapa se ve todo mezclado con `=` y `-`, porque al meter claves al azar se parten hojas del medio y quedan a medias.
El de 90/10 es IX_CRECIENTE (`07a-grafico-IX_CRECIENTE`): 398 divisiones y todas 90/10, 99,9 % de ocupación y el mapa entero con `#`. Cada hoja se queda llena y se abre otra nueva por la derecha.
El patrón cola es IX_COLA antes de arreglarlo (`07b-grafico-IX_COLA-antes`): 487 hojas de las que 286 están vacías, 90403 entradas borradas (un 47 %) y en el mapa la izquierda vacía y la derecha llena de `#`. Ahí pedir el siguiente mensaje costaba 288 lecturas, y en un índice normal 2.

**12. ¿Qué remedio elegirías para el patrón cola en producción, en horario de servicio, y por qué?**
Un COALESCE. En `07c-remedios` pasó de 487 hojas a 201, quitó todas las entradas borradas y el siguiente mensaje volvió a costar 2 lecturas. Se hace en línea y no bloquea la tabla. Si además quisiera recuperar espacio usaría SHRINK SPACE, que dejó el índice en 1,69 MB en vez de 4. El REBUILD ONLINE no lo usaría porque necesita el doble de espacio y genera mucho redo para acabar igual (223 hojas y 2 lecturas). Lo ideal a largo plazo sería particionar la tabla por fecha y vaciar particiones enteras.

## SCN, control files, redo y FRA

**13. Con tu evidencia 08: ¿qué valores compara Oracle al abrir para decidir si hace falta recuperación?**
El SCN del último checkpoint que tiene apuntado el control file para cada datafile (v$datafile) y el que está en la cabecera del propio archivo (v$datafile_header). En el `08-scn-checkpoint` coincidían, 6300360 en los dos para system01, sysaux01 y academia01, y después del CHECKPOINT los dos subieron a 6307007. Los de PDB$SEED se quedan en 2098101 porque está en solo lectura. Si una cabecera tuviera un SCN más antiguo que el del control file, por ejemplo porque el archivo viene de una copia, Oracle pediría recuperación. Es lo que pasó en el simulacro D con el ORA-01113.

**14. ¿Qué protege la multiplexación del control file en el mismo volumen y qué no? ¿Cómo se haría en producción?**
Protege si se pierde o se estropea uno de los archivos. En el simulacro B se borró control03.ctl y bastó con copiar encima control01.ctl (`14b-perdida-controlfile`). Lo que no protege es perder el disco o el volumen, porque las tres copias están en /opt/oracle/oradata/FREE (`09-multiplexar-controlfile`) y se irían juntas, y tampoco un error que se escriba en las tres a la vez. En producción cada copia iría en un disco y una controladora distintos, o en ASM con redundancia, y además con el autobackup del control file activado en RMAN.

**15. Explica los estados CURRENT, ACTIVE e INACTIVE con tu evidencia 10 y por qué Oracle rechaza los tres borrados del paso 4.**
CURRENT es el grupo en el que está escribiendo LGWR. ACTIVE es uno que ya no se escribe pero que todavía haría falta si se cayera la instancia, porque el checkpoint aún no ha pasado por sus cambios. INACTIVE ya no hace falta y se puede reutilizar. En el `10-redo` al principio el grupo 2 (seq 14) era el CURRENT y el 1 y el 3 estaban INACTIVE. Después de los dos cambios de log el 1 pasó a CURRENT (seq 16) y el 2 y el 3 quedaron ACTIVE. Como yo tengo 3 grupos y no 2 como en la guía, redo02b.log seguía en INVALID al final, porque LGWR todavía no había vuelto a escribir en el grupo 2.
En el paso 4, el 4a da ORA-01623 porque no se puede borrar el grupo CURRENT y el 4b ORA-01609 porque tampoco se puede quitar un miembro del CURRENT. El 4c no me dio el ORA-01567 de la guía sino ORA-00350, que el log 2 tiene que archivarse. Al tener 3 grupos, borrar uno no me dejaba con menos de dos, pero como la base está en ARCHIVELOG Oracle no deja borrar un grupo que todavía no se ha archivado.

**16. Describe la cadena de causas del escenario de la FRA llena, desde ORA-19815 hasta que la base deja de aceptar cambios.**
En `11-fra-llena` se bajó la FRA a 2 MB cuando ya tenía 2,8 MB usados. Al pedir que se archivara el log, ARCn intentó archivar el grupo 3 (seq 18), no tenía sitio y salió ORA-19809, y por eso ORA-16038 (no se puede archivar). En el alert log quedó el ORA-19815 avisando de que la FRA estaba al 100 %. El grupo 3 se quedó ACTIVE sin archivar y el 1 pasó a CURRENT, también sin archivar. Cuando LGWR diera la vuelta y quisiera reutilizar un grupo no podría, porque no se puede sobrescribir uno sin archivar, y a partir de ahí las sesiones se quedan esperando en "log file switch (archiving needed)" y las conexiones nuevas reciben ORA-00257. En cuanto le volví a dar los 4 GB, ARCn lo reintentó solo y el grupo 3 salió archivado.

## Undo, WAL y simulacros

**17. Explica, con los pasos 1 a 4 de tu evidencia 12, la frase "lectores no bloquean a escritores".**
En el `12-lectura-consistente` la sesión A cambia la fila 101 a EGRESADO sin hacer COMMIT. En el paso 1 la sesión B la lee y ve ACTIVO sin esperarse a nada, porque Oracle le monta la versión confirmada con el undo. En el paso 2 se ve que A ya está usando 1 bloque de undo en el segmento 8, y en el paso 3 sus bloqueos: un TM en modo 3 sobre la tabla y un TX en modo 6 sobre la fila. Ni A le impide leer a B ni la lectura de B le molesta a A. Donde sí hay bloqueo es entre dos que escriben: en el paso 4 B intenta un SELECT FOR UPDATE de la misma fila y después de esperar 3 segundos da ORA-00054.

**18. ¿Por qué un DELETE genera más undo que un INSERT? ¿Por qué el ROLLBACK también genera redo?**
Para deshacer un INSERT solo hace falta saber qué fila borrar, pero para deshacer un DELETE hay que guardar la fila entera y sus entradas en los índices para poder volver a meterla. En el `12-undo-y-redo`, con 10000 filas, el INSERT dejó 142 bloques de undo (1136 KB) y el DELETE 428 (3424 KB). El ROLLBACK genera redo porque deshacer también es modificar bloques, y cualquier cambio en un bloque va protegido con redo por si la instancia se cae a mitad. Deshacer el DELETE generó 4701 KB de redo y deshacer el INSERT 355 KB.

**19. Con tu evidencia 13: ¿dónde está la garantía de durabilidad justo después del COMMIT? ¿Y después del checkpoint?**
En el `13-lgwr-dbwn`, después del UPDATE y el COMMIT se generaron 1779 KB de redo con 2 escrituras de LGWR, pero DBWn había escrito 0 bloques y había 231 bloques de la tabla sucios en memoria. Así que justo después del COMMIT el cambio solo está en disco en el redo log online, y esa es la garantía. Después del CHECKPOINT DBWn escribió 346 bloques y no quedó ninguno sucio, así que ya está también en los datafiles y ese redo ya no hace falta para recuperar la instancia.

**20. Simulacro A: explica roll forward y roll back con las filas 102 y 103 y las líneas del alert log.**
En el `14a-caida-instancia`, antes del SHUTDOWN ABORT la fila 102 se cambia a EGRESADO con COMMIT y la 103 a INACTIVO sin COMMIT. Al arrancar el alert log pone "Beginning crash recovery of 1 threads", "Recovery starting at checkpoint rba (logseq 20 block 42320), scn 6311809" y "read 10 KB redo, 6 data blocks need recovery". Eso es el roll forward: desde el último checkpoint se vuelve a aplicar todo el redo, tanto el cambio de la 102 como el de la 103 y el de su undo. Luego, con ese undo, se deshace lo que no estaba confirmado, que es el roll back. Al final la 102 sigue en EGRESADO y la 103 vuelve a ACTIVO. Todo esto lo hizo Oracle solo con un STARTUP normal.

**21. Simulacros C y D: tu RTO y tu RPO en cada uno. ¿Por qué son tan distintos?**
En el C (`14c-simulacro-restauracion`) me dio RTO 63 s y RPO 1 s, y en el D (`14d-perdida-datafile`) RTO 8 s y RPO 0. En el C, por una tabla, tuve que restaurar el volumen entero (1,3 GB de copia), así que tarda según lo grande que sea la base y además todo vuelve al momento de la copia. El RPO de 1 s no es real porque la copia se hizo justo antes del fallo, normalmente sería todo lo que haya pasado desde la última copia en frío. En el D solo se restauró un archivo (la copia era de 38 MB) y el RECOVER aplicó el redo archivado y el online hasta ese momento, por eso no se perdió nada: la fila 104 seguía en EGRESADO aunque ese cambio se hizo después de la copia.

**22. Clasifica cada error ORA- del bloque 5 de tu runbook: incidente real, efecto del laboratorio o ruido conocido.**
Miro el segundo `15-healthcheck`, el que salió bien. El ORA-63999 y el ORA-01110 del archivo 16 (17:18) son del laboratorio, del simulacro D: el proceso CKPT se encontró el archivo borrado y Oracle cerró FREEPDB1. Los dos ORA-01113 del archivo 21 (17:18) también son del laboratorio. El 21 es tiendasur01.dbf, y como la PDB se cerró de golpe, al volver a abrirla sus archivos estaban pendientes de recuperar y Oracle lo hizo solo antes de abrir. Los ORA-00800 de VKTM y LGWR (17:15 y 17:16) son ruido conocido, salen en cada arranque porque el contenedor no deja cambiar la prioridad de los procesos, ya los tenía en el Lab 4. Y el ORA-00202 de control03.ctl con el ORA-205 al montar (17:11) son del simulacro B. No hay ningún incidente real.

## RTO y RPO de los simulacros

| Simulacro | Qué se rompe | RTO | RPO |
|---|---|---|---|
| A | SHUTDOWN ABORT | unos 15 s (ABORT a las 17:09:53, abierta a las 17:10:08) | 0 |
| B | se borra control03.ctl | unos 13 s contando el arranque, 2 s desde el ORA-205 hasta abrir | 0 |
| C | DROP TABLE con PURGE | 63 s | 1 s (en la realidad, desde la última copia) |
| D | se pierde academia01.dbf | 8 s (FREEPDB1 cerrada de 17:18:17 a 17:18:25) | 0 |

Los scripts del A y del B no miden el tiempo, así que esos los he sacado de las horas del alert log.

## Retrospectivas

### Simulacros A a D

Los cuatro terminaron sin perder nada que estuviera confirmado y la verificación final dio TODO OK. Lo que no me esperaba fue que en el D Oracle cerrara la PDB entera con el ORA-63999 en vez de dejar caído solo el tablespace, y que al abrirla tuviera que recuperar también tiendasur01.dbf. También me llamó la atención que el C, que era "solo una tabla", fuera el más lento con diferencia (63 s) por tener que restaurar toda la base, y el D, que era perder un datafile, se arreglara en 8 s.
Cambiaría varias cosas: activar el autobackup del control file y hacer copias RMAN periódicas, porque con ARCHIVELOG la tabla del C se podría haber recuperado con RMAN RECOVER TABLE sin llevar toda la base al pasado. Pondría las copias del control file y de los redo en otro disco, y añadiría un cronómetro a los scripts del A y el B para no tener que ir al alert log a sacar el RTO.

### Incidente: DML colgado por FRA llena (entorno de laboratorio)

A mí no me llegó a pasar sin querer, la FRA está al 2,1 % (86 de 4096 MB según el runbook). Lo hice de forma controlada en la L.2, así que relleno la plantilla con el caso de la L.3 y lo que vi yo:

- Impacto: sesiones esperando "log file switch (archiving needed)"; ninguna pérdida de datos.
- Detección: una carga no terminaba y v$session mostró el evento de espera. En mi L.2 se veía antes, con el ORA-19815 en el alert log y los grupos con ARC = NO.
- Causa raíz: repetir varias veces la carga del millón de filas de TiendaSur con ARCHIVELOG activo y una FRA de 4 GB.
- Mitigación: RMAN DELETE NOPROMPT ARCHIVELOG ALL, que solo vale en laboratorio (en producción BACKUP ARCHIVELOG ALL DELETE INPUT). En mi L.2 bastó con volver a poner la FRA en 4 GB.
- Qué cambia: el runbook ya enseña el % de la FRA en el bloque 3 y habría que avisar al 80 %. Antes de repetir cargas grandes, mirar v$recovery_area_usage.
- Lección: el aviso (ORA-19815) sale antes de que la base se cuelgue, pero hay que mirarlo.

## Simulacro de examen 1Z0-182

| P | Respuesta | Parte | Justificación |
|---|---|---|---|
| S1 | **B** | E | Con un CF así casi cada fila está en un bloque distinto, como IX_PED_CLIENTE (999041). |
| S2 | **A, C** | D y F | El índice funcional bajó Q1 de 1486 a 4 lecturas y reescribirla con LOWER de 1523 a 4. |
| S3 | **B** | F | En el plan salía filter(TO_NUMBER("CODIGO_POSTAL")=30001). |
| S4 | **B** | D | Pocos valores y sin escrituras concurrentes, como el almacén de la Q14. |
| S5 | **B** | D | La Q6 filtrando solo por ciudad hizo INDEX SKIP SCAN. |
| S6 | **B** | G | Si algo va mal se vuelve a poner VISIBLE y no hay que reconstruir nada. |
| S7 | **A, C** | H | Los dos son en línea y los dos dejaron IX_COLA en 201 hojas. |
| S8 | **B** | H | El rango con el índice REVERSE acabó en TABLE ACCESS FULL (878 lecturas frente a 4). |
| S9 | **C** | G | El UPDATE de estado, que no estaba en ningún índice, eran 2 bloques. |
| S10 | **B** | C | La altura sale del fan-out, 1 + ceil(log_543(2216)) = 3. |
| S11 | **A, C** | D | La IOT leyó 3 bloques y el montículo con PK 4, porque no hay tabla aparte. |
| S12 | **B** | I | Control file y cabecera con el mismo SCN, 6300360. |
| S13 | **C** | I | CKPT apunta el checkpoint, los bloques los escribe DBWn. |
| S14 | **B, E** | O | En el simulacro B, en NOMOUNT, v$instance respondió STARTED; los parámetros ya están cargados del SPFILE. |
| S15 | **B** | L | El modo se cambia con la base montada, no abierta. |
| S16 | **B** | K | Si está INACTIVE y archivado no hace falta para nada y CLEAR LOGFILE lo vuelve a crear. |
| S17 | **A, C** | M | B leyó ACTIVO gracias al undo, y la consulta AS OF de hace 40 s también lo usa. |
| S18 | **B** | M | El undo también está en bloques que se pueden perder, hay que poder rehacerlos antes de deshacer. |
| S19 | **A** | N | Después del COMMIT DBWn llevaba 0 bloques, solo estaba el redo de LGWR. |
| S20 | **A** | I | Con un MTTR más bajo hace checkpoints incrementales más a menudo y queda menos redo por aplicar. |
| S21 | **B** | L | En la L.2 los grupos se quedaron sin archivar al llenarse la FRA. |
| S22 | **B** | O | Poniendo el archivo OFFLINE se aísla el problema y el resto sigue funcionando. |
| S23 | **B** | O | Sin SYSTEM la base no puede seguir abierta, hay que ir a MOUNT, RESTORE, RECOVER y OPEN. |
| S24 | **B** | O | Sin redo archivado no se puede hacer RECOVER, solo queda la copia en frío como en el C. |
| S25 | **B, D** | M | Más retención y más espacio de undo, y GUARANTEE si esa consulta no puede fallar. |
| S26 | **A** | L | Es la que daba el uso por tipo (ARCHIVED LOG 0,07 % en la L.1). |
