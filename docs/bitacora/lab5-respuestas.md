# Lab 5 — Respuestas

Las evidencias están en `docs/bitacora/lab5/evidencia/`. Pongo solo la parte del nombre que va después de la fecha.

Nota: el Lab 4 lo hice con la versión anterior del kit, así que copié del kit v3 lo del Lab 5 (scripts/lab5, V004, V005 y grafico-indice.sh). En la Parte P el bloque 5 me dio ORA-00942 porque a C##OPS_RO le faltaba el permiso del alert log, que en la v3 se da en el Lab 4. Lo arreglé con el 11-ampliar-ops-ro.sql del kit, por eso hay dos spools del 15.

## Punto de partida y banco de medida

**1. Compara el paso 6 de tu copia lab4-final con el de la copia lab3-final. ¿Qué demuestra la diferencia?**
En mi caso las dos dicen "ninguno" (`00-backup-lab4` y `00-backup-lab3` del Lab 4). Como ya expliqué en el Lab 4, mis datafiles de V000 estaban en $ORACLE_HOME/dbs, que es un enlace al volumen, y por eso ya contaban como dentro. Ahora están todos en FREE/FREEPDB1 (paso 3) y la copia no depende de nada de fuera.

**2. Calcula el factor de bloqueo (bfr) de CLIENTE y de PEDIDO con sus estadísticas. ¿Por qué PEDIDO cabe casi el triple de filas por bloque?**
Con el `01-perfil-tiendasur`: CLIENTE 100000 / 1517 = 66 filas por bloque y PEDIDO 1000000 / 5320 = 188. Es por el tamaño de fila: un cliente ocupa 102 bytes y un pedido 32, más o menos un tercio.
En el reto con DBMS_ROWID me salió mínimo 2, media 191,6 y máximo 200 filas por bloque. La media sale más que 188 porque BLOCKS cuenta también bloques sin filas, y el de 2 es el último, que quedó a medias. Varían porque no todas las filas miden lo mismo.

**3. ¿Por qué el banco de medida ejecuta cada sentencia dos veces y mide solo la segunda?**
Porque la primera incluye el parse y cargar bloques en la caché. La segunda va en caliente y da siempre lo mismo, por ejemplo las 4 lecturas de la prueba_pk (`01-herramienta-medir`).

## Anatomía y tipos de índice

**4. Con tu evidencia 02: altura, número de hojas y factor de ramificación (fan-out) real de IX_PED_CLIENTE. ¿Qué altura tendría con mil millones de filas?**
En `02-anatomia-indice`: altura 3, 2216 hojas y 6 ramas. Fan-out real (2216 + 6 - 1) / 6 = 370 y máximo 543. Con mil millones serían unas 2,2 millones de hojas y altura 4, solo un nivel más.

**5. En el volcado de la hoja de PK_CLIENTE, localiza una entrada y descompón el ROWID que guarda en bloque y fila. ¿Por qué no aparece el número de archivo?**
La primera entrada es `00 00 03 1b 00 00` con clave `c1 02` (el 1): bloque 0x31B = 795 y fila 0, y la consulta del script lo confirma. No sale el archivo porque TBS_TIENDASUR es BIGFILE, tiene un solo datafile y los 4 bytes van enteros al número de bloque.

**6. Elige tres pruebas de la Parte D y explica, con las lecturas lógicas medidas, por qué un tipo de índice gana al otro.**
Del `03-tipos-indice`. El único del DNI baja Q13 de 1550 lecturas a 3 y además la base rechaza el duplicado (ORA-00001). En Q7 el cubriente lee 3, frente a 7 del compuesto y 15 de IX_PED_CLIENTE, porque no tiene que ir a la tabla ni ordenar. Y el parcial de pendientes lee 239 frente a 5008, porque solo guarda los pedidos en 'P' (54 hojas frente a 2933).

## Agrupamiento, equilibrio y sargabilidad

