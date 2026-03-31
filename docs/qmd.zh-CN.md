# qmd — 配置指南

<p align="center">
  <a href="qmd.md">English</a> · <a href="qmd.zh-CN.md">中文</a>
</p>

[qmd](https://github.com/tobilen/qmd) 是驱动 OpenClaw `memory_search` 的本地语义搜索引擎，对 `memory/` 和 `blackboard/` 目录建立混合索引（BM25 全文 + 向量 embedding）。

## 安装

```bash
# 推荐：bun
bun install -g @tobilu/qmd

# 或：npm
npm install -g @tobilu/qmd

# 验证
qmd --version
which qmd   # 记录路径，填入 openclaw.json
```

## 配置 Collection

Collection 定义 qmd 索引哪些目录，安装后一次性配置：

```bash
# 索引 memory 目录
qmd collection add memory-root /path/to/workspace/memory --pattern "**/*.md"

# 索引 blackboard（项目状态）
qmd collection add blackboard /path/to/workspace/blackboard --pattern "**/*.md"

# 验证
qmd collection list
qmd status
```

可以添加多个 collection，搜索时用 `-c collection名` 过滤。

## 接入 OpenClaw

在 `~/.openclaw/openclaw.json` 中配置：

```json5
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "command": "/path/to/qmd",     // 运行 which qmd 获取
      "searchMode": "vsearch",       // 见下方搜索模式说明
      "includeDefaultMemory": true,
      "update": {
        "interval": "5m",            // Gateway 运行时每 5 分钟自动重建索引
        "onBoot": true,              // Gateway 启动时重建索引
        "waitForBootSync": false
      },
      "limits": {
        "maxResults": 10,
        "maxSnippetChars": 500
      },
      "scope": {
        "default": "allow"
      }
    }
  }
}
```

## 搜索模式

| 模式 | 适用场景 |
|------|---------|
| `vsearch` | **默认。** 语义相似度 — 适合概念查找、模糊召回 |
| `query` | 研究类最佳 — 自动扩展 query + 重排序 |
| `search` | BM25 关键词 — 精确词汇匹配，无 LLM 扩展 |

## Agent 如何使用

```
memory_search("项目截止日期")
  → qmd vsearch 扫描 memory/ + blackboard/
  → 返回 Top-N 片段（含文件路径 + 行号）
  → agent 调用 memory_get(path, from, lines) 读取具体内容
```

Agent 应先读 L0（`MEMORY.md`）定位目标文件，再用 `memory_get` 精准读取。位置不明时才用 `memory_search`。

## CLI 速查

```bash
# 语义搜索
qmd vsearch "项目截止日期"

# 研究级搜索（自动扩展+重排序）
qmd query "agent 使用什么模型" -c memory-root

# 关键词搜索
qmd search "claude-haiku" -c memory-root

# 读取特定文件
qmd get qmd://memory-root/user/profile.md

# 按 collection 过滤
qmd vsearch "微信配置" -c blackboard

# 检查健康状态
qmd status

# 强制重建索引（在 gateway 外新增文件后运行）
qmd update

# 重建向量 embedding
qmd embed -f

# 查看已索引文件
qmd ls memory-root
qmd ls blackboard
```

## 故障排查

**新文件未出现在搜索结果中：**
```bash
qmd update   # 重建索引
```

**语义搜索质量下降：**
```bash
qmd embed -f  # 强制重建 embedding
```

**查看已索引内容：**
```bash
qmd status
qmd ls memory-root
```
