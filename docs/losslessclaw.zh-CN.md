# LosslessClaw — 配置指南

<p align="center">
  <a href="losslessclaw.md">English</a> · <a href="losslessclaw.zh-CN.md">中文</a>
</p>

[LosslessClaw](https://github.com/martian-engineering/lossless-claw) 将 OpenClaw 默认的滑动窗口截断替换为 DAG 分层摘要系统。上下文满时，将较旧的对话压缩为摘要树，存储在本地 SQLite 数据库中。内容不丢弃 — 压缩但不删除，随时可恢复。

## 安装

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

安装后重启 gateway：

```bash
openclaw gateway restart
```

## 配置

在 `~/.openclaw/openclaw.json` 中：

```json5
{
  "plugins": {
    "allow": ["lossless-claw"],
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "summaryProvider": "anthropic",
          "summaryModel": "claude-haiku-4-5",  // 推荐用便宜快速的模型
          "freshTailCount": 32,                // 最近 N 条消息保持原文不压缩
          "contextThreshold": 0.75,           // 上下文达到 75% 时触发压缩
          "ignoreSessionPatterns": [
            "agent:*:cron:**"                 // cron session 不压缩
          ],
          "incrementalMaxDepth": 10           // DAG 最大深度
        }
      }
    }
  }
}
```

### 参数说明

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `summaryProvider` | `"anthropic"` | 摘要模型的 provider |
| `summaryModel` | `"claude-haiku-4-5"` | 推荐便宜快速的模型，如 haiku 或 gemini-flash |
| `freshTailCount` | `32` | 最近 N 条消息保持原文，不压缩 |
| `contextThreshold` | `0.75` | 触发压缩的上下文占比 |
| `ignoreSessionPatterns` | `["agent:*:cron:**"]` | 跳过压缩的 session 通配符 |
| `incrementalMaxDepth` | `10` | DAG 最大深度，超出强制全量汇总 |

## 工作原理

```
正常对话进行
  → 上下文增长
  → 达到 contextThreshold
  → LosslessClaw 将较旧消息压缩为 DAG 节点
  → 节点存入 lcm.db
  → 最近 freshTailCount 条消息保持原文在上下文中
  → 未来的压缩可以引用已有 DAG 节点（树状结构）
```

DAG 结构支持摘要的摘要 — 长期运行的 agent 会积累一个可导航的历史树，而不是有损的滑动窗口。

## 可用工具

| 工具 | 功能 |
|------|------|
| `lcm_grep` | 在所有压缩摘要中做正则/全文搜索 |
| `lcm_describe` | 按 ID 查看特定摘要节点（sum_xxx） |
| `lcm_expand` | 展开摘要树，恢复对话细节 |
| `lcm_expand_query` | 针对问题展开相关摘要并通过子 agent 给出聚焦回答 |

## 检索模式

**从旧 session 中找到某件事：**
```
lcm_grep("微信触发符")
  → 返回匹配的摘要 ID 和片段

lcm_expand(summaryIds=["sum_abc123"])
  → 返回该对话段的压缩内容
```

**从历史中回答问题：**
```
lcm_expand_query(
  query="触发符最终改成了什么",
  prompt="最终决定是什么，为什么"
)
  → 委托子 agent 展开摘要并回答
  → 返回聚焦回答 + 引用的摘要 ID
```

**查看特定摘要：**
```
lcm_describe("sum_abc123")
  → 返回摘要内容、谱系、token 数量
```

## 何时用 LosslessClaw vs qmd

| 场景 | 用哪个 |
|------|--------|
| 「用户对 X 的偏好是什么？」 | `memory_search` → qmd |
| 「上周我们决定了什么？」 | `lcm_grep` → `lcm_expand` |
| 「当前项目状态？」 | `memory_search` → qmd（blackboard） |
| 「为什么当时改了方案？」 | `lcm_expand_query` |
| 「讨论过但从未写入 memory 的内容？」 | `lcm_grep` → `lcm_expand` |

## 数据库

LosslessClaw 将所有摘要存储在 `~/.openclaw/lcm.db`（SQLite）。

```bash
# 查看数据库大小
ls -lh ~/.openclaw/lcm.db
```

数据库随 session 积累而增长，OpenClaw 自动管理，无需手动维护。