**7. ¿Por qué IX_PED_FECHA tiene un clustering factor cercano al número de bloques e IX_PED_CLIENTE cercano al de filas? ¿Lo arreglaría un REBUILD?**
En `04-agrupamiento-equilibrio` IX_PED_FECHA tiene CF 5220 (5320 bloques) e IX_PED_CLIENTE 999041 (1000000 filas). Los pedidos se cargaron en orden de fecha, pero el cliente está repartido al azar. Un REBUILD no cambia nada porque el CF depende del orden de la tabla, no del índice.

**8. Según tu curva de la Parte E, ¿a partir de qué porcentaje deja de compensar cada índice? Compáralo con la fórmula de la Unidad 3.**
La tabla entera son 5232 lecturas. El índice malo pasa de 4875 al 0,5 % a 9920 al 1 %, así que deja de compensar sobre el 0,53 %, justo lo que da 1/bfr = 1/188. El bueno aguanta hasta el 50 % (3938) y pierde al 100 %. El optimizador ya cambió al full scan desde el 0,2 %, más cerca del 0,11 % de la fórmula para disco.

**9. Explica por qué Q4b ('Gonz%') usa el índice y aun así lee más bloques que un recorrido completo.**
'Gonz%' es un rango desde el principio, así que puede usar el índice, pero esos clientes están repartidos por toda la tabla. Lee 2024 bloques para 2056 filas frente a 1523 del full scan (`05-sargabilidad`). Son el 2,1 % y con CF malo solo compensa por debajo de 1/66, un 1,5 %.

## Coste de escritura y mantenimiento

**10. Con tu evidencia 06: bloques modificados por INSERT con 4, 9 y 5 índices. ¿Qué índices de la propuesta del compañero rechazaste y con qué dato?**
En `06-indices`: 10,1 bloques por fila con 4 índices, 20,3 con 9 y 12,1 con 5. Rechacé IX_PED_CANAL e IX_PED_ESTADO (33 % y 25 % por valor, y no se usaron), IX_PED_FEC_CLI (no se usó) e IX_PED_CLI_FEC, que cambié por el cubriente. Solo me quedé IX_PED_CP, que sí se usó y es muy selectivo. V005 quita también IX_PED_CLIENTE (prefijo del cubriente) e IX_PED_IMPORTE (sin usos y probado como INVISIBLE). El precio es que el UPDATE de estado sube de 2 a 8,1 bloques, pero Q7 es la consulta más frecuente.

**11. Distingue, con tus gráficos de la Parte H, un índice con divisiones 50/50, uno con divisiones 90/10 y uno con patrón cola.**
IX_AZAR es el 50/50: 595 divisiones, 75 % de ocupación y el mapa mezclado. IX_CRECIENTE es el 90/10: 398 divisiones todas 90/10, 99,9 % y el mapa lleno de `#`. IX_COLA es la cola: 286 hojas vacías de 487, 47 % de entradas borradas y el mapa vacío a la izquierda y lleno a la derecha (gráficos `07a` y `07b`).

**12. ¿Qué remedio elegirías para el patrón cola en producción, en horario de servicio, y por qué?**
COALESCE: va en línea, no bloquea y en `07c-remedios` dejó el índice en 201 hojas y el siguiente mensaje en 2 lecturas (antes 288). Si necesitara recuperar espacio, SHRINK SPACE. REBUILD no, porque pide el doble de espacio y mucho redo para lo mismo.

## SCN, control files, redo y FRA

**13. Con tu evidencia 08: ¿qué valores compara Oracle al abrir para decidir si hace falta recuperación?**
El SCN de checkpoint que tiene el control file para cada datafile y el de la cabecera del archivo. En `08-scn-checkpoint` coincidían (6300360, y 6307007 tras el CHECKPOINT). Si la cabecera es más antigua, pide recuperación, como el ORA-01113 del simulacro D.

**14. ¿Qué protege la multiplexación del control file en el mismo volumen y qué no? ¿Cómo se haría en producción?**
Protege si se pierde o se estropea una copia, como en el simulacro B, que se arregló copiando control01.ctl. No protege de perder el disco, porque las tres están en la misma carpeta. En producción irían en discos distintos (o ASM) y con autobackup del control file en RMAN.

