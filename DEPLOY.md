---
name: deploy-memory-manager
description: Interactive deployment guide for the OpenClaw Memory Stack. Installs qmd (semantic search), LosslessClaw (context compression), initializes the memory directory structure, and optionally adds maintenance cron jobs.
---

# Deploy: OpenClaw Memory Stack

> **Interactive protocol.** Each step checks current state before acting. Pause and confirm with the user between major phases.

---

## Before Starting

```bash
# Check prerequisites
which git && git --version
which python3 && python3 --version
node --version 2>/dev/null || bun --version 2>/dev/null
openclaw --version
```

Ask the user:
> "What is your OpenClaw workspace path? (default: `~/.openclaw/workspace`)"

Use `<WORKSPACE>` as the placeholder throughout this guide.

---

## Phase 1 — qmd (Semantic Search)

### 1a: Install

```bash
# Check if already installed
which qmd && qmd --version && echo "already installed"
```

If not installed:
```bash
# Recommended: bun
bun install -g @tobilu/qmd

# Alternative: npm
npm install -g @tobilu/qmd
```

### 1b: Create collections

```bash
# Check existing collections
qmd collection list
```

Add only collections that don't already exist:

```bash
qmd collection add memory-root <WORKSPACE>/memory --pattern "**/*.md"
qmd collection add blackboard <WORKSPACE>/blackboard --pattern "**/*.md"

# Build initial index
qmd update
qmd status
```

Verify:
```bash
qmd vsearch "test" 2>/dev/null | head -3 && echo "✅ qmd search working"
```

### 1c: Wire into openclaw.json

Check if already configured:
```bash
cat ~/.openclaw/openclaw.json | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print('configured' if d.get('memory',{}).get('backend')=='qmd' else 'not configured')"
```

If not configured, get the qmd binary path:
```bash
which qmd
```

Show the user this block to add to `~/.openclaw/openclaw.json`:

```json5
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "command": "<output of: which qmd>",
      "searchMode": "vsearch",
      "includeDefaultMemory": true,
      "update": {
        "interval": "5m",
        "onBoot": true,
        "waitForBootSync": false
      },
      "limits": {
        "maxResults": 10,
        "maxSnippetChars": 500
      },
      "scope": { "default": "allow" }
    }
  }
}
```

> "Add this block to `openclaw.json`, replace `command` with the path above. Tell me when done."

Wait for confirmation.

---

## Phase 2 — LosslessClaw (Context Compression)

### 2a: Install

```bash
# Check if already installed
cat ~/.openclaw/openclaw.json | python3 -c \
  "import sys,json; d=json.load(sys.stdin); lc=d.get('plugins',{}).get('installs',{}).get('lossless-claw'); print('installed:', lc.get('version') if lc else 'not found')"
```

If not installed:
```bash
openclaw plugins install @martian-engineering/lossless-claw
```

### 2b: Configure

Ask: "Which model for summarization? (default: `claude-haiku-4-5` — fast and cheap; alternatives: `google/gemini-3-flash-preview`, `openai/gpt-4o-mini`)"

Wait for answer, then show config block to add under `plugins.entries` in `openclaw.json`:

```json5
{
  "plugins": {
    "allow": ["lossless-claw"],
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "summaryProvider": "anthropic",
          "summaryModel": "<chosen model>",
          "freshTailCount": 32,
          "contextThreshold": 0.75,
          "ignoreSessionPatterns": ["agent:*:cron:**"],
          "incrementalMaxDepth": 10
        }
      }
    }
  }
}
```

> "Add this block to `openclaw.json`. Tell me when done."

Wait for confirmation.

### 2c: Restart gateway

```bash
openclaw gateway restart
```

Verify:
```bash
openclaw gateway status
ls -lh ~/.openclaw/lcm.db 2>/dev/null && echo "✅ lcm.db created"
```

---

## Phase 3 — Memory Manager Skill

### 3a: Install via ClawHub

```bash
# Check if already installed
ls <WORKSPACE>/skills/memory-manager/SKILL.md 2>/dev/null && echo "already installed"
```

If not installed:
```bash
cd <WORKSPACE>
clawhub install agent-memory-protocol
```

### 3b: Initialize memory directory structure

```bash
ls <WORKSPACE>/memory/MEMORY.md 2>/dev/null && echo "already exists"
```

If not exists:
```bash
mkdir -p <WORKSPACE>/memory/user/preferences
mkdir -p <WORKSPACE>/memory/user/entities
mkdir -p <WORKSPACE>/memory/user/events
mkdir -p <WORKSPACE>/memory/agent/cases
mkdir -p <WORKSPACE>/memory/agent/patterns
mkdir -p <WORKSPACE>/memory/archive

cat > <WORKSPACE>/MEMORY.md << 'EOF'
# Memory Index (L0)
> Full contents in memory/INDEX.md; this file holds high-frequency entry points only.

## User
- Profile → memory/user/profile.md
- Preferences → memory/user/preferences/

## Agent
- Cases → memory/agent/cases/
- Patterns → memory/agent/patterns/
EOF
```

---

## Phase 4 — Maintenance Cron Jobs (optional)

Ask: "Do you want to add automated memory maintenance cron jobs? (y/n)"

If yes, ask: "What timezone? (e.g. `America/New_York`, `Europe/London`, `Asia/Tokyo`)"

```bash
# Check existing jobs
openclaw cron list
```

Add only jobs not already listed:

**Weekly memory consolidation (Dream Cycle):**
```bash
openclaw cron add \
  --name "Dream Cycle (Memory Consolidation)" \
  --cron "0 8 * * 0" \
  --tz "<TIMEZONE>" \
  --session isolated \
  --message "Run memory consolidation: scan memory/ root for dated session log files, refine each into a ≤30-line structured summary, move originals to memory/archive/YYYY-MM/, deduplicate patterns/." \
  --announce
```

**Monthly session cleanup:**
```bash
openclaw cron add \
  --name "Monthly Session Cleanup" \
  --cron "0 3 1 * *" \
  --tz "<TIMEZONE>" \
  --session isolated \
  --message "Archive memory/ root session log files (YYYY-MM-DD.md format) older than 7 days to memory/archive/YYYY-MM/." \
  --announce
```

---

## Final Check

```bash
qmd status
openclaw gateway status
openclaw cron list
ls <WORKSPACE>/skills/memory-manager/SKILL.md && echo "✅ skill installed"
ls <WORKSPACE>/memory/ && echo "✅ memory structure ready"
```

| Component | Status |
|-----------|--------|
| qmd index | ✅/❌ |
| LosslessClaw | ✅/❌ |
| Memory Manager skill | ✅/❌ |
| Memory directory | ✅/❌ |
| Cron jobs | N registered |

→ Full docs: https://github.com/OttoPrua/openclaw-memory-manager
