# OpenClaw Memory Architecture

<p align="center">
  <strong>A multi-agent memory system for OpenClaw — structured files, semantic search, and lossless context compression</strong>
</p>

<p align="center">
  <a href="README.md">English</a> · <a href="README.zh-CN.md">中文</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/OpenClaw-compatible-orange?style=for-the-badge" alt="OpenClaw">
  <img src="https://img.shields.io/badge/clawhub-agent--memory--protocol-brightgreen?style=for-the-badge" alt="ClawHub">
  <img src="https://img.shields.io/badge/license-MIT-blue?style=for-the-badge" alt="License">
</p>

This repository documents a complete memory architecture for [OpenClaw](https://github.com/openclaw/openclaw) multi-agent systems. It covers three layers: the structured file layout agents write to, the semantic search engine that retrieves from it, and the lossless compression system that preserves conversation history beyond context limits.

---

## The Problem

As agents handle more tasks across longer sessions, memory breaks down in predictable ways:

- The same fact lives in multiple files and drifts out of sync
- Important decisions get buried in session logs and lost after context compaction
- Agents waste tokens reading everything instead of navigating to the right file
- Old conversations become inaccessible once compressed away

This architecture addresses all four.

---

## System Overview

```
┌───────────────────────────────────────────────────────────┐
│                      Agent (LLM)                          │
│                                                           │
│  memory_search ──► qmd (vector + BM25 hybrid search)      │
│  memory_get    ──► direct file read (L0 → L2 navigation)  │
│  lcm_grep      ──► LosslessClaw SQLite DAG                │
│  lcm_expand    ──► recover detail from compressed history  │
└───────────────────────────────────────────────────────────┘
         │                              │
    ┌────▼──────────┐          ┌────────▼──────────┐
    │  qmd index    │          │  lcm.db            │
    │  (SQLite +    │          │  (SQLite DAG        │
    │   vectors)    │          │   summaries)        │
    └────┬──────────┘          └────────────────────┘
         │
    ┌────▼───────────────────────────┐
    │  Markdown files on disk        │
    │  memory/  ·  blackboard/       │
    └────────────────────────────────┘
```

**Three components, one pipeline:**

| Component | Role | Tools |
|-----------|------|-------|
| **Memory files** | Structured, curated facts — the source of truth | `read`, `write`, `edit` |
| **qmd** | Semantic + keyword search over memory files | `memory_search`, `memory_get` |
| **LosslessClaw** | Lossless compression of past conversations | `lcm_grep`, `lcm_expand`, `lcm_expand_query` |

---

## Memory File Structure

Agents write to a three-layer hierarchy:

```
memory/
├── MEMORY.md               ← L0: minimal index, path pointers only (~30 lines)
├── INDEX.md                ← L1: category overview and navigation
├── user/
│   ├── profile.md          # Identity, background, physical data
│   ├── preferences/        # Learning, lifestyle, tech, communication prefs
│   ├── entities/           # Tools, people, services
│   └── events/             # Key decisions and milestones (append-only)
└── agent/
    ├── cases/              # First-time task type records (append-only)
    └── patterns/           # Reusable handling patterns
```

**The L0 → L1 → L2 retrieval sequence keeps token cost proportional to need:**

```
Agent reads MEMORY.md (L0)  →  locates category path
  → memory_get the L2 file  →  reads only what's needed
  → memory_search as fallback when location is unknown
```

**Six write categories — every piece of information goes to exactly one place:**

| Category | File | Rules |
|----------|------|-------|
| User identity / background | `user/profile.md` | Appendable |
| Preferences / habits | `user/preferences/[topic].md` | Appendable |
| Projects / tools / people | `user/entities/[type].md` | Updatable |
| Key decisions / events | `user/events/YYYY-MM-[name].md` | Append-only, never modify |
| New task type handled | `agent/cases/[name].md` | Append-only, never modify |
| Reusable patterns | `agent/patterns/[name].md` | Appendable |

---

## Information Lifecycle

```
Conversation happens
       │
       ├──► Agent writes structured facts ──► memory/ files
       │                                           │
       │                                     qmd re-indexes
       │                                    (every 5 min or on change)
       │
       └──► Context fills up
                  │
            LosslessClaw compresses old messages into DAG node
                  │
             stored in lcm.db (never deleted)
```

- **qmd** answers: *"What do we know about X?"* — searches curated memory files
- **LosslessClaw** answers: *"What did we discuss about X?"* — searches compressed conversation history

---

## Multi-Agent Considerations

In a multi-agent stack, all agents share the same memory files but have distinct session histories in lcm.db.

**Write rules for sub-agents:**
- Write structured facts directly to the appropriate `memory/` file per the six-category spec
- Note `_Written by: [agent-id] YYYY-MM-DD_` at the end of each write
- Notify the orchestrating agent; it syncs the L0/L1 indexes

**Trust model:**
- Memory files are shared truth — any agent can read, authorized agents can write
- LosslessClaw histories are per-session — use `lcm_grep` with session filters when needed

---

## Components

### Agent Memory Protocol (installable skill)

The write/read protocol all agents follow. Defines the six categories, dedup strategy, cascade update rules, session reflection, and flush checklist.

```bash
clawhub install agent-memory-protocol
```

→ [skills/memory-manager/SKILL.md](skills/memory-manager/SKILL.md)

### qmd

Local semantic search engine. Hybrid BM25 + vector index over your Markdown files. Powers `memory_search`.

→ [docs/qmd.md](docs/qmd.md) — setup and configuration guide

### LosslessClaw

DAG-based context compression. Replaces lossy sliding-window truncation. All past conversations remain recoverable via `lcm_grep` / `lcm_expand`.

→ [docs/losslessclaw.md](docs/losslessclaw.md) — setup and configuration guide

### Full Architecture Reference

→ [docs/architecture.md](docs/architecture.md)

---

## Quick Reference

| I want to... | Tool |
|-------------|------|
| Find a fact in memory | `memory_search("query")` |
| Read a specific file section | `memory_get(path, from, lines)` |
| Search past conversations | `lcm_grep("pattern")` |
| Recover a past decision | `lcm_expand_query("what was decided about X")` |
| Force re-index memory | `qmd update` (CLI) |
| Check qmd health | `qmd status` (CLI) |
| Inspect a summary node | `lcm_describe("sum_xxx")` |

---

## Related

- [OpenClaw](https://github.com/openclaw/openclaw) — the core gateway
- [OpenClaw Docs](https://docs.openclaw.ai)
- [ClawHub: agent-memory-protocol](https://clawhub.ai/OttoPrua/agent-memory-protocol)
- [qmd](https://github.com/tobilen/qmd)
- [LosslessClaw](https://github.com/martian-engineering/lossless-claw)
- [Discord](https://discord.gg/clawd)

## License

MIT
