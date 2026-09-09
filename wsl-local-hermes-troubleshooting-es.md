# Corriendo Hermes Agent en local con WSL2 (Windows) — Problemas reales que viví y cómo los resolví

Comparto esto porque construir un agente de IA con Hermes de forma local, en vez de un VPS de pago, me ahorró dinero pero me costó bastantes horas de diagnóstico que quizás le ahorren tiempo a alguien más. Aquí está todo lo que me trabó, y la solución real de cada cosa.

## Mi configuración, en un párrafo

Laptop con Windows 11, corriendo **WSL2 con Ubuntu 24.04** en vez de un VPS de pago. Dentro de Ubuntu, creé un **usuario dedicado, sin privilegios de administrador, llamado `hermes`** (sin acceso a `sudo`, a propósito — mínimo privilegio necesario) para correr el agente, separado de mi usuario normal de Windows/Ubuntu (le voy a llamar "usuario administrador" más abajo — es la cuenta que usas día a día, la que sí tiene `sudo`). Los archivos del agente viven en `~/workspace`, dentro de la cuenta `hermes`. Si tu configuración es distinta, las causas de fondo que menciono abajo probablemente igual apliquen — es sobre todo comportamiento de WSL2 y permisos de Linux, no algo exclusivo de Hermes.

---

## Instalación y sistema base

### El PATH no encuentra el comando `hermes` después de instalarlo

**Síntoma:** `hermes --version` devuelve `command not found` justo después de correr el script de instalación.

**Causa:** el instalador coloca el ejecutable en `~/.hermes/hermes-agent/venv/bin/`, pero esa ruta no siempre se agrega automáticamente al `PATH` de tu terminal.

