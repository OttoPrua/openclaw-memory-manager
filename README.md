# OpenClaw Memory Architecture

<p align="center">
  <picture>
    <img src="docs/assets/banner.svg" alt="OpenClaw Memory Architecture" width="480">
  </picture>
</p>

<p align="center">
  <strong>A complete memory system for OpenClaw multi-agent setups</strong><br>
  Structured file hierarchy · semantic search · lossless context compression · automated maintenance
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
  <img src="https://img.shields.io/badge/license-MIT-blue?style=for-the-badge" alt="License">
</p>

[OpenClaw](https://github.com/openclaw/openclaw) agents accumulate context across many sessions. Without a system, memory fragments: facts drift out of sync across files, decisions vanish after context compaction, and agents waste tokens reading everything to find one thing. This repository describes a complete architecture that solves all three problems.

**It covers:**
- A structured Markdown file hierarchy agents write to (the source of truth)
- [qmd](https://github.com/tobilen/qmd) — local semantic search over those files (powers `memory_search`)
- [LosslessClaw](https://github.com/martian-engineering/lossless-claw) — DAG-based context compression (past conversations stay recoverable)
- Automated maintenance via OpenClaw cron jobs (re-indexing, archival, consolidation)

---

## Contents

- [System Overview](#system-overview)
- [Memory File Structure](#memory-file-structure)
- [Information Lifecycle](#information-lifecycle)
- [Automated Maintenance (Cron)](#automated-maintenance-cron)
- [Multi-Agent Considerations](#multi-agent-considerations)
- [Components](#components)
- [Quick Reference](#quick-reference)

---

## System Overview

```
┌──────────────────────────────────────────────────────────────┐
│                       Agent (LLM)                            │
│                                                              │
│  memory_search ──► qmd  (vector + BM25 hybrid search)        │
│  memory_get    ──► direct file read  (L0 → L2 navigation)    │
│  lcm_grep      ──► LosslessClaw SQLite DAG                   │
│  lcm_expand    ──► recover detail from compressed history    │
└───────────────────────────────┬──────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                                   │
        ┌─────▼──────┐                   ┌────────▼──────┐
        │  qmd index │                   │    lcm.db      │
        │ (SQLite +  │                   │ (SQLite DAG    │
        │  vectors)  │                   │  summaries)    │
        └─────┬──────┘                   └───────────────┘
              │
        ┌─────▼──────────────────────┐
        │   Markdown files on disk   │
        │   memory/  ·  blackboard/  │
        └────────────────────────────┘
```

| Component | Purpose | Agent tools |
|-----------|---------|------------|
| **Memory files** | Structured, curated facts — the source of truth | `read`, `write`, `edit` |
| **qmd** | Semantic + keyword search over memory files | `memory_search`, `memory_get` |
| **LosslessClaw** | Lossless compression of past conversation history | `lcm_grep`, `lcm_expand`, `lcm_expand_query` |
| **Cron jobs** | Automated re-indexing, archival, and consolidation | scheduled background tasks |

---

## Memory File Structure

A three-layer hierarchy keeps retrieval cost proportional to need:

```
memory/
├── MEMORY.md               ← L0: minimal index, path pointers only (~30 lines)
├── INDEX.md                ← L1: category overview and navigation
├── user/
│   ├── profile.md          # Identity, background, body data
│   ├── preferences/        # Learning, lifestyle, tech, communication prefs
│   │   ├── learning.md
│   │   ├── lifestyle.md
│   │   ├── tech.md
│   │   └── communication.md
│   ├── entities/           # Tools, people, services in use
│   │   ├── tools.md
│   │   └── people.md
│   └── events/             # Key decisions and milestones — append-only
│       └── YYYY-MM-name.md
└── agent/
    ├── cases/              # First-time task type records — append-only
    │   └── task-name.md
    └── patterns/           # Reusable handling patterns
        └── pattern-name.md
```

### Retrieval sequence

```
Agent reads MEMORY.md (L0)      ← always start here; ~30 lines, fast
  → locates category path
  → memory_get(L2 file, from, lines)   ← reads only the needed section
  → falls back to memory_search        ← when location is unknown
```

### Six write categories

Every piece of information goes to exactly one place:

| Category | File | Write rule |
|----------|------|-----------|
| User identity / background | `user/profile.md` | Appendable |
| Preferences / habits | `user/preferences/[topic].md` | Appendable |
| Projects / tools / people | `user/entities/[type].md` | Updatable |
| Key decisions / milestones | `user/events/YYYY-MM-[name].md` | **Append-only, never modify** |
| First time handling a task type | `agent/cases/[name].md` | **Append-only, never modify** |
| Reusable handling patterns | `agent/patterns/[name].md` | Appendable |

---

## Information Lifecycle

```
Conversation happens
       │
       ├──► Agent writes structured facts to memory/ files
       │              │
       │         qmd re-indexes (every 5 min while gateway runs,
       │                          or on gateway boot)
       │
       └──► Context window fills up
                  │
            LosslessClaw compresses older messages into a DAG node
                  │
             node stored in lcm.db
             (never deleted — always recoverable via lcm_grep / lcm_expand)
```

**qmd answers:** *"What do we know about X?"* — searches curated memory files  
**LosslessClaw answers:** *"What did we discuss about X?"* — searches compressed conversation history

---

## Automated Maintenance (Cron)

Memory stays healthy through a set of scheduled background jobs. These run as isolated OpenClaw cron sessions and do not pollute the main chat context.

### Memory-related cron jobs

#### Dream Cycle — Weekly Memory Consolidation
`cron: 0 8 * * 0` (Sunday 08:00 Asia/Shanghai)

Scans the `memory/` root for dated session log files, refines each into a ≤30-line structured summary, moves originals to `memory/archive/YYYY-MM/`, and deduplicates `patterns/`. Keeps the working memory directory lean.

```json5
{
  "name": "Dream Cycle (Memory Consolidation)",
  "schedule": { "kind": "cron", "expr": "0 8 * * 0", "tz": "Asia/Shanghai" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Run memory consolidation: scan memory/ root for dated session logs, summarize each to ≤30 lines, archive originals to memory/archive/YYYY-MM/, deduplicate patterns/."
  },
  "delivery": { "mode": "announce", "channel": "last" }
}
```

#### Daily Progress Sync
`cron: 0 4 * * *` (04:00 Asia/Shanghai)

Reads `blackboard/REGISTRY.md` and yesterday's calendar events to sync project progress to Blackboard project cards. Ensures project state stays current even after quiet days.

#### Monthly Session Cleanup
`cron: 0 3 1 * *` (1st of month, 03:00 Asia/Shanghai)

Archives session log files older than 7 days from `memory/` root to `memory/archive/YYYY-MM/`. Enforces the rolling log retention policy.

#### Provider Quota Alert
`every: 6h`

Sends a brief ping to each configured model provider. Reports failures and checks memory for quota tracking. Catches provider outages before they affect active sessions.

### Sample cron configuration

```json5
// openclaw.json — cron section
{
  "cron": {
    "enabled": true,
    "store": "~/.openclaw/cron/jobs.json"
  }
}
```

Add jobs via CLI:

```bash
# Weekly memory consolidation (Sunday 08:00)
openclaw cron add \
  --name "Dream Cycle (Memory Consolidation)" \
  --cron "0 8 * * 0" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "Run memory consolidation: scan memory/ root for dated session logs, summarize each to ≤30 lines, archive originals to memory/archive/YYYY-MM/, deduplicate patterns/." \
  --announce

# Monthly cleanup (1st of month, 03:00)
openclaw cron add \
  --name "Monthly Session Cleanup" \
  --cron "0 3 1 * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "Archive memory/ root session logs older than 7 days to memory/archive/YYYY-MM/." \
  --announce

# View all scheduled jobs
openclaw cron list
```

---

## Multi-Agent Considerations

In a multi-agent stack, all agents share the same memory files but have distinct session histories in `lcm.db`.

### Write rules for sub-agents

- Write structured facts to the appropriate `memory/` category file
- Append `_Written by: [agent-id] YYYY-MM-DD_` at the end of each write
- Notify the orchestrating agent; it syncs the L0/L1 indexes

### Trust model

| Resource | Access |
|----------|--------|
| Memory files (`memory/`) | Shared truth — any agent can read; write per category rules |
| Blackboard (`blackboard/`) | Shared project state — read/write per `blackboard/_schema.md` |
| LosslessClaw history (`lcm.db`) | Per-session — use `lcm_grep` with session filters when needed |

### Preventing cascading drift

When any shared fact changes (agent config, tool assignments, project status, protocol rules), search for all files that reference it and update them. Use `memory_search` on the changed keyword to find affected files.

---

## Components

### Agent Memory Protocol (installable skill)

The write/read protocol all agents follow. Defines the six categories, dedup strategy, cascade update rules, session reflection, and flush checklist.

```bash
clawhub install agent-memory-protocol
```

→ [`skills/memory-manager/SKILL.md`](skills/memory-manager/SKILL.md)

### qmd — Semantic Search

Local hybrid search engine (BM25 + vector embeddings) over Markdown files. Powers `memory_search`.

→ [`docs/qmd.md`](docs/qmd.md) — installation, collection setup, OpenClaw wiring, CLI reference

### LosslessClaw — Context Compression

DAG-based summarization that replaces lossy sliding-window truncation. Past conversations remain recoverable via `lcm_grep` / `lcm_expand`.

→ [`docs/losslessclaw.md`](docs/losslessclaw.md) — installation, parameters, retrieval patterns

### Full Architecture Reference

→ [`docs/architecture.md`](docs/architecture.md)

---

## Quick Reference

| I want to... | Tool / Command |
|-------------|---------------|
| Find a fact in memory | `memory_search("query")` |
| Read a specific file section | `memory_get(path, from, lines)` |
| Search past conversations | `lcm_grep("pattern")` |
| Recover a past decision | `lcm_expand_query("what was decided about X")` |
| Force re-index memory files | `qmd update` (CLI) |
| Check qmd index health | `qmd status` (CLI) |
| Inspect a summary node | `lcm_describe("sum_xxx")` |
| List scheduled cron jobs | `openclaw cron list` |
| Run a cron job manually | `openclaw cron run <jobId> --force` |

---

## Related

- [OpenClaw](https://github.com/openclaw/openclaw) — the core gateway
- [OpenClaw Docs](https://docs.openclaw.ai) — full documentation
- [ClawHub: agent-memory-protocol](https://clawhub.ai/OttoPrua/agent-memory-protocol) — install the skill
- [qmd](https://github.com/tobilen/qmd) — quick markdown search
- [LosslessClaw](https://github.com/martian-engineering/lossless-claw) — lossless context management
- [Discord](https://discord.gg/clawd) — OpenClaw community

## License

MIT
