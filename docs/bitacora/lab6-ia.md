He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 6 · E040

<!--
CÓMO SE RELLENA (léelo y borra este comentario si quieres; el script de evaluación lo ignora)
- La PRIMERA línea es obligatoria y no se toca: «He leído la política de uso de IA de la asignatura.»
- Copia este archivo como docs/bitacora/lab6-ia.md (junto a lab6-respuestas.md) y pon tu código en el título.
  Cada laboratorio tiene el suyo: en el Lab 7 será docs/bitacora/lab7-ia.md.
- Si NO has usado IA: deja solo la línea «No he usado IA.» y borra la entrada de ejemplo.
- Si la has usado: borra la línea «No he usado IA.» y escribe UNA entrada por cada uso, con los 7 campos.
  Un campo vacío es una entrada incompleta (D3: −0,5 por entrada).
- Preguntar a la IA como tutor o corregir la ortografía de un texto propio está permitido: decláralo también.
  Si no copiaste código, escribe «no aplica» en «Código recibido» y en «Evidencia de ejecución» (no cuentan como vacíos).
  Los demás campos son siempre obligatorios.
- Una entrada con TODOS los campos vacíos (la de ejemplo sin tocar) no cuenta: se ignora.
- El código copiado de una IA puntúa si lo EJECUTAS en tu contenedor, lo EXPLICAS con tus palabras,
  dices para qué lo usaste y adjuntas aquí el código y el prompt LITERAL.
-->


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

[durante el laboratorio le he ido pegando las salidas de los comandos para que las revisara]
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):** no aplica
- **Dónde lo usé (parte, script o pregunta):** Todo el laboratorio, y esta declaración
- **Propósito (para qué lo necesitaba):** que me guiara en el laboratorio, me dijera si tenía fallos y me explicara cómo corregirlos
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** he ido pasándole los comandos que ejecutaba y sus salidas, y me decía si había algún fallo y cómo corregirlo. También me explicó qué poner en config/.env (las contraseñas de las dos cuentas nuevas y mi código de alumno), cómo guardar en nano, y me dio el texto de esta declaración, que he revisado.
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
```
- **Dónde lo usé (parte, script o pregunta):** Parte A (A.1 a A.4), para preparar el entorno
- **Propósito (para qué lo necesitaba):** arrancar el contenedor, copiar el kit a mi repositorio, revisar mi código de alumno en config/.env y poner mi código en esta declaración
- **Qué hace (con mis palabras; línea a línea si es SQL corto):**
  - `docker start oralab-26ai`: arranca el contenedor de la base, que estaba parado después de reiniciar el ordenador.
  - `ls /mnt/c/Users/`: lista las carpetas de usuario de Windows vistas desde Ubuntu, para encontrar dónde estaba el kit.
  - `cp -r …/repo-kit/{scripts,config,docs} .`: copia las carpetas scripts, config y docs del kit a la raíz de mi repositorio, respetando las rutas.
  - `grep '^ESTUDIANTE' config/.env`: muestra solo la línea de mi código de alumno, sin enseñar las contraseñas.
  - `| cat -A`: muestra además los caracteres invisibles; el `$` marca el final de la línea.
  - `grep -n`: añade el número de línea, para ver si la variable estaba repetida.
  - `sed -i "s/E0NN/E040/"`: cambia E0NN por mi código E040 en el título de este archivo.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):**
  - docker start: cabecera de la evidencia 00 (…_00-backup-lab6-inicio.script.log), con arranque=2026-11-25T16:30:52, la hora en que arrancó el contenedor.
  - cp: el commit de la Parte A, que añade los archivos del kit (scripts/lab6/, scripts/ops/, config/.env.example y docs/plantillas/).
  - grep: login.sql (DEFINE ESTUDIANTE = E040) y alumno=E040 en la cabecera de mis evidencias.
  - sed: el título de este archivo (Lab 6 · E040).
  - El cp, los grep y el sed se ejecutan en Ubuntu, no en el contenedor; por eso su evidencia es su efecto en el repositorio. 
