# Lab 6 · Respuestas

E040. Las evidencias estan en `docs/bitacora/lab6/evidencia/`.

## Partes A-C

### 1.
En el turno salieron **4** errores ORA- en 24 h (`script/20261125T175059Z_01-turno.script.log`). Mirando la validacion del arranque (`script/20261125T175801Z_02-estados-pdb.script.log`, sección 4) casi todo es ORA-00800 Set Priority Failed del LGWR, que sale en cada arranque porque en el contenedor no le dejan subir la prioridad. En el ADR es el problema 1 con 40 incidentes desde julio. Los ORA-65019 los provoqué yo en la práctica.
El número solo no es una alarma porque hay que ver que errores son y compararlo con lo normal. En la guardia llegó a 14 y era todo por los reinicios del lab.

### 2.
WEDNESDAY_WINDOW: empieza a las 22:00 y dura 4 horas (`spool/20261125T175159Z_01-automatizacion.spool.log`). Corren auto optimizer stats collection, auto space advisor y sql tuning advisor, las tres ENABLED.

### 3.
En el paso 5 (`script/20261125T175426Z_02-parametros.script.log`) despues del SCOPE = SPFILE el archivo tenia db_files = **250** (ISSPECIFIED TRUE) pero en memoria seguía en **200**. V$SPPARAMETER es lo que pone el spfile y V$PARAMETER lo que se esta usando ahora. Como db_files es estático no cambia hasta reiniciar, así que sin el RESET el siguiente arranque habria cogido 250.

### 4.
db_files no estaba fijado (no sale en `config/20261125T175427Z_02-init_desde_spfile.ora`), asi que RESET lo deja como estaba, con el valor por defecto. processes si esta fijado:

```
*.processes=200
```

Si le haces RESET a processes pierdes ese 200 y arranca con el defecto de Oracle. Ahí habría que volver a poner 200 con SCOPE = SPFILE.

### 5.
C##OPS_RO sacó **ORA-01109: database not open** y FREEPDB1 estaba MOUNTED (sin estado guardado) (`script/20261125T175801Z_02-estados-pdb.script.log`). Pone "Database opened" porque eso es el CDB, la raíz, no la PDB. La PDB sin estado guardado se queda montada.

## Partes D-E

### 6.
SID **218**, SPID **3055**, evento **PL/SQL lock timer** (`script/20261125T180755Z_03-sesiones.script.log`). USERNAME solo dice APP_ACADEMIA, que es la cuenta de toda la aplicación. MODULE (matriculas) y ACTION (cierre-periodo) dicen que parte de la app y que estaba haciendo, y CLIENT_IDENTIFIER (ana.garcia@ucam) quien es la persona de verdad.

### 7.
Tras CANCEL SQL seguía habiendo **1** transacción abierta, la sesión siguió viva ("sigo viva"). Tras KILL SESSION quedaron **0** sesiones de la app. La fila 101 volvio a **ACTIVO** porque al matar la sesión se hace rollback de lo no confirmado (era 1 registro de undo).

### 8.
common_user_prefix = **C##** (`script/20261125T180017Z_03-cuenta-app.script.log`). Las cuentas comunes tienen que empezar por C## (por eso C##OPS_RO) y las locales no pueden. APP_ACADEMIA no lo lleva y sale COMMON = NO.

### 9.
`script/20261125T180949Z_04-tres-miradas.script.log`

| | OPS_RO en root | OPS_RO en freepdb1 | APP_ACADEMIA |
|---|---|---|---|
| ALL_TABLES (ADMIN_ACADEMIA) | 0 | 0 | 1 |
| DBA_USERS | 37 | 47 | ORA-00942 |
| CDB_USERS | 37 (con 1) y 47 (con 3) | 47 | ORA-00942 |
| V$DATAFILE | 17 | 10 | ORA-00942 |

Desde la raíz ve 17 datafiles (raíz + seed + freepdb1) y desde freepdb1 solo los 10 de la PDB, con el mismo usuario. Cambia el contenedor, no los permisos.

### 10.
**5397** lecturas lógicas por recorrido, DB time **0,26 s** y DB CPU **0,25 s** (`script/20261125T181133Z_04-deltas.script.log`). Como son casi iguales casi no hubo esperas, todo fue CPU con los bloques ya en memoria.

