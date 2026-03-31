# LosslessClaw — Setup & Configuration Guide

<p align="center">
  <a href="losslessclaw.md">English</a> · <a href="losslessclaw.zh-CN.md">中文</a>
</p>

[LosslessClaw](https://github.com/martian-engineering/lossless-claw) replaces OpenClaw's default sliding-window context truncation with a DAG-based summarization system. When context fills up, older exchanges are compressed into a tree of summaries stored in a local SQLite database. Nothing is thrown away — old conversations are compressed, not deleted, and remain recoverable.

## Installation

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

After installation, restart the gateway:

```bash
openclaw gateway restart
```

## Configuration

In `~/.openclaw/openclaw.json`:

```json5
{
  "plugins": {
    "allow": ["lossless-claw"],
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "summaryProvider": "anthropic",
          "summaryModel": "claude-haiku-4-5",  // use a cheap fast model
          "freshTailCount": 32,                // keep last N messages verbatim
          "contextThreshold": 0.75,           // compact when context hits 75%
          "ignoreSessionPatterns": [
            "agent:*:cron:**"                 // skip cron sessions
          ],
          "incrementalMaxDepth": 10           // DAG depth limit
        }
      }
    }
  }
}
```

### Parameter Reference

| Parameter | Recommended | Notes |
|-----------|------------|-------|
| `summaryProvider` | `"anthropic"` | Provider for the summarization model |
| `summaryModel` | `"claude-haiku-4-5"` | Use cheap + fast; haiku or gemini-flash recommended |
| `freshTailCount` | `32` | Recent N messages stay verbatim, not summarized |
| `contextThreshold` | `0.75` | Fraction of context window that triggers compaction |
| `ignoreSessionPatterns` | `["agent:*:cron:**"]` | Glob patterns for sessions to skip |
| `incrementalMaxDepth` | `10` | Max DAG depth before forcing a full roll-up summary |

## How It Works

```
Normal conversation
  → context grows
  → hits contextThreshold
  → LosslessClaw summarizes older messages into a DAG node
  → DAG node stored in lcm.db
  → recent freshTailCount messages stay verbatim in context
  → future compactions can reference previous DAG nodes (tree structure)
```

The DAG structure means summaries of summaries are possible — long-running agents accumulate a navigable history tree rather than a lossy rolling window.

## Available Tools

| Tool | What it does |
|------|-------------|
| `lcm_grep` | Regex/full-text search across all compressed summaries |
| `lcm_describe` | Look up a specific summary node by ID (sum_xxx) |
| `lcm_expand` | Expand a summary tree to recover conversation detail |
| `lcm_expand_query` | Answer a question by expanding relevant summaries via sub-agent |

## Retrieval Patterns

**Find something from a past session:**
```
lcm_grep("wechat trigger symbol")
  → returns matching summary IDs and snippets
  
lcm_expand(summaryIds=["sum_abc123"])
  → returns the compressed content of that conversation segment
```

**Answer a question from history:**
```
lcm_expand_query(
  query="what was decided about the wechat trigger symbol",
  prompt="what was the final decision and why"
)
  → delegates expansion + Q&A to a sub-agent
  → returns focused answer with cited summary IDs
```

**Inspect a specific summary:**
```
lcm_describe("sum_abc123")
  → returns summary content, lineage, token counts
```

## When to Use LosslessClaw vs qmd

| Scenario | Use |
|---------|-----|
| "What's the user's preference on X?" | `memory_search` → qmd |
| "What did we decide last week?" | `lcm_grep` → `lcm_expand` |
| "Current project status?" | `memory_search` → qmd (blackboard) |
| "Why did we change approach on X?" | `lcm_expand_query` |
| "What was discussed but never written to memory?" | `lcm_grep` → `lcm_expand` |

## Database

LosslessClaw stores all summaries in `~/.openclaw/lcm.db` (SQLite).

```bash
# Check database size
ls -lh ~/.openclaw/lcm.db
```

The database grows over time as sessions accumulate. OpenClaw manages it automatically; no manual maintenance required.
