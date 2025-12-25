---
mode: plan
cwd: /Users/bytedance/gh-space/stord
task: 调查 stord 脚本服务器运行问题并产出 Report.md
complexity: medium
planning_method: builtin
created_at: 2025-12-25T11:49:40+08:00
---

# Plan: stord 服务器运行问题调查与报告

🎯 任务概述

当前仓库仅包含一个 Bash 单文件脚本 `stord`（v0.2.7），用于在服务器上通过 Docker Compose 部署 MariaDB/MongoDB/Redis 并提供备份/恢复/cron/防火墙等运维能力。
目标是在“不稳定的服务器现场”下定位脚本实际运行问题的根因，形成可复现的证据链，并产出 `Report.md`（问题清单、影响面、修复建议与验证方式）。

📋 执行计划

1. 现场信息采集与最小复现
   - 采集 OS、bash、docker/compose 版本与运行方式（交互/非交互、curl|bash、sudo 环境变量）。
   - 开启调试输出复现：`STORD_DEBUG=1 stord doctor`、`STORD_DEBUG=1 stord <command>`，并保存完整 stdout/stderr。

2. 关键路径静态审查（先 P0 再 P2）
   - 优先检查会“直接导致命令不可用/部署失败”的路径：compose 调用、.env 生成与解析、依赖安装、端口检测。
   - 同时标注「环境相关」点（cron/防火墙/包管理器）以便对照不同发行版差异。

3. 对齐版本与兼容性矩阵
   - 建立最小矩阵：Ubuntu/Debian（apt）、CentOS/RHEL（yum/dnf）、Alpine（apk）各至少一台（或容器/VM）。
   - 针对 compose：分别验证 `docker compose`（v2 插件）与 `docker-compose`（v1）路径。

4. 形成问题清单 + 优先级 + 证据
   - 每个问题按「现象 → 触发条件 → 根因 → 修复建议 → 验证步骤」结构记录。
   - 优先级建议：P0（阻断部署/命令）、P1（高概率误用/数据风险）、P2（体验/覆盖面）。

5. 输出并迭代 `Report.md`
   - 先输出“静态审查版报告”（不依赖现场日志），明确待补信息清单。
   - 收到服务器日志后，补齐“已确认问题”与“已复现证据”，更新修复建议与验证结论。

6. 后续（可选）：落地修复补丁与回归清单
   - 按 P0→P2 顺序提交小步补丁（避免大重构），每个补丁附验证命令。
   - 关键动作（deploy/backup/restore/cron/uninstall）至少回归一遍。

⚠️ 风险与注意事项

- 缺少“服务器现场错误日志/版本信息”时，很多问题只能给出高概率风险点；必须通过采样信息确认。
- 部署/恢复/卸载涉及数据目录与凭据，任何自动化修复都必须提供回滚/备份建议。
- 兼容性修复要避免破坏现有 CLI 行为（向后兼容优先）。

📎 参考

- `stord:394`：compose 统一调用中使用 `--project-directory`（可能与旧版 docker-compose v1 不兼容）
- `stord:939` / `stord:1078` / `stord:1189`：`.env` 直接写入未做转义/校验（用户输入特殊字符可能导致解析异常）
- `stord:944` / `stord:963` / `stord:1081` / `stord:1190`：时区硬编码 `Asia/Seoul` / `+09:00`
- `stord:530`：`ensure_crontab()` 安装后仅提示，不保证 cron 服务已运行
- `stord:1725`：防火墙助手仅支持 `ufw` / `firewalld`
- `stord:193` / `stord:479`：脚本隐含依赖 bash>=4（`${var,,}`、`mapfile`）
