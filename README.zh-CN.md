<div align="center">
<sub>🇬🇧 <a href="./README.md">English</a> &nbsp;|&nbsp; 🇨🇳 <a href="./README.zh-CN.md">中文</a> &nbsp;|&nbsp; 🇪🇸 <a href="./README.es.md">Español</a> &nbsp;|&nbsp; 🇯🇵 <a href="./README.ja.md">日本語</a> &nbsp;|&nbsp; 🇧🇷 <a href="./README.pt.md">Português</a></sub>
</div>

<br/>

<div align="center">

<img src="logo.png" alt="Perch" width="120">

# Perch

**轻量级 Windows 进程与服务管理器，单文件、内置 Web UI。**

Rust 编写，无运行时依赖。一个可执行文件，双击即用。

</div>

---

## 截图展示

### 交互菜单（双击 `perch.exe`）

![menu](docs/screenshots/menu.png)

### 登录页

![login](docs/screenshots/login.png)

### 进程面板

![processes](docs/screenshots/processes.png)

### 进程详情 —— 实时日志流

![process-detail](docs/screenshots/process-detail.png)

### 定时任务

![tasks](docs/screenshots/tasks.png)

---

## 为什么做 Perch？

在 Windows 上管理后台进程和常驻服务，比想象中麻烦得多：

- **任务计划程序（Task Scheduler）** 默认会杀掉长时间运行的进程（会话超时、进程树终止、依赖登录会话）——它本就不是为守护进程设计的。
- **NSSM / WinSw / Shawl** 都是一个包装器管一个服务。没有面板、没有全局视图，要注册 N 次。
- **PM2** 依赖 Node.js 运行时，守护进程空载就占 40–80 MB，且在 Windows 上是二等公民（自启动不可靠、要 hack 注册表）。
- **Supervisor** 根本不支持 Windows。

Perch 填补了这个空白：**注册一次，管理全部。** 单个原生服务看管所有进程，提供 Web UI，安静地待在后台。

## 特性

- **单文件** —— 无运行时、无依赖。Web UI 直接内嵌进可执行文件。
- **原生 Windows 服务** —— 注册一次，开机自启，注销不停，运行于 Session 0。
- **极低占用** —— Rust 编译，单个守护进程，空闲时常驻内存约 4–5 MB，对比 PM2 的 40–80 MB。
- **内置 Web UI** —— 浏览器里查看状态、启停/重启进程、查看日志。支持远程管理，不必守在机器前。
- **崩溃自动拉起** —— 进程挂掉自动重启，退避策略可配置。
- **定时任务** —— cron 风格的周期任务调度。
- **按日切分日志** —— 应用日志写到可执行文件同级的 `logs/perch.YYYY-MM-DD.log`，管道分隔格式。
- **交互菜单** —— 双击 exe 即可安装/卸载/启动/停止服务、打开 Web UI，特权操作按需触发 UAC 提权。

## 项目状态

Perch 已发布，当前版本 v0.1.0，可正常使用。作为 0.x 阶段的项目，接口与配置在后续版本中仍可能调整。

## 快速开始

双击 `perch.exe`，在菜单中选 `5) 打开 Web UI (Open Web UI)`——Perch 会自动生成默认 `perch.toml`（含 `demo` 进程和 token `change-me`）并启动服务。

或命令行启动：

```
# 前台运行（Ctrl-C 停止）
perch run

# 注册为 Windows 服务（需管理员）
perch install
perch start

# 打开 Web UI
http://127.0.0.1:9000
```

默认 token 是 `change-me`——正式使用前请改 `perch.toml` 里的 `server.token`。

### 首次运行：Windows SmartScreen 提示

发布的二进制文件未做代码签名，因此首次启动时 Windows SmartScreen 可能会显示 **"Windows 已保护你的电脑"**。这是未签名开源工具的正常现象，点击 **更多信息 → 仍要运行** 即可继续。（下载 `.zip` 可以避免浏览器的下载警告；但首次运行时的 SmartScreen 提示是另一回事，除非购买代码签名证书，否则无法避免。）

### 让服务以你自己的账户运行

Perch 服务默认以 `LocalSystem` 身份运行，只能看到系统级环境变量，看不到你的用户级 PATH（`HKCU\Environment`，例如 `%USERPROFILE%\.local\bin` 这类 `uv`、`pipx` 安装的 CLI 工具所在的目录）。如果被管理的进程或计划任务需要调用依赖用户级环境变量的命令，用服务运行时会报 "command not found"，即便你自己手动运行完全正常。

解决方法是让服务以你自己的账户运行，而不是 `LocalSystem`：

1. 按 `Win+R`，输入 `services.msc`，回车。
2. 找到 **Perch Process Manager**，右键 → **属性**。
3. 切换到 **登录** 选项卡。
4. 选择 **此账户**，填入你的 Windows 账户（本地账户填 `.\你的用户名`，域账户填 `DOMAIN\你的用户名`），并输入两次密码。
5. 点击确定，Windows 会自动为该账户授予"以服务方式登录"权限。然后重启服务（右键 → 重新启动，或执行 `perch stop` 再 `perch start`）。

之后服务以你的身份运行，子进程会继承你完整的用户环境变量，包括用户 PATH。

注意：
- 之后修改了 Windows 密码，服务会因密码不对而无法启动，需要到同一个"登录"选项卡里更新密码。
- 也可以不改账户，直接在进程/任务命令里用可执行文件的绝对路径，或显式设置所需的环境变量来规避这个问题。

## 安全

管理端点可以启停任意进程，**权限极高**。Perch 默认仅监听 `127.0.0.1` 并要求访问令牌。不要将管理端口暴露到不可信网络。

## 对比

| | Perch | NSSM / WinSw | PM2 | Supervisor |
|---|:---:|:---:|:---:|:---:|
| 单文件 | ✅ | ✅ | ❌（需 Node） | ❌（需 Python） |
| 单实例管理多进程 | ✅ | ❌（一个一包装） | ✅ | ✅ |
| 内置 Web UI | ✅ | ❌ | ✅ | ✅ |
| 原生 Windows 服务 | ✅ | ✅ | ⚠️ | ❌ |
| 定时任务 | ✅ | ❌ | ✅ | ⚠️ |
| 按日切分应用日志 | ✅ | ❌ | ⚠️ | ⚠️ |
| 低内存占用 | ✅ | ✅ | ❌ | ⚠️ |

## 从源码构建

```
cargo build --release
```

需要较新的 stable Rust 工具链。

## 许可证

[MIT](LICENSE) — 详见 [LICENSE](LICENSE) 文件。