**15. Explica los estados CURRENT, ACTIVE e INACTIVE con tu evidencia 10 y por qué Oracle rechaza los tres borrados del paso 4.**
CURRENT es donde escribe LGWR, ACTIVE todavía hace falta para recuperar la instancia e INACTIVE ya se puede reutilizar. En `10-redo`, tras los cambios de log, el 1 quedó CURRENT y el 2 y el 3 ACTIVE.
4a (ORA-01623) y 4b (ORA-01609) fallan porque no se puede borrar el grupo CURRENT ni un miembro suyo. El 4c me dio ORA-00350 en vez de ORA-01567: tengo 3 grupos, así que no me quedaba con menos de dos, pero el grupo 2 aún no estaba archivado.

**16. Describe la cadena de causas del escenario de la FRA llena, desde ORA-19815 hasta que la base deja de aceptar cambios.**
Con la FRA a 2 MB, ARCn no pudo archivar el grupo 3 (ORA-19809 y ORA-16038) y el alert log avisó con ORA-19815 (`11-fra-llena`). Los grupos se quedan sin archivar, LGWR no puede reutilizarlos y las sesiones se quedan esperando en "log file switch (archiving needed)" con ORA-00257 para las conexiones nuevas. Al volver a 4 GB, ARCn lo archivó solo.

## Undo, WAL y simulacros

**17. Explica, con los pasos 1 a 4 de tu evidencia 12, la frase "lectores no bloquean a escritores".**
A cambia la fila 101 sin COMMIT y B la lee como ACTIVO sin esperar, gracias al undo (`12-lectura-consistente`). A tiene sus bloqueos TM y TX y no le afecta que B lea. Solo choca cuando B intenta escribir la misma fila: ORA-00054 a los 3 segundos.

**18. ¿Por qué un DELETE genera más undo que un INSERT? ¿Por qué el ROLLBACK también genera redo?**
Para deshacer un INSERT basta con saber qué fila borrar, pero un DELETE tiene que guardar la fila entera. En `12-undo-y-redo`: 142 bloques de undo el INSERT y 428 el DELETE. El ROLLBACK vuelve a modificar bloques, y todo cambio se protege con redo (4701 KB al deshacer el DELETE).

**19. Con tu evidencia 13: ¿dónde está la garantía de durabilidad justo después del COMMIT? ¿Y después del checkpoint?**
Después del COMMIT, en el redo log online: LGWR había escrito 1779 KB y DBWn 0 bloques, con 231 sucios en memoria (`13-lgwr-dbwn`). Después del CHECKPOINT ya está en los datafiles: DBWn escribió 346 bloques y quedaron 0 sucios.

**20. Simulacro A: explica roll forward y roll back con las filas 102 y 103 y las líneas del alert log.**
La 102 se cambió con COMMIT y la 103 sin COMMIT antes del ABORT (`14a-caida-instancia`). Al arrancar, "Beginning crash recovery" y "read 10 KB redo, 6 data blocks need recovery" son el roll forward, que rehace todo desde el checkpoint. Luego el roll back deshace la 103 con el undo. Resultado: 102 en EGRESADO y 103 en ACTIVO.

**21. Simulacros C y D: tu RTO y tu RPO en cada uno. ¿Por qué son tan distintos?**
C: RTO 63 s y RPO 1 s. D: RTO 8 s y RPO 0. En el C se restauró toda la base (1,3 GB) por una tabla y todo volvió al momento de la copia. En el D solo se restauró un archivo y RECOVER aplicó el redo hasta ese momento, por eso la fila 104 seguía cambiada. El RPO de 1 s del C es porque la copia se hizo justo antes.

**22. Clasifica cada error ORA- del bloque 5 de tu runbook: incidente real, efecto del laboratorio o ruido conocido.**
Del segundo `15-healthcheck`: ORA-63999 y ORA-01110 son del simulacro D, y los ORA-01113 del archivo 21 (tiendasur01) también, porque al reabrir la PDB Oracle tuvo que recuperarlo. ORA-00202 y ORA-205 son del simulacro B. Los ORA-00800 de VKTM y LGWR son ruido conocido, salen en cada arranque del contenedor. Ningún incidente real.

