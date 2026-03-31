# OpenClaw 记忆架构

<p align="center">
  <picture>
    <img src="docs/assets/banner.svg" alt="OpenClaw 记忆架构" width="480">
  </picture>
</p>

<p align="center">
  <strong>OpenClaw 多 agent 完整记忆系统</strong><br>
  结构化文件层级 · 语义搜索 · 无损上下文压缩 · 自动化维护
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.zh-CN.md">中文</a>
</p>

<p align="center">
  <a href="https://clawhub.ai/OttoPrua/agent-memory-protocol">
    <img src="https://img.shields.io/badge/clawhub-agent--memory--protocol-brightgreen?style=for-the-badge" alt="ClawHub">
  </a>
  <a href="https://github.com/openclaw/openclaw">
    <img src="https://img.shields.io/badge/OpenClaw-compatible-orange?style=for-the-badge" alt="OpenClaw">
  </a>
  <img src="https://img.shields.io/badge/许可证-MIT-blue?style=for-the-badge" alt="License">
</p>

[OpenClaw](https://github.com/openclaw/openclaw) agent 在多个 session 中积累上下文。没有系统时，记忆会以可预期的方式崩溃：事实在多个文件中不同步漂移，决策在上下文压缩后消失，agent 浪费 token 通读所有内容只为找到一件事。本仓库描述了一套解决上述全部问题的完整架构。

**涵盖内容：**
- agent 写入的结构化 Markdown 文件层级（真相来源）
- [qmd](https://github.com/tobilen/qmd) — 对这些文件做本地语义搜索（驱动 `memory_search`）
- [LosslessClaw](https://github.com/martian-engineering/lossless-claw) — DAG 分层上下文压缩（历史对话保持可恢复）
- 通过 OpenClaw cron 任务自动维护（重建索引、归档、整合）

---

## 目录

- [系统概览](#系统概览)
- [Memory 文件结构](#memory-文件结构)
- [信息生命周期](#信息生命周期)
- [自动化维护（Cron）](#自动化维护cron)
- [多 Agent 注意事项](#多-agent-注意事项)
- [组件](#组件)
- [快速参考](#快速参考)

---

## 系统概览

```
┌──────────────────────────────────────────────────────────────┐
│                       Agent（LLM）                            │
│                                                              │
│  memory_search ──► qmd（向量 + BM25 混合检索）                │
│  memory_get    ──► 直接读取文件（L0 → L2 导航）               │
│  lcm_grep      ──► LosslessClaw SQLite DAG                   │
│  lcm_expand    ──► 从压缩历史中恢复细节                        │
└───────────────────────────┬──────────────────────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                                   │
    ┌─────▼──────┐                   ┌────────▼──────┐
    │  qmd 索引  │                   │    lcm.db      │
    │（SQLite +  │                   │（SQLite DAG    │
    │  向量）    │                   │  压缩摘要）     │
    └─────┬──────┘                   └───────────────┘
          │
    ┌─────▼──────────────────────┐
    │   磁盘上的 Markdown 文件    │
    │   memory/  ·  blackboard/  │
    └────────────────────────────┘
```

| 组件 | 职责 | Agent 工具 |
|------|------|-----------|
| **Memory 文件** | 结构化精选事实 — 真相来源 | `read`, `write`, `edit` |
| **qmd** | 对 memory 文件做语义+关键词搜索 | `memory_search`, `memory_get` |
| **LosslessClaw** | 无损压缩历史对话 | `lcm_grep`, `lcm_expand`, `lcm_expand_query` |
| **Cron 任务** | 自动化重建索引、归档和整合 | 后台定时任务 |

---

## Memory 文件结构

三层层级结构使检索成本与需求成正比：

```
memory/
├── MEMORY.md               ← L0：极简索引，只有路径指针（~30 行）
├── INDEX.md                ← L1：分类概览与导航
├── user/
│   ├── profile.md          # 身份、背景、身体数据
│   ├── preferences/        # 学习、生活、技术、沟通偏好
│   │   ├── learning.md
│   │   ├── lifestyle.md
│   │   ├── tech.md
│   │   └── communication.md
│   ├── entities/           # 使用中的工具、人物、服务
│   │   ├── tools.md
│   │   └── people.md
│   └── events/             # 重要决策与里程碑 — 只增不改
│       └── YYYY-MM-名称.md
└── agent/
    ├── cases/              # 首次处理的新类型任务 — 只增不改
    │   └── 任务名.md
    └── patterns/           # 可复用处理规律
        └── 规律名.md
```

### 检索顺序

```
Agent 读 MEMORY.md（L0）      ← 永远从这里开始；~30 行，极快
  → 定位分类路径
  → memory_get(L2 文件, from, lines)   ← 只读所需内容
  → 位置不明时用 memory_search          ← 作为回退
```

### 六分类写入

每条信息只存在一个地方：

| 分类 | 文件 | 写入规则 |
|------|------|---------|
| 用户身份/背景 | `user/profile.md` | 可追加 |
| 偏好/习惯 | `user/preferences/[主题].md` | 可追加 |
| 项目/工具/人物 | `user/entities/[类型].md` | 可更新 |
| 重要决策/里程碑 | `user/events/YYYY-MM-[名称].md` | **只增不改** |
| 首次处理的新类型任务 | `agent/cases/[名称].md` | **只增不改** |
| 可复用处理规律 | `agent/patterns/[名称].md` | 可追加 |

---

## 信息生命周期

```
对话发生
    │
    ├──► Agent 将结构化事实写入 memory/ 文件
    │              │
    │         qmd 重新索引（gateway 运行时每 5 分钟，
    │                        或 gateway 启动时）
    │
    └──► 上下文窗口满
              │
        LosslessClaw 将较旧消息压缩为 DAG 节点
              │
         节点存入 lcm.db
         （永不删除 — 随时可通过 lcm_grep / lcm_expand 恢复）
```

**qmd 回答：** *「我们对 X 了解什么？」* — 搜索精选 memory 文件  
**LosslessClaw 回答：** *「我们讨论过 X 的什么？」* — 搜索压缩的历史对话

---

## 自动化维护（Cron）

通过一组定时后台任务保持记忆健康。这些任务以隔离的 OpenClaw cron session 运行，不污染主聊天上下文。

### 记忆相关 Cron 任务

#### Dream Cycle — 每周记忆整合
`cron: 0 8 * * 0`（周日 08:00 Asia/Shanghai）

扫描 `memory/` 根目录的日期格式 session 日志，将每个文件精炼为 ≤30 行结构化摘要，将原文件移入 `memory/archive/YYYY-MM/`，并对 `patterns/` 去重。保持工作记忆目录简洁。

```json5
{
  "name": "Dream Cycle (Memory Consolidation)",
  "schedule": { "kind": "cron", "expr": "0 8 * * 0", "tz": "Asia/Shanghai" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "执行记忆整合：扫描 memory/ 根目录日期格式 session 日志，精炼为 ≤30 行结构化摘要，归档原文件至 memory/archive/YYYY-MM/，对 patterns/ 去重。"
  },
  "delivery": { "mode": "announce", "channel": "last" }
}
```

#### 每日进度同步
`cron: 0 4 * * *`（04:00 Asia/Shanghai）

读取 `blackboard/REGISTRY.md` 和昨日日历事件，同步项目进度到 Blackboard 项目卡。即使在安静的一天之后也保持项目状态最新。

#### 每月 Session 清理
`cron: 0 3 1 * *`（每月 1 日 03:00 Asia/Shanghai）

将 `memory/` 根目录超过 7 天的 session 日志归档到 `memory/archive/YYYY-MM/`。执行滚动日志保留策略。

#### Provider 配额告警
`every: 6h`

向每个已配置的模型 provider 发送简短 ping。报告失败并检查 memory 中的配额追踪。在提供商故障影响活跃 session 之前捕获问题。

### Cron 配置示例

```json5
// openclaw.json — cron 部分
{
  "cron": {
    "enabled": true,
    "store": "~/.openclaw/cron/jobs.json"
  }
}
```

通过 CLI 添加任务：

```bash
# 每周记忆整合（周日 08:00）
openclaw cron add \
  --name "Dream Cycle (Memory Consolidation)" \
  --cron "0 8 * * 0" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "执行记忆整合：扫描 memory/ 根目录日期格式 session 日志，精炼为 ≤30 行结构化摘要，归档原文件。" \
  --announce

# 每月清理（每月 1 日 03:00）
openclaw cron add \
  --name "Monthly Session Cleanup" \
  --cron "0 3 1 * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "将 memory/ 根目录超过 7 天的 session 日志归档到 memory/archive/YYYY-MM/。" \
  --announce

# 查看所有计划任务
openclaw cron list
```

---

## 多 Agent 注意事项

在多 agent 体系中，所有 agent 共享同一套 memory 文件，但在 `lcm.db` 中各有独立的 session 历史。

### 子 Agent 写入规则

- 按六分类规范写入对应 `memory/` 文件
- 每次写入末尾追加 `_写入者：[agent-id] YYYY-MM-DD_`
- 通知编排 agent，由编排 agent 同步 L0/L1 索引

### 信任模型

| 资源 | 访问权限 |
|------|---------|
| Memory 文件（`memory/`） | 共享真相 — 任何 agent 可读；按分类规则写入 |
| Blackboard（`blackboard/`） | 共享项目状态 — 按 `blackboard/_schema.md` 读写 |
| LosslessClaw 历史（`lcm.db`） | 按 session 隔离 — 需要时用 `lcm_grep` 加 session 过滤 |

### 防止级联漂移

当任何共享事实变更（agent 配置、工具分配、项目状态、协议规则）时，搜索所有引用该信息的文件并同步更新。用 `memory_search` 对变更关键词搜索来找到受影响的文件。

---

## 组件

### Agent Memory Protocol（可安装 skill）

所有 agent 遵循的写入/读取协议。定义六分类、去重策略、级联更新规则、会话反思和冲写检查清单。

```bash
clawhub install agent-memory-protocol
```

→ [`skills/memory-manager/SKILL.md`](skills/memory-manager/SKILL.md)

### qmd — 语义搜索

本地混合搜索引擎（BM25 + 向量 embedding）。驱动 `memory_search`。

→ [`docs/qmd.zh-CN.md`](docs/qmd.zh-CN.md) — 安装、collection 配置、OpenClaw 接入、CLI 速查

### LosslessClaw — 上下文压缩

DAG 分层摘要系统，替代有损滑动窗口截断。历史对话保持可恢复。

→ [`docs/losslessclaw.zh-CN.md`](docs/losslessclaw.zh-CN.md) — 安装、参数、检索模式

### 完整架构参考

→ [`docs/architecture.zh-CN.md`](docs/architecture.zh-CN.md)

---

## 快速参考

| 我想... | 工具 / 命令 |
|---------|-----------|
| 在记忆中查找事实 | `memory_search("query")` |
| 读取特定文件段落 | `memory_get(path, from, lines)` |
| 搜索历史对话 | `lcm_grep("pattern")` |
| 恢复过去的决策 | `lcm_expand_query("X 当时决定了什么")` |
| 强制重建 qmd 索引 | `qmd update`（CLI） |
| 检查 qmd 健康状态 | `qmd status`（CLI） |
| 查看摘要节点 | `lcm_describe("sum_xxx")` |
| 查看计划任务 | `openclaw cron list` |
| 手动执行 cron 任务 | `openclaw cron run <jobId> --force` |

---

## 相关链接

- [OpenClaw](https://github.com/openclaw/openclaw) — 核心 Gateway
- [OpenClaw 文档](https://docs.openclaw.ai)
- [ClawHub: agent-memory-protocol](https://clawhub.ai/OttoPrua/agent-memory-protocol) — 安装 skill
- [qmd](https://github.com/tobilen/qmd) — 快速 Markdown 搜索
- [LosslessClaw](https://github.com/martian-engineering/lossless-claw) — 无损上下文管理
- [Discord](https://discord.gg/clawd) — OpenClaw 社区

## 许可证

MIT
