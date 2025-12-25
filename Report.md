# stord 脚本服务器运行问题调查报告（静态审查 + 已修复项）

> 版本基线：`stord` v0.2.12
>
> 说明：当前仍未提供“服务器现场报错/系统信息”，因此本报告以静态审查为主；同时已按“尽量兼容”的要求，对脚本做了一轮兼容性加固，并在本报告中标注已修复项与后续建议。

## 0. 本次变更摘要（v0.2.12）

- 已修复：兼容 `docker-compose` v1（不再使用 `--project-directory`）
- 已加固：部署时对关键输入做保守校验（减少 `.env`/配置解析坑）
- 已修复：备份/恢复不再通过容器内 `sh -c` 拼接命令（减少引号/转义问题）
- 已调整：默认时区改为可配置（`STORD_TZ`/`STORD_DB_TZ`），默认使用 UTC
- 已移除：防火墙助手（菜单项 + `stord firewall` 命令）
- 已调整：新部署默认绑定 `127.0.0.1`；选择 `0.0.0.0` 时需要二次确认
- 已加固：卸载/清理时会提示并清理 `crontab` 定时备份任务；compose 不可用时尝试用 `docker rm -f` 兜底停容器
- 已调整：默认数据目录不再强制 `/opt/stord`，改为优先使用 `~/.stord`（若检测到已有 `/opt/stord` 会沿用以保持兼容；也可用 `STORD_HOME` 覆盖）
- 已调整：不再全局强制 root；仅在需要安装系统依赖/写入系统目录时提示使用 sudo
- 已加固：Docker 已安装但不可用（权限/daemon 未启动）时给出更明确的提示

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
- **处理状态**：✅ 已修复（改为 `cd` 到服务目录后执行 compose，兼容 v1/v2）。
- **建议验证**：
  - 在目标服务器执行：`docker-compose version` 与 `docker compose version`（两者可能只存在一个）。
  - 复现命令：`stord status` / `stord logs mariadb`。

### P1：`.env` 生成缺少输入校验/转义

- **现象（可能）**：用户在交互输入密码/用户名等字段时包含空格、引号、换行、不可见字符等，会导致 `.env` 解析异常或值被截断，进而出现：
  - `docker compose up` 失败；
  - 容器启动但鉴权失败（密码与预期不一致）；
  - 后续 `backup/restore` 等命令执行失败。
- **位置**：写入 `.env` 的位置：`stord:991` / `stord:1136` / `stord:1255`。
- **处理状态**：✅ 已加固（对关键字段做保守校验：仅允许字母数字及 `_@%+=:,./-`，且禁止空格/引号/`#`/`$`/换行）。
- **说明**：这是“兼容优先”的取舍，避免不同 compose/dotenv/配置文件解析差异导致的隐性故障；如确实需要更宽松的字符集，建议后续再做「引号/转义」增强并配套回归测试。
- **建议验证**：用包含空格/引号的密码做一次部署，确认脚本行为是否可控。

### P1：时区硬编码（`Asia/Seoul` / `+09:00`）

- **现象（可能）**：容器内时区与服务器时区不一致，导致：
  - 备份文件名时间、日志时间与预期不一致；
  - 数据库侧默认时区与业务预期不一致（尤其 MariaDB `default_time_zone`）。
- **位置**：`stord:991` / `stord:1136` / `stord:1255`（服务 `.env` 写入 `TZ=${STORD_TZ}`），以及 `stord:1015`（MariaDB `default_time_zone='${STORD_DB_TZ}'`）。
- **处理状态**：✅ 已修复（新增可配置项 `STORD_TZ` 与 `STORD_DB_TZ`，默认使用 UTC / `+00:00`）。
- **建议验证**：部署后对比 `date`、MariaDB `SELECT @@global.time_zone, @@system_time_zone;`。

### P2：cron 依赖与服务状态

- **现象（可能）**：脚本可安装 `cron/cronie` 并写入 `crontab`，但如果系统 cron 服务未启用/未运行，定时备份不会执行。
- **位置**：`ensure_crontab()` 仅提示“确保 cron 服务在运行”：`stord:571`。
- **建议修复方向**：
  - 在报告/文档里补充明确的启用命令（例如 Debian/Ubuntu：`systemctl enable --now cron`；RHEL：`systemctl enable --now crond`）。
  - 是否由脚本自动启动需谨慎（避免越权修改系统服务）。

### P2：防火墙助手（已移除）

- **处理状态**：✅ 已移除（为减少跨发行版差异与误导性配置）。
- **建议**：优先使用云厂商安全组/ACL，或由运维侧使用 `ufw`/`firewalld`/`iptables`/`nft` 进行统一管理。

### P2：隐含 bash 版本要求

- **现象（可能）**：在 bash 版本较老（<4）的环境下，脚本会因为 `${var,,}`、`mapfile` 等语法/内建缺失而失败。
- **位置**：`stord:199`（`${ans,,}`），`stord:520`（`mapfile`）。
- **处理状态**：✅ 已加固（启动时显式检查 Bash >= 4.0，并给出明确错误信息）。
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

## 7. 需求对齐：是否仍符合“多服务器快速复用”初衷

结论：整体仍符合“多台服务器快速落地并复用同一套部署/运维流程”的初衷，但有几个前提与边界需要明确。

- 安装成本：支持一行命令安装到用户目录（`~/.local/bin`），不再强制 sudo，适合在多台机器快速落地。
- 数据落盘位置：默认使用 `~/.stord`，更贴近“非 root 即可用”；如你希望统一落到数据盘，请在所有机器上显式设置 `STORD_HOME=/data/stord`（或写入环境/脚本封装）。
- 依赖边界：核心依赖仍是 Docker/Compose。脚本不负责“把服务器从裸机变成可运行 Docker”的全部工作（只提供 best-effort 安装与提示），这符合低侵入原则。
- 复现与迁移：通过导出/导入配置与迁移打包（含 data）可以在机器之间搬迁，但仍建议把“配置/数据/备份”与“脚本本身”分层管理（例如备份目录单独同步）。

## 8. 若开源给所有人用，潜在问题与欠缺

- 安全默认：已将新部署默认绑定改为 `127.0.0.1` 并对公网绑定二次确认；但开源后仍需在文档中更醒目地强调“不要把数据库直接暴露到公网”。
- 密钥与日志：`.env` 含明文密码；`secrets` 命令会原样输出，容易进入终端历史/日志收集系统。开源版本建议补充更严格的提示/防误用设计（甚至默认隐藏该入口）。
- 兼容矩阵：不同发行版/包管理器/系统服务管理方式差异较大（systemd / openrc / busybox），建议明确支持范围与“已知不支持/需手动”的场景。
- Docker 权限：非 root 用户常见的 docker.sock 权限问题需要更多 FAQ/引导（加入 docker 组、rootless docker、daemon 未启动等）。
- 可测试性：目前主要是冒烟验证；若面向公众，建议增加最小 CI（bash -n、shellcheck、在 Debian/Ubuntu/Alpine 容器里跑基础命令）。
- 发布与完整性：建议提供 release（带 checksum/签名）而非只依赖 raw GitHub URL，避免供应链风险与误更新。

## 9. macOS 使用结论

- 脚本面向 Linux 服务器；macOS 自带 Bash 3.x 不满足 Bash >= 4 的要求。
- 若你希望“本地先跑通再上服务器”，更推荐用 Linux 容器/虚拟机做本地验证（与目标服务器环境更接近）。