## RTO y RPO de los simulacros

| Simulacro | Qué se rompe | RTO | RPO |
|---|---|---|---|
| A | SHUTDOWN ABORT | unos 15 s | 0 |
| B | se borra control03.ctl | unos 13 s | 0 |
| C | DROP TABLE con PURGE | 63 s | 1 s |
| D | se pierde academia01.dbf | 8 s | 0 |

El A y el B no los cronometra el script, los saqué de las horas del alert log.

## Retrospectivas

### Simulacros A a D

Ninguno perdió datos confirmados y la verificación dio TODO OK. Me sorprendió que en el D Oracle cerrara toda la PDB, y que el C, por una sola tabla, tardara mucho más que el D. Cambiaría: autobackup del control file y copias RMAN periódicas (así la tabla se podría recuperar con RECOVER TABLE), control files y redo en otro disco, y cronómetro en los scripts A y B.

### Incidente: DML colgado por FRA llena (entorno de laboratorio)

A mí no me pasó sin querer (la FRA está al 2,1 %), lo relleno con el caso de la L.3:

- Impacto: sesiones esperando "log file switch (archiving needed)"; ninguna pérdida de datos.
- Detección: una carga no terminaba y v$session mostró el evento de espera.
- Causa raíz: repetir cargas de un millón de filas con ARCHIVELOG activo y una FRA de 4 GB.
- Mitigación: RMAN DELETE NOPROMPT ARCHIVELOG ALL (solo en laboratorio).
- Qué cambia: el runbook muestra el % de la FRA y hay que avisar al 80 %; antes de repetir cargas, mirar v$recovery_area_usage.
- Lección: el ORA-19815 avisa antes del bloqueo, pero hay que mirarlo.

## Simulacro de examen 1Z0-182

| P | Respuesta | Parte | Justificación |
|---|---|---|---|
| S1 | **B** | E | Como IX_PED_CLIENTE: casi un bloque por fila. |
| S2 | **A, C** | D, F | Las dos bajaron Q1 a 4 lecturas. |
| S3 | **B** | F | El plan mostraba TO_NUMBER sobre la columna. |
| S4 | **B** | D | Pocos valores y sin escrituras concurrentes. |
| S5 | **B** | D | Q6 hizo INDEX SKIP SCAN. |
| S6 | **B** | G | Se vuelve atrás con VISIBLE. |
| S7 | **A, C** | H | Los dos en línea, IX_COLA quedó en 201 hojas. |
| S8 | **B** | H | El rango con REVERSE hizo full scan. |
| S9 | **C** | G | UPDATE de estado sin índice: 2 bloques. |
| S10 | **B** | C | La altura sale del fan-out. |
| S11 | **A, C** | D | La IOT leyó 3 frente a 4. |
| S12 | **B** | I | Mismo SCN en control file y cabecera. |
| S13 | **C** | I | CKPT apunta el checkpoint. |
| S14 | **B, E** | O | En el simulacro B v$instance daba STARTED. |
| S15 | **B** | L | Se cambia con la base montada. |
| S16 | **B** | K | INACTIVE y archivado: se puede recrear. |
| S17 | **A, C** | M | Lectura consistente y la consulta AS OF. |
| S18 | **B** | M | El undo también se puede perder en una caída. |
| S19 | **A** | N | Tras el COMMIT DBWn llevaba 0 bloques. |
| S20 | **A** | I | Más checkpoints, menos redo que aplicar. |
| S21 | **B** | L | Lo que pasó en la L.2. |
| S22 | **B** | O | Aísla el archivo y el resto sigue. |
| S23 | **B** | O | Sin SYSTEM no puede seguir abierta. |
| S24 | **B** | O | Sin redo archivado no hay RECOVER. |
| S25 | **B, D** | M | Más undo y GUARANTEE si es crítica. |
| S26 | **A** | L | Da el uso de la FRA por tipo. |
