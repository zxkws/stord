# stord 脚本服务器运行问题调查报告（静态审查 + 已修复项）

> 版本基线：`stord` v0.2.10
>
> 说明：当前仍未提供“服务器现场报错/系统信息”，因此本报告以静态审查为主；同时已按“尽量兼容”的要求，对脚本做了一轮兼容性加固，并在本报告中标注已修复项与后续建议。

## 0. 本次变更摘要（v0.2.10）

- 已修复：兼容 `docker-compose` v1（不再使用 `--project-directory`）
- 已加固：部署时对关键输入做保守校验（减少 `.env`/配置解析坑）
- 已修复：备份/恢复不再通过容器内 `sh -c` 拼接命令（减少引号/转义问题）
- 已调整：默认时区改为可配置（`STORD_TZ`/`STORD_DB_TZ`），默认使用 UTC
- 已移除：防火墙助手（菜单项 + `stord firewall` 命令）
- 已调整：新部署默认绑定 `127.0.0.1`；选择 `0.0.0.0` 时需要二次确认

## 1. 背景与目标

`stord` 是一个 Bash 单文件部署/运维脚本，用 Docker Compose 在服务器上部署 MariaDB/MongoDB/Redis，并提供状态/日志/DSN、备份恢复、crontab 定时备份等能力。

本次目标：
- 解释“服务器实际运行遇到不少问题”的可能根因分布；
- 给出可复现与可验证的调查路径；
- 明确修复优先级与交付物（后续补丁/验证清单）。

## 2. 调查范围与方法

- 范围：仅当前仓库文件 `stord` 与 `README.md`。
- 方法：静态审查 + 风险点定位（兼容性/依赖/权限/配置生成/跨发行版差异）。

## 3. 已发现的高概率问题点（按优先级）

### P0：`docker-compose` 兼容性（`--project-directory`）

- **现象（可能）**：在安装了旧版 `docker-compose` v1（尤其是发行版仓库里较老版本）时，执行任意 `stord status|logs|start|deploy ...` 可能直接报错：`unknown flag: --project-directory`。
- **位置**：`compose()` 统一入口 `stord:389`。
- **影响面**：所有 compose 调用路径。
- **处理状态**：✅ 已在 v0.2.9 修复（改为 `cd` 到服务目录后执行 compose，兼容 v1/v2）。
- **建议验证**：
  - 在目标服务器执行：`docker-compose version` 与 `docker compose version`（两者可能只存在一个）。
  - 复现命令：`stord status` / `stord logs mariadb`。

### P1：`.env` 生成缺少输入校验/转义

- **现象（可能）**：用户在交互输入密码/用户名等字段时包含空格、引号、换行、不可见字符等，会导致 `.env` 解析异常或值被截断，进而出现：
  - `docker compose up` 失败；
  - 容器启动但鉴权失败（密码与预期不一致）；
  - 后续 `backup/restore` 等命令执行失败。
- **位置**：写入 `.env` 的位置：`stord:991` / `stord:1136` / `stord:1255`。
- **处理状态**：✅ 已在 v0.2.9 加固（对关键字段做保守校验：仅允许字母数字及 `_@%+=:,./-`，且禁止空格/引号/`#`/`$`/换行）。
- **说明**：这是“兼容优先”的取舍，避免不同 compose/dotenv/配置文件解析差异导致的隐性故障；如确实需要更宽松的字符集，建议后续再做「引号/转义」增强并配套回归测试。
- **建议验证**：用包含空格/引号的密码做一次部署，确认脚本行为是否可控。

### P1：时区硬编码（`Asia/Seoul` / `+09:00`）

- **现象（可能）**：容器内时区与服务器时区不一致，导致：
  - 备份文件名时间、日志时间与预期不一致；
  - 数据库侧默认时区与业务预期不一致（尤其 MariaDB `default_time_zone`）。
- **位置**：`stord:991` / `stord:1136` / `stord:1255`（服务 `.env` 写入 `TZ=${STORD_TZ}`），以及 `stord:1015`（MariaDB `default_time_zone='${STORD_DB_TZ}'`）。
- **处理状态**：✅ 已在 v0.2.9 修复（新增可配置项 `STORD_TZ` 与 `STORD_DB_TZ`，默认使用 UTC / `+00:00`）。
- **建议验证**：部署后对比 `date`、MariaDB `SELECT @@global.time_zone, @@system_time_zone;`。

### P2：cron 依赖与服务状态

- **现象（可能）**：脚本可安装 `cron/cronie` 并写入 `crontab`，但如果系统 cron 服务未启用/未运行，定时备份不会执行。
- **位置**：`ensure_crontab()` 仅提示“确保 cron 服务在运行”：`stord:571`。
- **建议修复方向**：
  - 在报告/文档里补充明确的启用命令（例如 Debian/Ubuntu：`systemctl enable --now cron`；RHEL：`systemctl enable --now crond`）。
  - 是否由脚本自动启动需谨慎（避免越权修改系统服务）。

### P2：防火墙助手（已移除）

- **处理状态**：✅ 已在 v0.2.9 移除（为减少跨发行版差异与误导性配置）。
- **建议**：优先使用云厂商安全组/ACL，或由运维侧使用 `ufw`/`firewalld`/`iptables`/`nft` 进行统一管理。

### P2：隐含 bash 版本要求

- **现象（可能）**：在 bash 版本较老（<4）的环境下，脚本会因为 `${var,,}`、`mapfile` 等语法/内建缺失而失败。
- **位置**：`stord:199`（`${ans,,}`），`stord:520`（`mapfile`）。
- **处理状态**：✅ 已在 v0.2.9 加固（启动时显式检查 Bash >= 4.0，并给出明确错误信息）。
  - 说明：脚本内部还使用了 `declare -A` 等 Bash 4 特性，短期内“强行兼容 Bash 3.x”会带来较大改动与回归成本。

## 4. 需要补充的服务器现场信息（强烈建议）

为把“高概率问题”收敛为“已确认问题”，请在出问题的服务器上收集以下信息并粘贴（脱敏后）：

1) 系统与版本
- `cat /etc/os-release`
- `bash --version | head -n 1`

2) Docker 与 Compose
- `docker --version`
- `docker compose version || true`
- `docker-compose version || true`

3) stord 运行方式与日志
- 运行命令（例如 `sudo -E stord deploy mariadb` / `curl ... | sudo bash`）
- 复现时开启调试：`STORD_DEBUG=1 stord <command>` 的完整输出

4) 服务侧日志（如部署/启动失败）
- `stord logs mariadb` / `stord logs mongodb` / `stord logs redis`
- `docker ps -a`

## 5. 建议的调查/修复执行顺序

- **先验证已修复项（v0.2.9）**：在目标服务器升级后跑 `stord status` / `stord logs <svc>`，确认 compose v1/v2 均正常。
- 然后验证 **P1（.env 输入）** 与 **P1（时区）**：确认新增校验符合预期；如需非 UTC 时区，设置 `STORD_TZ`/`STORD_DB_TZ` 后再部署。
- 最后处理 **P2（cron）**：更多是系统服务启用问题。

## 6. 下一步交付物

- `Report.md`（本文件）：持续更新为“已确认/已复现版”。
- （可选）修复补丁：按 P0→P2 小步提交，并附最小回归清单。
