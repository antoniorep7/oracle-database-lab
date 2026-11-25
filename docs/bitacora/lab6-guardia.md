## INC-601 · La pantalla de secretaría se queda colgada
- Detección: 18:22 UTC · ticket de Ana García (secretaría)
- Impacto: Ana no podía guardar la ficha del alumno 101; esperó desde las 18:22 hasta las 18:25 UTC
- Diagnóstico: 08-diagnostico.sql: SID 220 (ANA_GARCIA) en «enq: TX - row lock contention», BLOQ = 21; SID 21 = APP_ACADEMIA (matriculas / cierre-periodo) con el TX en lmode 6 y 1 registro de undo
- Causa raíz: El cierre de periodo del módulo de matrículas lanzó un UPDATE y se quedó parado sin hacer COMMIT ni ROLLBACK. Mientras esa transacción siga abierta, la fila queda bloqueada en modo 6 y cualquier otra sesión que intente tocarla se queda esperando, que es lo que le pasó a Ana.
- Intervención: ALTER SYSTEM KILL SESSION '21,21995' IMMEDIATE. Era la más pequeña porque un CANCEL SQL no servía: la sesión no estaba ejecutando nada, solo tenía la transacción abierta, y cancelar la sentencia no suelta el bloqueo. Matando la sesión 21 se hace rollback de lo que tenía pendiente (1 registro de undo) y la fila queda libre. La sesión de Ana sigue viva y no hay que reiniciar la instancia.
- Validación: 08-diagnostico.sql tras la intervención: secciones 1 a 4 vacías; ASH registra la espera hasta las 18:25:01
- Acción preventiva: Revisar el código de cierre-periodo para que toda transacción termine con COMMIT o ROLLBACK, también en los caminos de error. Aparte, poner una alerta que salte cuando una sesión lleve un rato inactiva con una transacción abierta o bloqueando a otra.

## INC-602 · La aplicación no conecta: ORA-12514
- Detección: 18:26 UTC · ticket del equipo de matrículas
- Impacto: ninguna instancia de la aplicación podía conectar por academia_app, desde las 18:26 hasta las 18:28 UTC
- Diagnóstico: turno.sql: instancia OPEN, FREEPDB1 READ WRITE; 08-diagnostico.sql sección 6: academia_app ARRANCADO = NO; lsnrctl status no lo anuncia
- Causa raíz: El servicio academia_app estaba parado y nadie lo vigilaba. No hay ninguna comprobación que avise si el servicio cae o deja de estar registrado en el listener, así que nos enteramos cuando la aplicación empezó a fallar y llegó el ticket.
- Intervención: 03-servicio.sql → DBMS_SERVICE.START_SERVICE('academia_app') y ALTER PLUGGABLE DATABASE freepdb1 SAVE STATE (18:28 UTC). Era la más pequeña porque la instancia estaba abierta y la PDB ya estaba en READ WRITE, así que no hacía falta reiniciar nada: lo único que fallaba era el servicio. Con START_SERVICE academia_app vuelve a estar levantado, el listener lo anuncia otra vez y la aplicación puede conectar en ese momento. Con SAVE STATE se guarda cómo está ahora la PDB (abierta y con academia_app arrancado), de modo que si se reinicia el CDB vuelve a quedar igual y nadie tiene que acordarse de arrancar el servicio a mano.
- Validación: app.sh con 06-app-sigue.sql conecta por academia_app; sección 6 = SI
- Acción preventiva: Añadir al chequeo de turno una comprobación de que academia_app está arrancado y aparece en lsnrctl status, con aviso si no. El SAVE STATE ya cubre los reinicios del CDB, pero conviene comprobarlo en el próximo reinicio planificado.

## INC-603 · La aplicación no entra: ORA-28000
- Detección: 18:28 UTC · ticket del equipo de matrículas; nota: hoy se ejecutó el script trimestral de bajas
- Impacto: la aplicación no podía iniciar sesión tras el despliegue, desde las 18:28 hasta las 18:31 UTC
- Diagnóstico: 08-diagnostico.sql sección 7: APP_ACADEMIA LOCKED (no LOCKED(TIMED)) a las 18:28, perfil DEFAULT con 10 intentos
- Causa raíz: El script trimestral de bajas bloquea cuentas sin distinguir entre usuarios de personas y usuarios de aplicación. No tiene lista de exclusiones ni mira el perfil, así que se llevó por delante a APP_ACADEMIA.
- Intervención: ALTER USER app_academia ACCOUNT UNLOCK; (18:31 UTC). Era la más pequeña porque el estado era LOCKED y no LOCKED(TIMED): la cuenta la había bloqueado alguien a propósito, no se bloqueó por intentos fallidos de contraseña. No había motivo para pensar que la contraseña estuviera comprometida y el usuario conservaba sus privilegios, así que bastaba con desbloquearlo. No hacía falta cambiar la contraseña, recrear el usuario ni reiniciar nada.
- Validación: app.sh con 06-app-sigue.sql conecta; sección 7 vacía
- Acción preventiva: Cambiar el script de bajas para que nunca toque las cuentas de aplicación, ya sea con una lista de exclusiones o dándoles un perfil propio y filtrando por él. Antes de bloquear, que el script saque la lista de cuentas afectadas para revisarla.
