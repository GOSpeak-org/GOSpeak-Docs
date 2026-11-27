# 单二进制部署

GOSpeak 支持编译为单文件二进制部署形态：前端资源已内嵌其中，将二进制复制到目标机器即可直接运行。

## 适用场景

- 裸机部署，且未安装 Docker
- 树莓派、NAS、软路由等资源有限的小型设备
- 无法安装容器的内网机器
- 需要注册为 systemd 服务

## 不适用场景

单二进制**只包含应用本身**，不含 LiveKit。因此：

- 仅文字聊天 → 单二进制已可满足需求
- 需要语音 → 另行部署 LiveKit，并通过 `LIVEKIT_HOST` 指向该实例

若希望减少部署步骤，可使用 [Docker Compose](/deploy/compose)，该方案会一并管理 LiveKit。

## 一键安装（推荐）

仓库的 `deploy/install-binary.sh` 会自动识别当前平台的系统与架构，从 GitHub Releases 拉取最新版本的 GOSpeak 与 LiveKit，校验 SHA256 后落地为可直接运行的服务：

```bash
curl -fsSL https://raw.githubusercontent.com/GOSpeak-org/GOSpeak-Server/release/deploy/install-binary.sh | bash
```

默认安装到 `/opt/gospeak`，包含：

```
/opt/gospeak/
├── gospeak            GOSpeak 单二进制
├── livekit-server     LiveKit（非 Linux 平台自动跳过）
├── livekit.yaml       LiveKit 配置，密钥由 .env 注入
├── .env               随机生成的配置（权限 600）
└── data/              SQLite 与应用数据
```

会同时写入 `gospeak.service` 与 `livekit.service`（无 systemd 环境自动跳过），并已为 `.env` 填入随机的 `JWT_KEY` / `STORAGE_ENCRYPT_KEY` / `METRICS_TOKEN` / `LIVEKIT_KEY` / `LIVEKIT_SECRET`（`METRICS_TOKEN` 是 `server -e prod` 启动的必填项，老安装重复执行时会补上）。

先看要装什么，不下载：

```bash
curl -fsSL https://raw.githubusercontent.com/GOSpeak-org/GOSpeak-Server/release/deploy/install-binary.sh | bash -s -- --dry-run
```

常用参数：

| 参数 | 作用 |
|------|------|
| `--dir DIR` | 安装目录，默认 `/opt/gospeak` |
| `--user USER` | systemd 运行用户，默认当前用户（sudo 时为 `SUDO_USER`） |
| `--target OS-ARCH` | 指定平台，如 `linux-amd64`，默认自动检测 |
| `--version VER` | 指定 GOSpeak 版本，默认最新 |
| `--livekit-version VER` | 指定 LiveKit 版本，默认最新 |
| `--without-livekit` | 只要应用，不下载 LiveKit |
| `--without-systemd` | 只落盘，不写 systemd 单元 |
| `--dry-run` | 只解析版本与 URL |

**重复执行即升级**：脚本会替换二进制，但**不覆盖已存在的 `.env` 与 `livekit.yaml`**。

