# Runbook: se pierde un datafile

Esto es lo que hay que hacer si desaparece o se rompe el datafile de un tablespace de negocio (no de SYSTEM). Lo escribí después del simulacro D del Lab 5, donde se borró academia01.dbf (file 16) de FREEPDB1. A mí la recuperación me tardó 8 segundos y no se perdió nada.

## Cómo se ve

Las consultas a ese tablespace fallan con tres errores a la vez:

- ORA-01116: no puede abrir el archivo N.
- ORA-01110: dice qué archivo es, con la ruta completa.
- ORA-27041 con "Linux-x86_64 Error: 2: No such file or directory": es la causa a nivel de sistema operativo, el archivo no está.

Si en el alert log también sale ORA-63999 (data file suffered media failure), es que Oracle ya ha cerrado la PDB entera y está en MOUNTED. La CDB y las demás PDB siguen funcionando normal. A mí me pasó esto.

## Antes de tocar nada

Entrar como SYSDBA y mirar:

```sql
SELECT log_mode FROM v$database;
SELECT name, open_mode FROM v$pdbs;
```

Y en RMAN (`docker exec -it oralab-26ai rman target /`):

```
LIST BACKUP OF DATAFILE N;
```

Tiene que estar en ARCHIVELOG y tiene que haber una copia del archivo. Si la base está en NOARCHIVELOG esto no sirve, lo único que queda es restaurar la última copia en frío con restaurar-estado.sh y se pierde todo lo de después, así que hay que avisar antes. Si no hay copia RMAN, parar y avisar. Si el archivo es de SYSTEM o del undo de la raíz tampoco vale, ahí hay que hacer SHUTDOWN ABORT, STARTUP MOUNT, RESTORE, RECOVER y OPEN.

Si la PDB se está cerrando en ese momento hay que esperar a que salga MOUNTED en v$pdbs. Si lanzas el RESTORE a mitad del cierre falla con ORA-01089.

## Arreglarlo

Si la PDB sigue en READ WRITE (solo se ha caído ese tablespace):

```
SQL>  ALTER SESSION SET CONTAINER = FREEPDB1;
SQL>  ALTER DATABASE DATAFILE N OFFLINE;
RMAN> RESTORE DATAFILE N;
RMAN> RECOVER DATAFILE N;
SQL>  ALTER DATABASE DATAFILE N ONLINE;
```

El ONLINE se hace otra vez dentro de FREEPDB1. Si el OFFLINE da ORA-01089 es que Oracle está cerrando la PDB justo entonces, y se sigue como en el otro caso.

Si la PDB ya está en MOUNTED:

```
RMAN> RESTORE DATAFILE N;
RMAN> RECOVER DATAFILE N;
SQL>  ALTER SESSION SET CONTAINER = FREEPDB1;
SQL>  ALTER DATABASE DATAFILE N ONLINE;
SQL>  ALTER SESSION SET CONTAINER = CDB$ROOT;
SQL>  ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;
```

Al abrir la PDB me salió ORA-01113 (file 21 needs media recovery) en otro archivo de la misma PDB, tiendasur01.dbf. Es por el cierre brusco. Oracle lo recuperó solo con el redo y abrió sin hacer nada más. Si no abriera, habría que hacer RECOVER DATAFILE de ese archivo y volver a intentar el OPEN.

Importante: no hacer RESTORE sin RECOVER. El archivo vuelve al SCN de la copia y al ponerlo ONLINE da ORA-01113.

## Comprobar que ha funcionado

```sql
SELECT name, open_mode FROM v$pdbs;
SELECT file#, status FROM v$datafile WHERE file# = N;
```

FREEPDB1 tiene que estar en READ WRITE y el archivo ONLINE. Después consultar alguna tabla de ese tablespace y, si se puede, algún dato que se cambiara después de la copia (en el simulacro era la fila 104 en EGRESADO). Si está, el RECOVER ha aplicado el redo y no se ha perdido nada. Por último pasar el runbook de salud: `bash scripts/ops/ro.sh scripts/lab5/15-healthcheck.sql`.

Hasta que v$pdbs no diga READ WRITE el incidente no está cerrado.

## Después

Avisar al responsable de la base y a los de la aplicación que usa ese tablespace (en el lab, academia) de cuánto tiempo ha estado caído, desde el ORA-63999 o el OFFLINE hasta que vuelve a estar ONLINE. Hacer una copia RMAN nueva del tablespace y apuntar en la retrospectiva por qué desapareció el archivo.