**Solución:**
```bash
echo 'export PATH="$HOME/.hermes/hermes-agent/venv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

### WSL se suspende y todos los servicios en segundo plano mueren

**Síntoma:** el dashboard y el bot de Telegram dejan de responder de la nada, aunque la laptop esté prendida.

**Causa — en realidad son dos problemas distintos, uno encima del otro:**
1. **Windows** suspende la laptop tras un rato de inactividad, incluso conectada a la corriente.
2. Independientemente de eso, **WSL2 apaga su propia máquina virtual ligera** unos 15-20 segundos después de cerrar la última ventana de terminal abierta — un comportamiento completamente separado de la suspensión de Windows.

**Solución (necesitas ambas partes):**
```powershell
# En PowerShell, como administrador — evita que la laptop se suspenda mientras está enchufada
powercfg /change standby-timeout-ac 0
```
Y una tarea en el **Programador de tareas de Windows**, disparada "al iniciar sesión", corriendo en segundo plano:
```
Programa: wsl.exe
Argumentos: -d Ubuntu --exec sleep infinity
```
Esto mantiene la VM de WSL viva sin necesidad de ninguna ventana visible.

---

### Este es el de los permisos — archivos creados desde el Explorador de Windows no son accesibles para el usuario del agente

**Síntoma:** el usuario `hermes` no puede leer/escribir archivos dentro de su propia carpeta de trabajo, cuando esos archivos se crearon arrastrándolos desde el Explorador de Windows.

**Causa:** un archivo creado desde el Explorador de Windows (o tu sesión normal de Windows/Ubuntu) queda con **ese** usuario como dueño, no con `hermes`. En Linux, ser dueño de un archivo y los permisos son cosas separadas de "en qué carpeta está físicamente" — que un archivo esté dentro del workspace de `hermes` no significa automáticamente que `hermes` sea su dueño o pueda tocarlo.

**Solución puntual** (dar acceso sin romper el acceso del otro usuario):
```bash
# Corre esto como el usuario que realmente es dueño del archivo/carpeta
chmod o+rx /home/hermes/workspace/alguna-carpeta
chmod o+rw /home/hermes/workspace/alguna-carpeta/*.md
```

**Solución permanente** (para que las carpetas nuevas hereden el permiso correcto automáticamente):
```bash
# Como hermes, sobre una carpeta que hermes ya es dueño
setfacl -d -m other:rwx ~/workspace/alguna-carpeta-padre
```

Esto fue, por mucho, lo que más tiempo me consumió — no es nada obvio si eres nuevo en permisos de Linux, y los mensajes de error no te dicen "esto es un problema de dueño del archivo, no de código".

---

## Obsidian (si lo usas como ventana a los archivos del agente)

### Error `EISDIR` al abrir el vault desde Windows

**Síntoma:** Obsidian (instalado en Windows) muestra `Error EISDIR: illegal operation on a directory, watch \\wsl.localhost\...` al intentar abrir la carpeta de WSL como vault.

**Causa:** es un bug conocido y de larga data en Obsidian — puede *listar* el contenido de una carpeta a través de una ruta de red de Windows (`\\wsl.localhost\...`) sin problema, pero no puede *vigilar cambios* en ella de forma confiable, que es justo lo que necesita para abrirla como vault.

**Solución real:** instalar la **versión de Obsidian para Linux** dentro de la propia WSL (WSLg permite que las apps gráficas de Linux se muestren como ventanas normales de Windows), para que acceda a la carpeta de forma nativa, sin cruzar la red:
```bash
# Como tu usuario administrador (no el usuario dedicado del agente), ya que apt install necesita sudo
deb_link=$(curl -s https://api.github.com/repos/obsidianmd/obsidian-releases/releases | grep "browser_download_url.*amd64.deb" | head -1 | cut -d '"' -f 4)
curl -L -o ~/obsidian.deb "$deb_link"
sudo apt install -y ~/obsidian.deb
obsidian &
```
Luego abre el vault usando la ruta nativa de Linux (ej. `/home/hermes/workspace`), no la ruta de red de Windows.

---

### El proceso de Obsidian corre, pero no aparece ninguna ventana (WSLg)

**Síntoma:** `ps aux` muestra a Obsidian corriendo, pero no aparece nada en el escritorio de Windows.

**Causa:** el motor gráfico de WSLg no arrancó correctamente esa sesión.

**Solución:**
```powershell
wsl --shutdown
```
Espera ~20 segundos, abre una terminal nueva, y prueba primero con algo ligero (`sudo apt install -y x11-apps && xeyes`) antes de reintentar con Obsidian.

---

## Modelos y control de costos

### Un modelo específico no está disponible con tu conexión OAuth de ChatGPT

**Síntoma:** `HTTP 400: model is not supported when using Codex with a ChatGPT account`.

**Causa:** algunos modelos de una familia no están disponibles vía login OAuth, solo con una API key de pago directo.

**Solución:** cambia a un modelo hermano que sí esté disponible en ese mismo método de conexión (revisa las opciones con `/model` dentro de un chat).

---

### Se agota la cuota mensual del plan gratis

**Síntoma:** el modelo deja de responder; el panel de uso muestra 0% restante, y se restablece en semanas, no en horas.

**Causa:** algunos planes gratis limitan el uso **mensualmente**, a diferencia de los planes de pago, que suelen usar ventanas móviles cortas (mucho más difíciles de agotar por completo).

**Solución:** configura un `fallback_providers` que apunte a un modelo de pago genuinamente distinto y barato — centavos al día para uso ligero, y sin la saturación de la capa gratuita.

---

### Tu modelo de "respaldo" no sirve de nada

**Síntoma:** el log muestra `Fallback skip: chain entry ... resolves to the same backend as the current one`.

**Causa:** el modelo principal y el de respaldo eran exactamente el mismo.

**Solución:** el respaldo debe ser un modelo/proveedor genuinamente distinto:
```yaml
model:
  provider: openrouter
  default: <modelo-principal>
fallback_providers:
  - provider: openrouter
    model: <un-modelo-distinto-de-otra-compañía>
```

---

### Un modelo expone su razonamiento interno en el chat

**Síntoma:** de pronto ves texto en tercera persona, tipo "pensando en voz alta", en vez de una respuesta normal.

**Causa:** bug documentado en algunos modelos (a mí me pasó específicamente con M2.7 y M3 de MiniMax), donde el razonamiento interno no se oculta de forma confiable del resultado final.

**Solución:** cambiar a otra familia de modelo que no tenga ese bug.

---

### Un modelo gratis se retira sin previo aviso

**Síntoma:** `HTTP 404: This model is unavailable for free. The paid version is available now`.

**Causa:** los catálogos de modelos gratis (en mi caso, OpenRouter) cambian seguido — un modelo gratis hoy puede ser exclusivamente de pago mañana, sin aviso.

**Solución:** siempre revisa la lista *vigente* de modelos gratis antes de configurar uno, en vez de confiar en información de incluso un día atrás.

---

### Todo se vuelve lento/limitado a la vez, en varios modelos gratis sin relación entre sí

**Causa:** esto me pasó un día en que varios de los grandes proveedores de IA de pago tuvieron una caída simultánea — el tráfico se desbordó hacia las alternativas gratis, saturando la capacidad compartida.

**Solución:** en este caso específico, solo espera — es un pico de carga externo, no tu configuración. Un modelo *de pago*, aunque sea muy barato, no compite por esa misma capacidad gratuita saturada.

---

## Automatización (cron) y memoria entre sesiones

### Una tarea programada se salta sola después de cambiar la configuración del modelo

**Síntoma:** `RuntimeError: Skipped to prevent unintended spend: global inference config drifted since this job was created...`

**Causa:** es una protección intencional — si el modelo cambia después de crear la tarea, el agente prefiere detenerse antes que arriesgarse a un gasto no revisado.

**Solución:**
```bash
hermes cron edit <job_id> --provider <proveedor_actual> --model <modelo_actual>
```

---

### Las sesiones de chat no "recuerdan" lo que hizo una ejecución programada

**Síntoma:** en una conversación normal, el agente niega haber hecho algo que sí ocurrió durante una ejecución automática esa misma mañana.

**Causa:** cada ejecución programada corre en su propia sesión, separada de las conversaciones de chat normales — no comparten memoria de conversación automáticamente (aunque sí comparten los mismos archivos en disco).

**Solución:** agrega una regla permanente al archivo de instrucciones que se cargue siempre (en mi caso, `AGENTS.md`):
```markdown
Las ejecuciones programadas (cron) corren en sesiones separadas de las conversaciones de chat normales. Si el usuario menciona algo que ya sucedió y no lo reconoces en tu contexto actual, nunca lo niegues de inmediato — revisa los archivos relevantes en disco primero.
```

---

### Una tarea programada inventa un resultado que no existe

**Síntoma:** una búsqueda automática presentó un resultado específico y creíble (nombre, cifras, un link) — pero al investigar más a fondo, el propio agente admitió: "no encuentro evidencia de que esto exista... parece ser un error de mi parte".

**Causa:** el modelo generó algo que "sonaba" plausible en vez de admitir que no encontró nada, probablemente por la presión implícita de "debo tener resultados que mostrar".

**Solución:** agrega una regla explícita a la skill correspondiente:
```markdown
- Nunca presentes un resultado sin haber verificado que la fuente carga contenido real en esa búsqueda específica. Si no puedes verificarlo, exclúyelo o márcalo explícitamente como "no verificado" — nunca lo presentes con datos completos como si fuera confiable.
```

---

### Enlaces rotos se presentan como si fueran válidos

**Síntoma:** un enlace que devolvía error 403/404 se incluyó de todos modos, con una nota tipo "vale la pena confirmar" en letra pequeña.

**Causa:** no había ninguna regla que obligara a descartar (no solo "avisar") una fuente que no carga.

**Solución:** la misma regla de la entrada anterior — un fallo de carga es motivo de exclusión automática, no una nota al margen.

---

### El agente confunde dos resultados distintos de la misma fuente

**Síntoma:** dos publicaciones genuinamente distintas de la misma empresa se mezclaron, atribuyéndole los datos de una a la otra.

**Causa:** la comparación solo se hacía contra el registro histórico de "ya visto", nunca **entre** los resultados nuevos del mismo lote.

**Solución:**
```markdown
- Antes de presentar un lote de resultados nuevos, compáralos también ENTRE SÍ (no solo contra el registro histórico), para detectar duplicados o datos mezclados entre entradas similares.
```

---

### Los mensajes de "voy a hacer..." persisten aunque `tool_progress` esté en `off`

**Síntoma:** el agente seguía narrando cada paso en el chat, a pesar de tener correctamente configurado `display.tool_progress: off` en `config.yaml`.

**Causa:** ese ajuste solo controla los mensajes **automáticos del sistema** sobre actividad de herramientas — no evita que el propio modelo, por su cuenta, decida escribir texto narrando su plan como parte de su respuesta normal.

**Solución:** una instrucción explícita en `SOUL.md` (no en `config.yaml`):
```markdown
## Comunicación Durante Tareas de Varios Pasos
Cuando realices una tarea con varios pasos, no envíes mensajes intermedios narrando lo que vas a hacer. Trabaja en silencio a través de todos los pasos necesarios y responde una sola vez con el resultado final completo, a menos que genuinamente necesites hacer una pregunta para continuar.
```

---

### Una skill no se carga de forma confiable en ejecuciones programadas (cron)

**Síntoma:** una tarea programada evaluó cosas sin usar reglas/datos que sí estaban en una skill instalada, aunque el prompt de la tarea la mencionaba por nombre.

**Causa:** cada ejecución de cron corre en una sesión nueva y aislada — **mencionar** una skill en el texto del prompt no garantiza que se cargue por completo, ya que eso sigue dependiendo del juicio del propio modelo.

**Solución real:** adjunta la skill formalmente al trabajo, no solo la menciones en el texto:
```bash
hermes cron edit <job_id> --skill <nombre-de-la-skill>
```
Verifica con `hermes cron list` — debe mostrar un campo `Skills:` en el trabajo, no solo una mención dentro del prompt. Esto garantiza que la skill se cargue completa en cada ejecución, sin depender del criterio del modelo.

---

### Un modelo económico "olvida" partes de una skill

**Síntoma:** con un modelo barato, el agente evaluaba tareas sin usar datos que sí estaban en la referencia rápida de una skill instalada.

**Causa:** las skills se cargan en dos pasos — un disparador decide si el contenido entra al contexto, pero el propio modelo debe luego "darse cuenta" de usar cada detalle mientras razona. Los modelos más pequeños/baratos son menos consistentes en este segundo paso, especialmente en conversaciones largas.

**Solución de dos partes:**
1. Duplica los datos más críticos y estables en tu archivo de "quién eres" para el agente (se carga siempre, sin depender del disparador de ninguna skill)
2. Invoca la skill explícitamente con su comando, en vez de solo mencionarla dentro de una pregunta general, para forzar la carga determinista de su contenido completo

---

### Una tarea programada posiblemente se ejecuta dos veces

**Síntoma:** la misma tarea programada entregó dos respuestas separadas, con contenido distinto, el mismo día, sin que se disparara manualmente dos veces.

**Causa probable:** dos procesos del gateway corriendo al mismo tiempo (cada uno con su propio reloj interno independiente), algo que puede pasar tras un reinicio abrupto o una sesión de terminal que se cerró de forma inesperada.

**Diagnóstico, si vuelve a pasar (correrlo en el momento, no después):**
```bash
ps aux | grep hermes-agent | grep -v grep
```
Si aparece más de un proceso con `gateway` en el comando, ahí está la causa:
```bash
hermes gateway restart
```
Nota honesta: para cuando investigué el mío, ya no había ningún proceso duplicado (probablemente se limpió solo con un reinicio posterior de WSL) — así que no pude confirmar la causa exacta con certeza, solo la más probable según la documentación oficial.

---

## Mantener confiable la tarea de Windows que mantiene WSL vivo

### Un disparador "Al iniciar sesión" deja de activarse después de unos días

**Síntoma:** WSL dejó de arrancar solo después de aproximadamente una semana, aunque la laptop se usaba a diario — el "Último resultado de ejecución" en el Programador de tareas mostraba una fecha de más de una semana atrás.

**Causa:** un disparador de tipo **"Al iniciar sesión"** solo se activa con un inicio de sesión genuino de Windows (usuario + contraseña desde cero) — si tu rutina diaria es solo desbloquear la pantalla (sin cerrar sesión), ese disparador nunca se vuelve a activar. Y si WSL se cae a mitad del día por cualquier otro motivo (ver la entrada de suspensión más arriba), nada lo vuelve a levantar hasta el siguiente inicio de sesión real.

**Solución:** haz que la tarea se "auto-repare" sola, en vez de depender de un único disparo:
1. Abre el Programador de tareas (`taskschd.msc`) → propiedades de la tarea → pestaña **Desencadenadores**
2. Edita el disparador existente → en **Configuración avanzada**, activa **"Repetir la tarea cada: 5 minutos"**
3. Cambia **"durante un periodo de:"** a **"Indefinidamente"**

Esto es seguro — si WSL ya está corriendo cuando la tarea se repite, no crea nada nuevo ni duplica procesos, solo confirma que sigue vivo.

---

## Integraciones externas (usé Composio para Gmail/Drive/Docs)

### Una conexión rápida otorga mucho más acceso del que necesitas

**Síntoma:** una conexión sencilla de Gmail con un solo clic terminó otorgando 61 herramientas distintas, incluyendo borrado masivo permanente sin posibilidad de recuperación.

**Causa:** el flujo de conexión rápida usa un scope amplio de OAuth por defecto, no uno restringido a solo lectura.

**Lo que intenté (parcialmente bloqueado):** crear una configuración de autenticación personalizada, restringida por scope — Google la bloqueó directamente ("Se bloqueó esta app"), porque ese scope específico requiere que la aplicación conectora pase por su propio proceso de verificación de seguridad, algo que la app compartida/administrada de Composio no tiene aprobado para esa combinación. Resolverlo del todo implica crear tu propio proyecto de Google Cloud con tus propias credenciales OAuth — un proyecto aparte en sí mismo.

**Lo que hice mientras tanto:** listar explícitamente, por nombre, las herramientas peligrosas como prohibidas en el archivo de instrucciones principal del agente:
```markdown
## Herramientas de Gmail Prohibidas
Nunca uses estas bajo ninguna circunstancia: GMAIL_BATCH_DELETE_MESSAGES, GMAIL_DELETE_MESSAGE, GMAIL_DELETE_DRAFT, GMAIL_TRASH_MESSAGE, GMAIL_MOVE_TO_TRASH, GMAIL_CREATE_FILTER, GMAIL_MODIFY_LABELS, GMAIL_BATCH_MODIFY_MESSAGES.
```
No es tan fuerte como una restricción real a nivel de permisos, pero sí es una salvaguarda real y funcional mientras la solución "correcta" queda pendiente.

---

## Fundamentos de terminal que me trabaron (si eres nuevo en Linux)

### La terminal se queda esperando para siempre después de `cat ... << EOF`

**Síntoma:** al pegar un bloque largo de texto para crear un archivo, la terminal nunca regresa al prompt normal después de escribir `EOF`.

**Causa:** pegar bloques largos a veces introduce un carácter invisible o un salto de línea que evita que `EOF` quede solo en su propia línea, que es el requisito exacto para que se reconozca.

**Solución:** `Ctrl + C` para salir, y usar `nano` para bloques largos pegados en su lugar — es más tolerante, siempre que confirmes que de verdad estás escribiendo/pegando en la terminal y no en algún otro campo de texto en pantalla. Para bloques *muy* largos, a veces ni `nano` lo tolera bien — en ese caso, es más confiable descargar el archivo completo y reemplazarlo directamente vía el Explorador de Windows (`\\wsl.localhost\...`), sin pasar por copiar/pegar en la terminal en absoluto.

---

Si te está costando trabajo específicamente la parte de permisos, esa fue también mi mayor consumo de tiempo. Con gusto respondo dudas en los comentarios si algo de esto no coincide exactamente con lo que estás viendo.