## Partes F-G

### 11.
Asignado **10 MB**, libre **3,9 MB**, máximo **40 MB** (`script/20261125T181219Z_05-almacenamiento.script.log`). Alertaria con el % sobre el máximo (15,16 %), porque con autoextend el libre puede ser poco y aun así tener sitio para crecer.

### 12.
Insertó SYS pero la tabla es de CARGA_HIST, y paga el dueño del segmento, que no tenia cuota → ORA-01950. El ORA-01536 era por la cuota de 2 MB del usuario, el tablespace aún tenía sitio, así que ampliarlo no arreglaba nada.

### 13.
De **18:14:03** a **18:14:18**, unos 15 s, casi todo suspendida (a los 12 s seguia SUSPENDED) (`script/20261125T181403Z_05-resumable.script.log`). En el alert log quedó:

```
FREEPDB1(3):statement in resumable session 'carga-historico' was suspended due to
FREEPDB1(3):statement in resumable session 'carga-historico' was resumed
```

### 14.
`script/20261125T181430Z_05-hwm-shrink.script.log`: antes 20000 filas, 2944 bloques, 23 MB. Tras el DELETE 2000 filas pero los mismos 2944 bloques y 23 MB. Tras el SHRINK 304 bloques y 2,4 MB de cuota. El DELETE borra filas pero no baja la HWM, los bloques siguen siendo del segmento.

### 15.
DEFAULT: 10 intentos, 180 dias de vida de contraseña, 7 de gracia, 1 de bloqueo y sin función de verificación (`script/20261125T181524Z_06-gobierno-acceso.script.log`). En PRF_PERSONAS pondría 3 intentos porque una persona no necesita mas, si falla tres veces o la ha olvidado o es alguien probando. Y la función de verificación, que rechazó "corta" con ORA-28003 (menos de 8 caracteres).

### 16.
Porque su rol de secretaría incluye ROL_LECTURA_ACADEMIA, que tiene SELECT ANY TABLE sobre el esquema ADMIN_ACADEMIA, y eso cubre todas las tablas. Leyó **60000** matrículas. Al cambiar el nombre le dio **ORA-41900** missing UPDATE privilege, solo tiene UPDATE en ESTADO (`script/20261125T181527Z_06-probar-acceso.script.log`).

## Partes H-I

### 17.
(`script/20261125T181809Z_07-objetos.script.log`) Columna nueva: nada, porque nadie la usaba. Email: V_CONTACTO_LAB6. Nombres: V_ACTIVOS_LAB6 y la funcion F_TOTAL_ACTIVOS_LAB6 en cascada porque depende de esa vista.

### 18.
No, con la SGA al doble (3072 MB) el DB time factor sigue en 1 y las lecturas físicas igual, 37007. La PGA tendría sobreasignaciones por debajo de **384 MB** (a 256 y 128 sale OVERALLOC 2) (`script/20261125T181812Z_07-memoria.script.log`).

### 19.
**247** lecturas lógicas, lo dice el cr=247 de esta línea (`script/20261125T182119Z_07-adr-trazas.script.log`):

```
FETCH #123676203551280:c=0,e=1067,p=0,cr=247,cu=0,mis=0,r=1,dep=0,og=1,plh=3163684005,tim=6954941684
```

En tkprof sale lo mismo en query.

### 20.
- **INC-601**: Ana (SID 220) esperando en enq TX por el SID 21 con lmode 6 (`182304Z_08-diagnostico`) → KILL SESSION '21,21995' IMMEDIATE → secciones 1-4 vacias y ASH hasta 18:25:01 (`182515Z_08-diagnostico`).
- **INC-602**: academia_app ARRANCADO = NO y no sale en lsnrctl (`182707Z_08-diagnostico`) → 03-servicio.sql, START_SERVICE + SAVE STATE → app.sh conecta y sección 6 = SI (`182818Z_08-validacion-inc602`).
- **INC-603**: APP_ACADEMIA LOCKED (no TIMED) desde las 18:28 (`182900Z_08-diagnostico`) → ALTER USER app_academia ACCOUNT UNLOCK → app.sh conecta y sección 7 vacia (`183126Z_08-validacion-inc603`).

## Preguntas Simulacro

