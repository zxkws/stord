# stord

单文件存储服务部署脚本（MariaDB / MongoDB / Redis），适合 1 核 1G 这类小机器做“存储节点”。

你可以把它当成一个“交互式运维菜单 + 一组可直接调用的 CLI 命令”，用于：
- 一键生成配置并用 Docker Compose 拉起服务
- 快速查看状态/日志/连接串（DSN）
- 备份/恢复（MariaDB、MongoDB）
- 定时备份（crontab）
- 导出/导入配置（不包含 data）

## 安装（one-liner）

```bash
curl -fsSL https://raw.githubusercontent.com/zxkws/stord/main/stord | sudo bash
```

这会把 `stord` 安装到：
- `/usr/local/bin/stord`（需要 root）

也可以用 `wget`：

```bash
wget -O /tmp/stord https://raw.githubusercontent.com/zxkws/stord/main/stord \
  && chmod +x /tmp/stord \
  && sudo cp /tmp/stord /usr/local/bin/stord
```

手动安装（离线/内网）：

```bash
curl -fsSL https://raw.githubusercontent.com/zxkws/stord/main/stord -o stord
chmod +x stord
sudo cp stord /usr/local/bin/stord
```

## 运行

```bash
sudo -E stord
```

提示：
- 脚本每次运行都会检查 root；请用 `sudo` 或切换到 root 账户。
- 默认数据目录是 `/opt/stord`。
- 所有确认提示默认是 **Yes**，直接回车表示同意默认选项。

## 依赖

- Linux 服务器（推荐 Ubuntu/Debian/CentOS/Alpine 等）
- Docker + Docker Compose（脚本会检测；缺失时可选择用官方脚本安装 Docker）
- 如需“定时备份”：需要 `crontab`（脚本会检测并提示安装），并确保 cron 服务在运行

Docker 安装提示：
- `get.docker.com` 只支持主流发行版；遇到不支持的发行版会提示并中止安装。
- 可用环境变量自定义镜像/代理：
  - `STORD_DOCKER_MIRROR`：传给 `get.docker.com` 的 `--mirror` 参数
  - `STORD_HTTP_PROXY` / `STORD_HTTPS_PROXY` / `STORD_NO_PROXY`：为安装过程设置代理

## 快速开始（推荐流程）

1) 安装：`curl ... | sudo bash`
2) 运行：`sudo -E stord`
3) 选择「部署服务」部署 MariaDB/MongoDB/Redis
4) 进入「管理服务」查看状态/日志/DSN
5) 配置你的业务应用连接 DSN（建议不要把 DB/Redis 暴露到公网）

## 菜单使用（纯数字）

打开后是顶层菜单（输入数字即可）：
- `1) 管理服务`：先选服务，再选动作（状态/日志/重启/备份/...）
- `2) 部署服务`：部署 MariaDB/MongoDB/Redis
- `3) 备份`：列出/清理备份、定时备份
- `4) Doctor`：环境自检
- `5) 安装/更新` / `6) 自更新`：安装/更新脚本
- `7) 检查更新`
- `8) 卸载/清理`：卸载脚本或清理全部服务与数据

## CLI 用法

脚本既能交互，也能直接跑命令（适合自动化）：

```bash
stord --help
stord doctor
stord deploy mariadb
stord status            # 默认 = status all
stord logs mongodb
stord dsn               # 默认 = dsn all
```

常用示例：

```bash
# 查看所有服务状态（如果未部署，会提示暂无已部署服务）
stord status

# 只看 MongoDB 日志（follow）
stord logs mongodb

# 显示（模板）连接串：密码会用 <password> 占位
stord dsn mariadb
```

## 数据目录结构

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

时区相关（可选）：

- `STORD_TZ`：写入容器环境变量 `TZ`，默认 `UTC`
- `STORD_DB_TZ`：MariaDB `default_time_zone`，默认 `+00:00`（也可用 `SYSTEM`）

## 卸载/清理

仅卸载脚本：

```bash
stord uninstall script
```

卸载脚本 + 所有服务与数据（会逐项提示，可跳过）：

```bash
stord uninstall all
```

仅卸载所有服务（保留数据与脚本）：

```bash
stord uninstall services
```

卸载单个服务（保留数据与脚本）：

```bash
stord uninstall mariadb
```

说明：
- 只清理由本脚本创建的服务目录/配置/备份/日志；不会卸载 Docker/cron 等系统组件。
- 安装/部署路径会记录在 `${STORD_HOME}/state/install-records`，便于卸载时列出并确认。
- `stord uninstall all` 会尝试清理 `crontab` 中由本工具写入的定时备份任务（`# stord:backup:`）。
- 如需删除服务数据目录，请使用菜单里的「彻底清理」或命令 `stord purge <svc>`。

## 安全提示（重要）

- 你可以选择把服务绑定到 `127.0.0.1`（仅本机可访问）或 `0.0.0.0`（远程可访问）。
- 新部署默认绑定到 `127.0.0.1`（更安全）；选择 `0.0.0.0` 时会要求二次确认。
- 部署时会询问使用默认端口还是随机端口；如果端口已被占用，会提示并建议随机端口或输入可用端口。
- 如果绑定 `0.0.0.0`，务必用防火墙只放行你的应用服务器 IP；MongoDB/Redis 尤其不要直接暴露到公网。
- `stord secrets <svc>` 会把 `.env` 原文打印到终端历史记录里，请谨慎使用。

## 备份

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

## 导出 / 导入配置（不含数据）

导出（不包含 `data/`）：

```bash
stord export-config mariadb
```

导入（覆盖 `.env`/`docker-compose.yml`/conf，**不会动 data/**）：

```bash
stord import-config mariadb /path/to/stord-mariadb-config-XXX.tgz
```

## 更新

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

## 其他

- 默认数据目录：`/opt/stord`
- 默认时区：`Asia/Seoul`（可自行改各服务的 `.env`）
