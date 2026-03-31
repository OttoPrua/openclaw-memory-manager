# Cron 任务 — 参考配置

<p align="center">
  <a href="README.md">English</a> · <a href="README.zh-CN.md">中文</a>
</p>

用于维护记忆健康的 OpenClaw cron 任务参考配置。每个 `.json` 文件可直接配合 `openclaw cron add` 使用，或作为 `cron.add` 工具调用的模板。

## 任务列表

| 文件 | 调度 | 说明 |
|------|------|------|
| [`dream-cycle.json`](dream-cycle.json) | 每周（周日 08:00） | 记忆整合 — 精炼 session 日志、归档、去重 patterns |
| [`daily-progress-sync.json`](daily-progress-sync.json) | 每日（04:00） | 从日历同步项目进度到 Blackboard |
| [`monthly-cleanup.json`](monthly-cleanup.json) | 每月（1 日 03:00） | 归档 memory/ 根目录旧 session 日志 |

> 调度时区为 `Asia/Shanghai`，请按实际时区调整 `tz` 和 `expr`。

## 使用方式

### 通过 CLI 添加

```bash
# 以 Dream Cycle 为例
openclaw cron add \
  --name "Dream Cycle (Memory Consolidation)" \
  --cron "0 8 * * 0" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "执行记忆整合：扫描 memory/ 根目录日期格式 session 日志，精炼为 ≤30 行，归档至 memory/archive/YYYY-MM/，对 patterns/ 去重。" \
  --announce
```

### 通过工具调用（agent）

```json
{
  "name": "Dream Cycle (Memory Consolidation)",
  "schedule": { "kind": "cron", "expr": "0 8 * * 0", "tz": "Asia/Shanghai" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "执行记忆整合：扫描 memory/ 根目录日期格式 session 日志，精炼为 ≤30 行，归档至 memory/archive/YYYY-MM/，对 patterns/ 去重。"
  },
  "delivery": { "mode": "announce", "channel": "last" }
}
```

### 验证

```bash
openclaw cron list
openclaw cron run <jobId> --force   # 手动测试运行
openclaw cron runs --id <jobId>     # 查看运行历史
```