**S1 · Parte B.** ¿Qué proceso genera las alertas de umbral de espacio de los tablespaces?
B. MMON

**S2 · Parte C.** Un parámetro tiene ISSYS_MODIFIABLE = DEFERRED. ¿Qué ocurre con ALTER SYSTEM SET ... DEFERRED SCOPE = BOTH?
C. Se aplica a las sesiones que se conecten a partir de ese momento.

**S3 · Parte C.** ¿Qué sentencia crea una copia de texto de los parámetros en uso, incluidos los calculados por Oracle?
B. CREATE PFILE FROM MEMORY;

**S4 · Parte C.** Una PDB está abierta en READ ONLY. ¿Qué dos sentencias la dejan en READ WRITE?
B. ALTER PLUGGABLE DATABASE pdb OPEN READ WRITE FORCE;
C. ALTER PLUGGABLE DATABASE pdb CLOSE IMMEDIATE; seguida de ALTER PLUGGABLE DATABASE pdb OPEN;

**S5 · Parte C.** ¿Qué vista muestra el estado que se guardó para cada PDB?
C. DBA_PDB_SAVED_STATES

**S6 · Parte C.** Un usuario sin el privilegio RESTRICTED SESSION intenta conectarse a una PDB abierta en modo restringido. ¿Qué obtiene?
B. ORA-01035

**S7 · Parte D.** ¿Qué paquete crea y arranca un servicio en una base sin Grid Infrastructure?
A. DBMS_SERVICE

**S8 · Parte D.** ¿Qué columnas unen V$SESSION con V$PROCESS?
B. PADDR = ADDR

**S9 · Parte D.** ¿Qué ocurre con la transacción pendiente de una sesión al ejecutar ALTER SYSTEM CANCEL SQL sobre ella?
C. Sigue abierta: solo se cancela la llamada en curso.

**S10 · Parte D.** Te conectas como SYSOPER. ¿Qué devuelve SHOW USER?
C. PUBLIC

**S11 · Parte E.** Conectado a una PDB, ¿qué vista te da las métricas del último minuto de esa PDB?
B. V$CON_SYSMETRIC

**S12 · Parte E.** ¿Qué dos componentes suma DB time?
A. Tiempo de CPU en llamadas a la base
C. Tiempo de espera no Idle de las sesiones de primer plano

**S13 · Parte F.** El usuario SYS inserta en la tabla HR.T y recibe ORA-01950 sobre el tablespace USERS. ¿Quién necesita cuota?
B. HR

**S14 · Parte F.** ¿Sobre qué dos tipos de tablespace no se puede conceder cuota (ORA-30041)?
A. Undo
C. Temporal

**S15 · Parte F.** Una sesión con RESUMABLE habilitado se queda sin cuota. ¿Qué dos afirmaciones son ciertas?
A. La sentencia se suspende hasta el timeout.
C. La suspensión se ve en DBA_RESUMABLE.

**S16 · Parte F.** ¿Qué requisito previo tiene ALTER TABLE t SHRINK SPACE?
B. ENABLE ROW MOVEMENT y ASSM.

**S17 · Parte G.** Un perfil tiene FAILED_LOGIN_ATTEMPTS 3 y PASSWORD_LOCK_TIME 1. Tras tres fallos, ¿qué estado muestra DBA_USERS?
B. LOCKED(TIMED)

**S18 · Parte G.** A concedió a B un privilegio de sistema que había recibido WITH ADMIN OPTION. El DBA lo revoca a A. ¿Qué pasa con B?
B. Lo conserva.

**S19 · Parte H.** ¿Qué dos acciones dejan un índice en estado UNUSABLE?
A. ALTER TABLE ... MOVE sin UPDATE INDEXES
B. ALTER INDEX ... UNUSABLE

**S20 · Parte H.** ¿Qué procedimiento activa la traza de la sesión de otro usuario indicando su SID y SERIAL#?
B. DBMS_MONITOR.SESSION_TRACE_ENABLE

**S21 · Parte H.** ¿Qué parámetro fija la raíz del ADR?
C. DIAGNOSTIC_DEST

**S22 · Parte I.** En la guardia, una sesión espera en «enq: TX - row lock contention». ¿Cuál es el primer paso correcto?
C. Identificar al bloqueador (BLOCKING_SESSION) y qué está haciendo.
