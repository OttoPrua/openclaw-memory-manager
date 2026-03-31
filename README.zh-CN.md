# OpenClaw 记忆架构

<p align="center">
  <strong>OpenClaw 多 agent 记忆系统 — 结构化文件、语义搜索与无损上下文压缩</strong>
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.zh-CN.md">中文</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/OpenClaw-compatible-orange?style=for-the-badge" alt="OpenClaw">
  <img src="https://img.shields.io/badge/clawhub-agent--memory--protocol-brightgreen?style=for-the-badge" alt="ClawHub">
  <img src="https://img.shields.io/badge/许可证-MIT-blue?style=for-the-badge" alt="License">
</p>

本仓库记录了一套完整的 [OpenClaw](https://github.com/openclaw/openclaw) 多 agent 记忆架构。涵盖三个层次：agent 写入的结构化文件体系、从中检索的语义搜索引擎，以及在上下文限制之外保存对话历史的无损压缩系统。

---

## 问题所在

随着 agent 跨越更长的 session 处理更多任务，记忆会以可预期的方式崩溃：

- 同一事实存在多个文件中并逐渐不同步
- 重要决策被埋在 session 日志里，上下文压缩后丢失
- Agent 浪费 token 通读所有内容，而不是直接定位到正确的文件
- 旧对话一旦被压缩就变得不可访问

这套架构解决上述全部四个问题。

---

## 系统概览

```
┌───────────────────────────────────────────────────────────┐
│                      Agent（LLM）                          │
│                                                           │
│  memory_search ──► qmd（向量 + BM25 混合检索）              │
│  memory_get    ──► 直接读取文件（L0 → L2 导航）             │
│  lcm_grep      ──► LosslessClaw SQLite DAG                │
│  lcm_expand    ──► 从压缩历史中恢复细节                     │
└───────────────────────────────────────────────────────────┘
         │                              │
    ┌────▼──────────┐          ┌────────▼──────────┐
    │  qmd 索引     │          │  lcm.db            │
    │  （SQLite +   │          │  （SQLite DAG       │
    │   向量）      │          │   压缩摘要）         │
    └────┬──────────┘          └────────────────────┘
         │
    ┌────▼───────────────────────────┐
    │  磁盘上的 Markdown 文件         │
    │  memory/  ·  blackboard/       │
    └────────────────────────────────┘
```

**三个组件，一条流水线：**

| 组件 | 职责 | 工具 |
|------|------|------|
| **Memory 文件** | 结构化精选事实 — 真相来源 | `read`, `write`, `edit` |
| **qmd** | 对 memory 文件做语义+关键词搜索 | `memory_search`, `memory_get` |
| **LosslessClaw** | 无损压缩历史对话 | `lcm_grep`, `lcm_expand`, `lcm_expand_query` |

---

## Memory 文件结构

Agent 写入三层层级结构：

```
memory/
├── MEMORY.md               ← L0：极简索引，只有路径指针（~30 行）
├── INDEX.md                ← L1：分类概览与导航
├── user/
│   ├── profile.md          # 身份、背景、身体数据
│   ├── preferences/        # 学习、生活、技术、沟通偏好
│   ├── entities/           # 工具、人物、服务
│   └── events/             # 重要决策与里程碑（只增不改）
└── agent/
    ├── cases/              # 首次处理的新类型任务（只增不改）
    └── patterns/           # 可复用处理规律
```

**L0 → L1 → L2 的检索顺序使 token 消耗与需求成正比：**

```
Agent 读 MEMORY.md（L0） → 定位分类路径
  → memory_get 读 L2 文件 → 只读所需内容
  → 位置不明时用 memory_search 作为回退
```

**六分类写入 — 每条信息只存在一个地方：**

| 分类 | 文件 | 规则 |
|------|------|------|
| 用户身份/背景 | `user/profile.md` | 可追加 |
| 偏好/习惯 | `user/preferences/[主题].md` | 可追加 |
| 项目/工具/人物 | `user/entities/[类型].md` | 可更新 |
| 重要决策/事件 | `user/events/YYYY-MM-[名称].md` | 只增不改 |
| 首次处理的新类型任务 | `agent/cases/[名称].md` | 只增不改 |
| 可复用规律 | `agent/patterns/[名称].md` | 可追加 |

---

## 信息生命周期

```
对话发生
    │
    ├──► Agent 写入结构化事实 ──► memory/ 文件
    │                                   │
    │                             qmd 重新索引
    │                            （每 5 分钟或变更时）
    │
    └──► 上下文满
              │
        LosslessClaw 将旧消息压缩为 DAG 节点
              │
         存入 lcm.db（永不删除）
```

- **qmd** 回答：*「我们对 X 了解什么？」* — 搜索精选 memory 文件
- **LosslessClaw** 回答：*「我们讨论过 X 的什么内容？」* — 搜索压缩的对话历史

---

## 多 Agent 注意事项

在多 agent 体系中，所有 agent 共享同一套 memory 文件，但在 lcm.db 中各有独立的 session 历史。

**子 agent 写入规则：**
- 按六分类规范直接写入对应 `memory/` 文件
- 每次写入末尾注明 `_写入者：[agent-id] YYYY-MM-DD_`
- 通知编排 agent，由编排 agent 同步 L0/L1 索引

**信任模型：**
- Memory 文件是共享真相 — 任何 agent 可读，授权 agent 可写
- LosslessClaw 历史按 session 隔离 — 需要时用 `lcm_grep` 加 session 过滤

---

## 组件

### Agent Memory Protocol（可安装 skill）

所有 agent 遵循的写入/读取协议。定义六分类、去重策略、级联更新规则、会话反思和冲写检查清单。

```bash
clawhub install agent-memory-protocol
```

→ [skills/memory-manager/SKILL.md](skills/memory-manager/SKILL.md)

### qmd

本地语义搜索引擎。对 Markdown 文件建立 BM25 + 向量混合索引，驱动 `memory_search`。

→ [docs/qmd.zh-CN.md](docs/qmd.zh-CN.md) — 配置指南

### LosslessClaw

DAG 分层上下文压缩，替代有损滑动窗口截断。所有历史对话均可通过 `lcm_grep` / `lcm_expand` 恢复。

→ [docs/losslessclaw.zh-CN.md](docs/losslessclaw.zh-CN.md) — 配置指南

### 完整架构参考

→ [docs/architecture.zh-CN.md](docs/architecture.zh-CN.md)

---

## 快速参考

| 我想... | 工具 |
|---------|------|
| 在记忆中查找事实 | `memory_search("query")` |
| 读取特定文件段落 | `memory_get(path, from, lines)` |
| 搜索历史对话 | `lcm_grep("pattern")` |
| 恢复过去的决策 | `lcm_expand_query("X 当时决定了什么")` |
| 强制重建 qmd 索引 | `qmd update`（CLI） |
| 检查 qmd 健康状态 | `qmd status`（CLI） |
| 查看摘要节点 | `lcm_describe("sum_xxx")` |

---

## 相关链接

- [OpenClaw](https://github.com/openclaw/openclaw) — 核心 Gateway
- [OpenClaw 文档](https://docs.openclaw.ai)
- [ClawHub: agent-memory-protocol](https://clawhub.ai/OttoPrua/agent-memory-protocol)
- [qmd](https://github.com/tobilen/qmd)
- [LosslessClaw](https://github.com/martian-engineering/lossless-claw)
- [Discord](https://discord.gg/clawd)

## 许可证

MIT
