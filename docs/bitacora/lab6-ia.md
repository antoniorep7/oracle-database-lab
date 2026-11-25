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
  - `grep '^ESTUDIANTE'
