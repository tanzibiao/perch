<div align="center">
<sub>🇬🇧 <a href="./README.md">English</a> &nbsp;|&nbsp; 🇨🇳 <a href="./README.zh-CN.md">中文</a> &nbsp;|&nbsp; 🇪🇸 <a href="./README.es.md">Español</a> &nbsp;|&nbsp; 🇯🇵 <a href="./README.ja.md">日本語</a> &nbsp;|&nbsp; 🇧🇷 <a href="./README.pt.md">Português</a></sub>
</div>

<br/>

<div align="center">

<img src="logo.png" alt="Perch" width="120">

# Perch

**A lightweight, single-binary process & service manager for Windows, with a built-in web UI.**

Written in Rust. No runtime dependencies. One binary, double-click and go.

</div>

---

## Screenshots

### Interactive Menu (double-click `perch.exe`)

![menu](docs/screenshots/menu.png)

### Login

![login](docs/screenshots/login.png)

### Processes Dashboard

![processes](docs/screenshots/processes.png)

### Process Detail — Live Log Streaming

![process-detail](docs/screenshots/process-detail.png)

### Scheduled Tasks

![tasks](docs/screenshots/tasks.png)

---

## Why Perch?

Managing background processes and long-running services on Windows is more painful than it should be:

- **Task Scheduler** kills long-running processes by default (session timeouts, process-tree termination, login-session dependencies) — it was never meant for daemons.
- **NSSM / WinSw / Shawl** are one-wrapper-per-service. No dashboard, no central view, register N times.
- **PM2** needs a Node.js runtime, its daemon idles at 40–80 MB, and it's a second-class citizen on Windows (unreliable startup, registry hacks).
- **Supervisor** doesn't run on Windows at all.

Perch fills the gap: **register once, manage everything.** A single native service supervises all your processes, exposes a web UI, and stays out of the way.

## Features

- **Single binary** — no runtime, no dependencies. The web UI is embedded directly in the executable.
- **Native Windows service** — registers once, starts on boot, survives logout, runs in Session 0.
- **Low footprint** — compiled Rust, a single supervisor process. Idles at around 4–5 MB resident, versus PM2's 40–80 MB.
- **Built-in web UI** — view status, start/stop/restart processes, tail logs from your browser. Manage remotely, not just from the console.
- **Auto-restart** — crashed processes are brought back up automatically, with configurable backoff.
- **Scheduled tasks** — cron-style scheduling for periodic jobs.
- **Daily-rotating logs** — application logs written to `logs/perch.YYYY-MM-DD.log` next to the executable, pipe-delimited format.
- **Interactive menu** — double-click the exe to install/uninstall/start/stop the service and open the Web UI, with on-demand UAC elevation.

## Status

Perch is released and usable today, currently at v0.1.0. As a young 0.x project, interfaces and configuration may still change between releases.

## Quick Start

Double-click `perch.exe` and select `5) Open Web UI (打开 Web UI)` from the menu — Perch will auto-generate a default `perch.toml` (with a `demo` process and token `change-me`) and launch the server.

Or from the command line:

```
# Run in foreground (Ctrl-C to stop)
perch run

# Register Perch as a Windows service (needs admin)
perch install
perch start

# Open the web UI
http://127.0.0.1:9000
```

The default token is `change-me` — change it in `perch.toml` before any real use.

### First run: Windows SmartScreen

The released binary is not code-signed, so on first launch Windows SmartScreen may show **"Windows protected your PC"**. This is expected for unsigned open-source tools. Click **More info → Run anyway** to proceed. (Downloading the `.zip` avoids the browser's download warning; the SmartScreen prompt on first run is separate and unavoidable without a paid code-signing certificate.)

### Running under your user account

By default the Perch service runs as `LocalSystem`, which only sees system-level environment variables — not your user PATH (`HKCU\Environment`, e.g. `%USERPROFILE%\.local\bin` where tools like `uv` or `pipx`-installed CLIs live). If a managed process or scheduled task calls a command that depends on user-level environment variables, it will fail with "command not found" under the service even though it works fine when you run it yourself.

To make the service run as your own account instead:

1. Press `Win+R`, type `services.msc`, press Enter.
2. Find **Perch Process Manager**, right-click it, select **Properties**.
3. Go to the **Log On** tab.
4. Select **This account**, enter your Windows account (`.\yourname` for a local account, or `DOMAIN\yourname`), and your password (twice).
5. Click OK — Windows grants the account the "Log on as a service" right automatically. Restart the service (right-click → Restart, or `perch stop` then `perch start`).

The service now runs as you, so child processes inherit your full user environment, including user PATH.

Caveats:
- If you change your Windows password later, the service will fail to start until you update the password on the same Log On tab.
- Alternatively, avoid the account change entirely by using absolute paths to executables in your process/task commands, or by setting the needed variables explicitly.

## Security

The management endpoint can start and stop arbitrary processes — it is **highly privileged**. Perch binds to `127.0.0.1` by default and requires an access token. Do not expose the management port to untrusted networks.

## Comparison

| | Perch | NSSM / WinSw | PM2 | Supervisor |
|---|:---:|:---:|:---:|:---:|
| Single binary | ✅ | ✅ | ❌ (needs Node) | ❌ (needs Python) |
| Manages many processes from one instance | ✅ | ❌ (1 per service) | ✅ | ✅ |
| Built-in web UI | ✅ | ❌ | ✅ | ✅ |
| Native Windows service | ✅ | ✅ | ⚠️ | ❌ |
| Scheduled tasks | ✅ | ❌ | ✅ | ⚠️ |
| Daily-rotating app logs | ✅ | ❌ | ⚠️ | ⚠️ |
| Low memory footprint | ✅ | ✅ | ❌ | ⚠️ |

## Building from Source

```
cargo build --release
```

Requires a recent stable Rust toolchain.

## License

[MIT](LICENSE) — see the [LICENSE](LICENSE) file for details.
