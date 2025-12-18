# stord

单文件存储服务部署脚本（MariaDB / MongoDB / Redis），适合 1 核 1G 这类小机器做“存储节点”。

你可以把它当成一个“交互式运维菜单 + 一组可直接调用的 CLI 命令”，用于：
- 一键生成配置并用 Docker Compose 拉起服务
- 快速查看状态/日志/连接串（DSN）
- 备份/恢复（MariaDB、MongoDB）
- 定时备份（crontab）
- 导出/导入配置（不包含 data）

## Install (one-liner)

```bash
curl -fsSL https://raw.githubusercontent.com/zxkws/stord/main/stord | bash
```

这会把 `stord` 安装到：
- root：`/usr/local/bin/stord`
- 非 root：`~/.local/bin/stord`（如未在 PATH，按脚本提示把它加入 PATH）

也可以用 `wget`：

```bash
wget -O /tmp/stord https://raw.githubusercontent.com/zxkws/stord/main/stord \
  && chmod +x /tmp/stord \
  && sudo mv /tmp/stord /usr/local/bin/stord
```

手动安装（离线/内网）：

```bash
curl -fsSL https://raw.githubusercontent.com/zxkws/stord/main/stord -o stord
chmod +x stord
sudo mv stord /usr/local/bin/stord
```

## Run

```bash
stord
```

提示：
- 默认数据目录是 `/opt/stord`，通常需要 root 权限创建；如果你用普通用户运行，脚本会提示并建议：
  - 继续使用默认路径：`sudo -E stord`
  - 或指定用户目录：`STORD_HOME="$HOME/.stord" stord`

## Prerequisites

- Linux 服务器（推荐 Ubuntu/Debian/CentOS/Alpine 等）
- Docker + Docker Compose（脚本会检测；缺失时可选择用官方脚本安装 Docker）
- 如需“定时备份”：需要 `crontab`（脚本会检测并提示安装），并确保 cron 服务在运行
- 如需“防火墙白名单助手”：需要 `ufw` 或 `firewalld`（并且必须 root）

## Quick Start (recommended flow)

1) 安装：`curl ... | bash`
2) 运行：`sudo -E stord`
3) 选择 `Deploy` 部署 MariaDB/MongoDB/Redis
4) 进入 `Manage a service` 查看 `Status / Logs / DSN`
5) 配置你的业务应用连接 DSN（建议不要把 DB/Redis 暴露到公网）

## Menu usage (all numeric)

打开后是顶层菜单（输入数字即可）：
- `1) Manage a service`：先选服务，再选动作（Status/Logs/Restart/Backup/...）
- `2) Deploy services`：部署 MariaDB/MongoDB/Redis
- `3) Backups`：列出/清理备份、定时备份
- `4) Firewall whitelist helper`：白名单某 IP -> 端口
- `5) Doctor`：环境自检
- `6) Install` / `7) Self-update`：安装/更新脚本

## CLI usage

脚本既能交互，也能直接跑命令（适合自动化）：

```bash
stord --help
stord doctor
stord deploy mariadb
stord status            # = status all
stord logs mongodb
stord dsn               # = dsn all
```

常用示例：

```bash
# 查看所有服务状态（如果未部署，会提示 No deployed services）
stord status

# 只看 MongoDB 日志（follow）
stord logs mongodb

# 显示（模板）连接串：密码会用 <password> 占位
stord dsn mariadb
```

## Data layout

默认在 `/opt/stord`：

- `/opt/stord/services/<svc>/`
  - `docker-compose.yml`
  - `.env`（包含端口/账号/密码等敏感信息）
  - `data/`（持久化数据目录）
  - `conf/` 或 `conf.d/`（服务配置）
- `/opt/stord/backups/`（备份文件）
- `/opt/stord/logs/`（备份计划任务日志等）

可用环境变量覆盖根目录：

```bash
STORD_HOME=/data/stord stord doctor
```

## Security notes (important)

- 你可以选择把服务绑定到 `127.0.0.1`（仅本机可访问）或 `0.0.0.0`（远程可访问）。
- 如果绑定 `0.0.0.0`，务必用防火墙只放行你的应用服务器 IP；MongoDB/Redis 尤其不要直接暴露到公网。
- `stord secrets <svc>` 会把 `.env` 原文打印到终端历史记录里，请谨慎使用。

## Backups

支持：
- MariaDB：`stord backup mariadb`（生成 `.sql` 或 `.sql.gz`）
- MongoDB：`stord backup mongodb`（生成 `.archive` 或 `.archive.gz`）
- Redis：不做逻辑备份（AOF/RDB 在 `data/`，建议用卷快照/文件级备份）

常用命令：

```bash
stord backup mariadb
stord backup mongodb
stord backup files all
stord backup prune all 14
```

恢复：

```bash
stord restore mariadb /opt/stord/backups/mariadb-XXXX.sql.gz
stord restore mongodb /opt/stord/backups/mongodb-XXXX.archive.gz
```

定时备份（crontab）：

```bash
stord backup enable mariadb
stord backup list
stord backup disable mariadb
```

## Export / Import config (no data)

导出（不包含 `data/`）：

```bash
stord export-config mariadb
```

导入（覆盖 `.env`/`docker-compose.yml`/conf，**不会动 data/**）：

```bash
stord import-config mariadb /path/to/stord-mariadb-config-XXX.tgz
```

## Update

- 自动提示：打开交互菜单时，会检查一次远端版本并询问是否更新。
- 手动：

```bash
stord check-update
stord self-update
```

如你不想自动检查更新：

```bash
STORD_NO_UPDATE_CHECK=1 stord
```

## Notes

- 默认数据目录：`/opt/stord`
- 默认时区：`Asia/Seoul`（可自行改各服务的 `.env`）
