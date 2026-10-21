<div align="center">
<sub>🇬🇧 <a href="./README.md">English</a> &nbsp;|&nbsp; 🇨🇳 <a href="./README.zh-CN.md">中文</a> &nbsp;|&nbsp; 🇪🇸 <a href="./README.es.md">Español</a> &nbsp;|&nbsp; 🇯🇵 <a href="./README.ja.md">日本語</a> &nbsp;|&nbsp; 🇧🇷 <a href="./README.pt.md">Português</a></sub>
</div>

<br/>

<div align="center">

<img src="logo.png" alt="Perch" width="120">

# Perch

**Un gestor de procesos y servicios para Windows, ligero y de un solo binario, con web UI integrada.**

Escrito en Rust. Sin dependencias de runtime. Un binario, doble clic y listo.

</div>

---

## Capturas de pantalla

### Menú interactivo (doble clic en `perch.exe`)

![menu](docs/screenshots/menu.png)

### Inicio de sesión

![login](docs/screenshots/login.png)

### Panel de procesos

![processes](docs/screenshots/processes.png)

### Detalle del proceso — Transmisión de logs en vivo

![process-detail](docs/screenshots/process-detail.png)

### Tareas programadas

![tasks](docs/screenshots/tasks.png)

---

## ¿Por qué Perch?

Gestionar procesos en segundo plano y servicios de larga duración en Windows es más difícil de lo que debería ser:

- **Task Scheduler** termina por defecto los procesos de larga duración (tiempos de espera de sesión, terminación del árbol de procesos, dependencias de la sesión de inicio) — nunca fue diseñado para daemons.
- **NSSM / WinSw / Shawl** son un wrapper por servicio. Sin panel, sin vista central, hay que registrar N veces.
- **PM2** necesita un runtime de Node.js, su daemon inactivo consume 40–80 MB, y es un ciudadano de segunda clase en Windows (inicio poco fiable, parches en el registro).
- **Supervisor** ni siquiera funciona en Windows.

Perch llena ese vacío: **registrar una vez, gestionar todo.** Un único servicio nativo supervisa todos tus procesos, expone una web UI y se mantiene fuera del camino.

## Características

- **Binario único** — sin runtime, sin dependencias. La web UI está incrustada directamente en el ejecutable.
- **Servicio nativo de Windows** — se registra una vez, arranca con el sistema, sobrevive al cierre de sesión, se ejecuta en Session 0.
- **Bajo consumo** — compilado en Rust, un único proceso supervisor. En reposo usa unos 4–5 MB de memoria residente, frente a los 40–80 MB de PM2.
- **Web UI integrada** — ver el estado, iniciar/detener/reiniciar procesos y seguir los logs desde el navegador. Gestiona de forma remota, no solo desde la consola.
- **Reinicio automático** — los procesos que fallan se reinician solos, con retroceso configurable.
- **Tareas programadas** — planificación estilo cron para trabajos periódicos.
- **Rotación diaria de logs** — los logs de la aplicación se escriben en `logs/perch.YYYY-MM-DD.log` junto al ejecutable, en formato delimitado por pipes.
- **Menú interactivo** — doble clic en el exe para instalar/desinstalar/iniciar/detener el servicio y abrir la web UI, con elevación UAC bajo demanda.

## Estado

Perch ya está publicado y es utilizable, actualmente en la versión v0.1.0. Al ser un proyecto en fase 0.x, las interfaces y la configuración todavía pueden cambiar entre versiones.

## Inicio rápido

Doble clic en `perch.exe` y selecciona `5) Open Web UI (Abrir Web UI)` en el menú — Perch generará automáticamente un `perch.toml` por defecto (con un proceso `demo` y el token `change-me`) e iniciará el servidor.

O desde la línea de comandos:

```
# Ejecutar en primer plano (Ctrl-C para detener)
perch run

# Registrar Perch como servicio de Windows (necesita admin)
perch install
perch start

# Abrir la web UI
http://127.0.0.1:9000
```

El token por defecto es `change-me` — cámbialo en `perch.toml` antes de cualquier uso real.

### Primer uso: SmartScreen de Windows

El binario publicado no está firmado digitalmente, así que en el primer inicio Windows SmartScreen puede mostrar **"Windows protegió su PC"**. Esto es normal en herramientas de código abierto sin firmar. Haz clic en **Más información → Ejecutar de todas formas** para continuar. (Descargar el `.zip` evita la advertencia de descarga del navegador; el aviso de SmartScreen en el primer inicio es un paso distinto e inevitable sin un certificado de firma de código de pago.)

### Ejecutar el servicio con tu propia cuenta de usuario

Por defecto el servicio de Perch se ejecuta como `LocalSystem`, que solo ve las variables de entorno a nivel de sistema, no tu PATH de usuario (`HKCU\Environment`, por ejemplo `%USERPROFILE%\.local\bin`, donde viven herramientas como `uv` o CLIs instaladas con `pipx`). Si un proceso gestionado o una tarea programada llama a un comando que depende de variables de entorno de usuario, fallará con "command not found" bajo el servicio, aunque funcione perfectamente cuando lo ejecutas tú mismo.

Para que el servicio se ejecute con tu propia cuenta en lugar de `LocalSystem`:

1. Pulsa `Win+R`, escribe `services.msc` y pulsa Enter.
2. Busca **Perch Process Manager**, haz clic derecho y elige **Propiedades**.
3. Ve a la pestaña **Iniciar sesión**.
4. Selecciona **Esta cuenta**, introduce tu cuenta de Windows (`.\tuusuario` para una cuenta local, o `DOMINIO\tuusuario` para una de dominio) y tu contraseña (dos veces).
5. Haz clic en Aceptar; Windows concede automáticamente a la cuenta el derecho "Iniciar sesión como servicio". Reinicia el servicio (clic derecho → Reiniciar, o `perch stop` seguido de `perch start`).

El servicio ahora se ejecuta como tú, así que los procesos hijos heredan todo tu entorno de usuario, incluido el PATH de usuario.

Advertencias:
- Si cambias tu contraseña de Windows más adelante, el servicio no podrá iniciarse hasta que actualices la contraseña en la misma pestaña Iniciar sesión.
- Alternativamente, evita el cambio de cuenta usando rutas absolutas a los ejecutables en tus comandos de proceso/tarea, o estableciendo explícitamente las variables necesarias.

## Seguridad

El endpoint de gestión puede iniciar y detener procesos arbitrarios — tiene **privilegios muy altos**. Perch se vincula a `127.0.0.1` por defecto y requiere un token de acceso. No expongas el puerto de gestión a redes no fiables.

## Comparación

| | Perch | NSSM / WinSw | PM2 | Supervisor |
|---|:---:|:---:|:---:|:---:|
| Binario único | ✅ | ✅ | ❌ (necesita Node) | ❌ (necesita Python) |
| Gestiona muchos procesos desde una instancia | ✅ | ❌ (1 por servicio) | ✅ | ✅ |
| Web UI integrada | ✅ | ❌ | ✅ | ✅ |
| Servicio nativo de Windows | ✅ | ✅ | ⚠️ | ❌ |
| Tareas programadas | ✅ | ❌ | ✅ | ⚠️ |
| Rotación diaria de logs | ✅ | ❌ | ⚠️ | ⚠️ |
| Bajo consumo de memoria | ✅ | ✅ | ❌ | ⚠️ |

## Compilar desde el código fuente

```
cargo build --release
```

Requiere una toolchain estable de Rust reciente.

## Licencia

[MIT](LICENSE) — consulte el archivo [LICENSE](LICENSE) para más detalles.
