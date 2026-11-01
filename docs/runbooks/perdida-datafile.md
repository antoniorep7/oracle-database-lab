# Runbook: pérdida de un datafile de negocio

Para quien esté de guardia. Sale del simulacro D del Lab 5, donde se borró `academia01.dbf` (file 16) de FREEPDB1 y se recuperó en 8 segundos sin perder nada.

## 1. Síntoma

Una consulta falla con estos tres errores juntos:

- `ORA-01116: error in opening database file N` → Oracle no puede abrir el archivo N.
- `ORA-01110: data file N: '/ruta/archivo.dbf'` → qué archivo es.
- `ORA-27041: unable to open file` + `Linux Error: 2: No such file or directory` → la causa en el sistema operativo (el archivo no está).

Si en el alert log aparece además `ORA-63999: data file suffered media failure`, Oracle ya ha cerrado la PDB entera (queda MOUNTED). La CDB y las demás PDB siguen funcionando.

## 2. Comprobaciones antes de tocar nada

Como SYSDBA (`bash scripts/ops/sysdba.sh` o `docker exec -it oralab-26ai sqlplus / as sysdba`):

```sql
SELECT log_mode FROM v$database;             -- tiene que ser ARCHIVELOG
SELECT name, open_mode FROM v$pdbs;          -- ¿la PDB sigue READ WRITE o está MOUNTED?
```

```
RMAN> LIST BACKUP OF DATAFILE N;             -- tiene que haber una copia
```

- Si la base está en NOARCHIVELOG no se puede usar este runbook: solo queda restaurar la última copia en frío (`restaurar-estado.sh`) y se pierde todo lo posterior. Avisar antes de hacerlo.
- Si no hay copia RMAN del archivo, parar y avisar.
- Si el archivo es de SYSTEM o del undo de la raíz, tampoco vale este runbook: hay que hacer SHUTDOWN ABORT, STARTUP MOUNT, RESTORE, RECOVER y OPEN.
- Si la PDB está cerrándose, esperar a que `v$pdbs` diga MOUNTED. Si se lanza el RESTORE a mitad del cierre falla (ORA-01089).

## 3. Reparación

SQL como SYSDBA y RMAN con `docker exec -it oralab-26ai rman target /`.

**Camino A: la PDB sigue READ WRITE** (solo está caído ese tablespace)

```
SQL>  ALTER SESSION SET CONTAINER = FREEPDB1;
SQL>  ALTER DATABASE DATAFILE N OFFLINE;
RMAN> RESTORE DATAFILE N;
RMAN> RECOVER DATAFILE N;
SQL>  ALTER DATABASE DATAFILE N ONLINE;      -- otra vez dentro de FREEPDB1
```

Si el OFFLINE da `ORA-01089` es que Oracle está cerrando la PDB justo en ese momento: pasar al camino B.

**Camino B: la PDB está MOUNTED** (fue lo que me pasó a mí)

```
RMAN> RESTORE DATAFILE N;
RMAN> RECOVER DATAFILE N;
SQL>  ALTER SESSION SET CONTAINER = FREEPDB1;
SQL>  ALTER DATABASE DATAFILE N ONLINE;
SQL>  ALTER SESSION SET CONTAINER = CDB$ROOT;
SQL>  ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;
```

Al abrir la PDB puede salir `ORA-01113: file X needs media recovery` en otros archivos de la misma PDB (a mí me pasó con el 21, tiendasur01.dbf). Es por el cierre brusco: Oracle los recupera solo con el redo y abre. Si no abre, `RECOVER DATAFILE X` y otra vez OPEN.

No hacer nunca RESTORE sin RECOVER: el archivo vuelve al SCN de la copia y al ponerlo ONLINE da ORA-01113.

## 4. Cómo saber que ha funcionado

```sql
SELECT name, open_mode FROM v$pdbs;                         -- FREEPDB1 en READ WRITE
SELECT file#, status FROM v$datafile WHERE file# = N;      -- ONLINE
```

- Consultar una tabla del tablespace afectado y comprobar que responde.
- Comprobar un cambio hecho después de la copia (en el simulacro, la fila 104 en EGRESADO). Si está, RECOVER aplicó el redo y el RPO es 0.
- Pasar el runbook de salud: `bash scripts/ops/ro.sh scripts/lab5/15-healthcheck.sql`.

No dar el incidente por cerrado hasta que `v$pdbs` diga READ WRITE.

## 5. Después

- Avisar al responsable de la base y al equipo de la aplicación afectada (en el lab, academia) de cuánto tiempo estuvo caído: desde el ORA-63999 o el OFFLINE hasta el ONLINE/OPEN.
- Hacer una copia RMAN nueva del tablespace, porque la anterior ya se ha usado.
- Averiguar por qué desapareció el archivo (disco, alguien con `rm`...) y apuntarlo en la retrospectiva.
