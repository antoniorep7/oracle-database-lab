He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 6 · E040

## Entrada 1
- **Herramienta:** Claude
- **Prompt (literal, copiado tal cual):**
```text
Tengo que realizar una tarea de Administración de Base de Datos, quiero que me guies en su realización, ayudandome cuando me atasque o indicandome si he realizado algun fallo. Los archivos que trae la tarea son los siguientes. La carpeta con docs y demás esta en C:\Users\Admin\Downloads\P2_Lab6_v1.3_Estudiante
[adjunté el PDF y el DOCX del enunciado]

Limitate solo a hacer lo que pide la practica, y evita comandos innecesarios o extras

en academia que debo de poner?

como guardo ahora el nano

Dime tu lo que tengo que ponerte exactamente aqui

Que te parece ahora?

Puedes pasarme lo que necesito para responder las 20 preguntas?

Y ahora?

pasame tu la descripcion

[durante el laboratorio le he ido pegando las salidas de los comandos para que las revisara]
[en la Parte I le pasé mis postmortems para que me dijera qué fallaba; los corregí yo]
[le pasé mis respuestas y el simulacro para que me dijera qué fallaba; los corregí yo]
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):** no aplica
- **Dónde lo usé (parte, script o pregunta):** Todo el laboratorio, esta declaración, la revisión de docs/bitacora/lab6-guardia.md, la revisión de docs/bitacora/lab6-respuestas.md y la descripción del Pull Request
- **Propósito (para qué lo necesitaba):** que me guiara en el laboratorio, me dijera si tenía fallos y me explicara cómo corregirlos
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** he ido pasándole los comandos que ejecutaba y sus salidas, y me decía si había algún fallo y cómo corregirlo. También me explicó qué poner en config/.env (las contraseñas de las dos cuentas nuevas y mi código de alumno), cómo guardar en nano, y me dio el texto de esta declaración, que he revisado. En la guardia me explicó cómo leer el diagnóstico y revisó mis postmortems, que escribí y corregí yo. Para las respuestas me indicó en qué evidencia estaba cada dato y revisó mis respuestas y el simulacro; las escribí y corregí yo. También me dio el texto de la descripción del Pull Request, que revisé.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** no aplica

## Entrada 2
- **Herramienta:** Claude
- **Prompt (literal, copiado tal cual):**
```text
[pegué la salida de A.1 con «RESULTADO: 20 FALLO(S)» y después «The command 'docker' could not be found in this WSL 2 distro»]

[pegué el error del cp del kit: «-bash: usuario: No such file or directory»]

Es el de raja

estan aqui C:\Users\Rajaa\Downloads\P2_Lab6_v1.3_Estudiante

[pegué el error de scripts/ops/alumno.sh: «ESTUDIANTE debe ser tu código de la lista de clase»]

[pegué el diagnóstico de 08-diagnostico.sql del incidente INC-601]

[pegué el diagnóstico de 08-diagnostico.sql del incidente INC-603]
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```bash
docker start oralab-26ai
ls /mnt/c/Users/
cp -r /mnt/c/Users/Rajaa/Downloads/P2_Lab6_v1.3_Estudiante/repo-kit/{scripts,config,docs} .
grep '^ESTUDIANTE' config/.env
grep '^ESTUDIANTE' config/.env | cat -A
grep -n 'ESTUDIANTE' config/.env | cat -A
sed -i "s/E0NN/E040/" docs/bitacora/lab6-ia.md
echo "ALTER SYSTEM KILL SESSION '21,21995' IMMEDIATE;" | docker exec -i oralab-26ai sqlplus -s / as sysdba
printf "ALTER SESSION SET CONTAINER = FREEPDB1;\nALTER USER app_academia ACCOUNT UNLOCK;\n" | docker exec -i oralab-26ai sqlplus -s / as sysdba
```
- **Dónde lo usé (parte, script o pregunta):** Parte A (A.1 a A.4), para preparar el entorno, y Parte I (incidentes INC-601 e INC-603)
- **Propósito (para qué lo necesitaba):** arrancar el contenedor, copiar el kit a mi repositorio, revisar mi código de alumno en config/.env, poner mi código en esta declaración y aplicar la intervención de dos incidentes de la guardia
- **Qué hace (con mis palabras; línea a línea si es SQL corto):**
  - `docker start oralab-26ai`: arranca el contenedor de la base, que estaba parado después de reiniciar el ordenador.
  - `ls /mnt/c/Users/`: lista las carpetas de usuario de Windows vistas desde Ubuntu, para encontrar dónde estaba el kit.
  - `cp -r …/repo-kit/{scripts,config,docs} .`: copia las carpetas scripts, config y docs del kit a la raíz de mi repositorio, respetando las rutas.
  - `grep '^ESTUDIANTE' config/.env`: muestra solo la línea de mi código de alumno, sin enseñar las contraseñas.
  - `| cat -A`: muestra además los caracteres invisibles; el `$` marca el final de la línea.
  - `grep -n`: añade el número de línea, para ver si la variable estaba repetida.
  - `sed -i "s/E0NN/E040/"`: cambia E0NN por mi código E040 en el título de este archivo.
  - `ALTER SYSTEM KILL SESSION '21,21995' IMMEDIATE`: termina la sesión 21 de matrículas que bloqueaba a Ana; Oracle deshace su UPDATE sin confirmar y libera la fila (INC-601). El ORA-00031 indica que queda marcada para terminar.
  - `ALTER USER app_academia ACCOUNT UNLOCK` (dentro de FREEPDB1): desbloquea la cuenta de la aplicación que había bloqueado el script de bajas, sin cambiar su contraseña (INC-603).
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):**
  - docker start: cabecera de la evidencia 00 (…_00-backup-lab6-inicio.script.log), con arranque=2026-11-25T16:30:52, la hora en que arrancó el contenedor.
  - cp: el commit de la Parte A, que añade los archivos del kit (scripts/lab6/, scripts/ops/, config/.env.example y docs/plantillas/).
  - grep: login.sql (DEFINE ESTUDIANTE = E040) y alumno=E040 en la cabecera de mis evidencias.
  - sed: el título de este archivo (Lab 6 · E040).
  - KILL: el …_08-diagnostico.script.log de las 18:25:16Z (secciones 1 a 4 vacías; ASH registra la espera hasta las 18:25:01).
  - UNLOCK: el …_08-validacion-inc603.script.log (la aplicación conecta) y el …_08-diagnostico.script.log de las 18:31:29Z (sección 7 vacía).
  - El cp, los grep y el sed se ejecutan en Ubuntu, no en el contenedor; por eso su evidencia es su efecto en el repositorio.

