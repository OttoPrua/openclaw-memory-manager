# qmd — Setup & Configuration Guide

<p align="center">
  <a href="qmd.md">English</a> · <a href="qmd.zh-CN.md">中文</a>
</p>

[qmd](https://github.com/tobilen/qmd) is the local semantic search engine that powers `memory_search` in OpenClaw. It builds a hybrid index (BM25 full-text + vector embeddings) over your `memory/` and `blackboard/` directories.

## Installation

```bash
# Recommended: bun
bun install -g @tobilu/qmd

# Or: npm
npm install -g @tobilu/qmd

# Verify
qmd --version
which qmd   # note this path for openclaw.json
```

## Configure Collections

Collections define which folders qmd indexes. Set them up once after installation:

```bash
# Index your memory directory
qmd collection add memory-root /path/to/workspace/memory --pattern "**/*.md"

# Index the blackboard (project state)
qmd collection add blackboard /path/to/workspace/blackboard --pattern "**/*.md"

# Verify
qmd collection list
qmd status
```

You can add multiple collections and filter by collection name at search time (`-c memory-root`).

## Wire into OpenClaw

In `~/.openclaw/openclaw.json`:

```json5
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "command": "/path/to/qmd",     // from: which qmd
      "searchMode": "vsearch",       // see Search Modes below
      "includeDefaultMemory": true,
      "update": {
        "interval": "5m",            // auto re-index every 5 min while gateway runs
        "onBoot": true,              // re-index on gateway start
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

## Search Modes

| Mode | When to use |
|------|------------|
| `vsearch` | **Default.** Semantic similarity — good for concepts, fuzzy recall |
| `query` | Best for research — auto-expands query + reranks results |
| `search` | BM25 keyword only — exact term matches, no LLM expansion |

## How Agents Use It

```
memory_search("project deadline")
  → qmd vsearch over memory/ + blackboard/
  → returns top-N snippets with file path + line numbers
  → agent calls memory_get(path, from, lines) to read exact section
```

Agents should always read L0 (`MEMORY.md`) first to locate the right file, then use `memory_get` for precise reads. Use `memory_search` when the location is unknown.

## CLI Reference

```bash
# Semantic search
qmd vsearch "project deadline"

# Research-quality search with reranking
qmd query "what models does the agent use" -c memory-root

# Keyword search
qmd search "claude-haiku" -c memory-root

# Read a specific file
qmd get qmd://memory-root/user/profile.md

# Filter by collection
qmd vsearch "wechat config" -c blackboard

# Check health
qmd status

# Force re-index (after adding files outside gateway)
qmd update

# Rebuild vector embeddings
qmd embed -f

# List indexed files
qmd ls memory-root
qmd ls blackboard
```

## Troubleshooting

**New files not appearing in search results:**
```bash
qmd update   # re-index
```

**Semantic search quality degraded:**
```bash
qmd embed -f  # force rebuild embeddings
```

**Check what's indexed:**
```bash
qmd status
qmd ls memory-root
```