安装完成后仍需放行端口（见 [端口](/deploy/#端口)）：`8998/TCP`、`7882/TCP+UDP`、`3478/UDP`。单机自用可先只开 8998，语音功能则必须开放后两者。

## 下载

从 [Releases](https://github.com/GOSpeak-org/GOSpeak-Server/releases) 页面下载 `gospeak-server-v` 开头的版本：对应二进制位于各 release 页面底部的 **Assets** 列表中，文件名形如 `gospeak-server-v<版本>-<系统>-<架构>`：

| 文件 | 平台 |
|------|------|
| `gospeak-server-v<VERSION>-linux-amd64` | Linux x86_64 |
| `gospeak-server-v<VERSION>-linux-arm64` | Linux ARM64（ARM 服务器、部分 NAS） |
| `gospeak-server-v<VERSION>-linux-armv7` | Linux ARMv7（树莓派 2/3） |
| `gospeak-server-v<VERSION>-darwin-amd64` | macOS Intel |
| `gospeak-server-v<VERSION>-darwin-arm64` | macOS Apple Silicon |
| `gospeak-server-v<VERSION>-windows-amd64.exe` | Windows x64 |
| `gospeak-server-v<VERSION>-windows-arm64.exe` | Windows on ARM |
| `gospeak-server-v<VERSION>-freebsd-*` | FreeBSD |
| `gospeak-server-v<VERSION>-openbsd-amd64` | OpenBSD |

同一页面还提供 `SHA256SUMS-gospeak-server-v<VERSION>.txt` 校验和文件，用于校验完整性：

```bash
sha256sum -c SHA256SUMS-gospeak-server-v<VERSION>.txt --ignore-missing
```

## 运行

```bash
chmod +x gospeak-server-v<VERSION>-linux-amd64
./gospeak-server-v<VERSION>-linux-amd64 server -e prod
```

服务默认监听 8998 端口，浏览器访问 `http://服务器IP:8998` 即进入 Web 界面。

前端资源已内嵌于二进制，无需额外部署静态文件。

### 参数

| 参数 | 作用 |
|------|------|
| `server` | 启动服务（未指定 `-e` 时默认 `dev` 环境；生产需显式 `server -e prod`） |
| `server -e dev` | 开发环境，读 `.env.dev` |
| `version` | 打印版本号 |

启动时会在**当前工作目录**读取 `.env` 文件，因此需在二进制所在目录放置 `.env`，或改用环境变量提供配置。

## 配置 systemd 服务

systemd 方式提供开机自启、崩溃自动重启与 journald 日志管理。

`/etc/systemd/system/gospeak.service`：

```ini
[Unit]
Description=GOSpeak
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=gospeak
Group=gospeak
WorkingDirectory=/opt/gospeak
ExecStart=/opt/gospeak/gospeak server -e prod
Restart=always
RestartSec=5
LimitNOFILE=65536

# 环境变量也可写在此处，或置于 /opt/gospeak/.env
Environment=DB_TYPE=SQLite
Environment=DB_PATH=/opt/gospeak/data/app.db
Environment=SERVER_PORT=8998

[Install]
WantedBy=multi-user.target
```

准备目录与用户：

```bash
sudo useradd -r -s /sbin/nologin gospeak
sudo mkdir -p /opt/gospeak/data
sudo cp gospeak-server-v<VERSION>-linux-amd64 /opt/gospeak/gospeak
sudo chown -R gospeak:gospeak /opt/gospeak
```

启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gospeak
sudo systemctl status gospeak
```

查看日志：

```bash
journalctl -u gospeak -f
```

## 外部前端目录

该方式通常无需使用；需要在不重新编译二进制的前提下更新前端时，可外挂一个静态目录：

```bash
STATIC_DIR=/var/www/gospeak ./gospeak-server-v<VERSION>-linux-amd64 server -e prod
```

优先级：`STATIC_DIR` > `/app/static` > `./static` > 二进制内嵌。

## 自行编译

以下命令在 GOSpeak-Server 仓库根目录执行（Go 模块位于 `server/`）：

```bash
cd server

# 先构建前端（会同步到 internal/webui/dist）
make web

# 单个平台
make linux-amd64-bin

# 一个系统下的所有架构
make linux
make darwin
make windows

# 全部
make all
```

编译产物位于 `server/build/`。

## 常见问题

**启动报 `no such file or directory` 找不到 .env**
进程的工作目录中不存在配置文件。通过 `WorkingDirectory` 指定工作目录，或将 `.env` 放置在二进制所在目录。

**SQLite 报权限错误**
运行用户对 `DB_PATH` 所在目录没有写权限，需通过 `chown` 修正目录属主。

**修改 .env 后未生效**
重启进程即可生效；systemd 部署下执行 `sudo systemctl restart gospeak`。

**内存占用持续增长**
Go 运行时的正常现象，内存占用会趋于稳定。若持续增长且不回落，可在项目中提交 issue。

**更换端口**
使用 `SERVER_PORT=9000 ./gospeak server -e prod` 启动，或将该变量写入 systemd 单元的 `Environment` 配置。

## 下一步

- [部署到服务器](/deploy/) — Docker 方案
- [配置项](/guide/configuration) — 全部环境变量