## Entrada 3
- **Herramienta:** Claude
- **Prompt (literal, copiado tal cual):**
```text
He hecho esta tarea en github, quiero que me digas si esta todo correcto, si hay algun rastro de IA que declarar y que notas estimas, este es el repositorio https://github.com/antoniorep7/oracle-database-lab/pulls
[adjunté el PDF del enunciado]

Te paso lo que he hecho yo en la terminal de ubuntu
[adjunté el historial de mi terminal de Ubuntu de todo el laboratorio]

[adjunté la salida de los comandos de reparación de lab6-ia.md]

[pegué la salida de las comprobaciones: git diff --stat, wc -l, md5sum y autoria.py]

Vale, pasamelo para copiar y pegar
[después le pedí el comando de nano y que revisara mi texto y la entrada antes del commit]```
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```bash
git switch main && git pull
git switch -c fix/7-lab6-ia
git show 9f2d2ff:docs/bitacora/lab6-ia.md > /tmp/ia_9f2d2ff.md
head -n 73 docs/bitacora/lab6-ia.md > /tmp/ia_arreglada.md
tail -n +72 /tmp/ia_9f2d2ff.md >> /tmp/ia_arreglada.md
cp /tmp/ia_arreglada.md docs/bitacora/lab6-ia.md
git diff 9f2d2ff -- docs/bitacora/lab6-ia.md
git --no-pager diff --stat 9f2d2ff -- docs/bitacora/lab6-ia.md
wc -l < docs/bitacora/lab6-ia.md
md5sum docs/bitacora/lab6-ia.md
python3 scripts/ops/autoria.py --alumno E040
```
- **Dónde lo usé (parte, script o pregunta):** revisión de toda la entrega antes de la corrección y reparación de docs/bitacora/lab6-ia.md (entrada 2)
- **Propósito (para qué lo necesitaba):** saber si la entrega estaba completa y qué tenía que declarar; recuperar el final de la entrada 2, que se perdió al editar el archivo con nano antes del commit c817937 y dejó vacío el campo «Evidencia de ejecución» (D3)
- **Qué hace (con mis palabras; línea a línea si es SQL corto):**
- Pone main al día, abre la rama fix/7-lab6-ia y recompone lab6-ia.md juntando su cabecera actual con el resto que tenía en el commit 9f2d2ff.
Luego revisa qué ha cambiado en el fichero y pasa el script de autoría del alumno E040.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** no aplica en el contenedor: los comandos se ejecutan en Ubuntu sobre el repositorio. Su efecto se ve en el propio lab6-ia.md: antes de añadir esta entrada, git diff --stat contra 9f2d2ff daba 4 inserciones y 2 borrados (solo mis cambios sobre el PR), el archivo tenía 87 líneas con md5 9bf8d2fc56bacf964b6fa2a1d5917bdc y autoria.py ya no daba el fallo D3 de la entrada 2.
