# Cron Jobs — Reference Configurations

<p align="center">
  <a href="README.md">English</a> · <a href="README.zh-CN.md">中文</a>
</p>

Reference configurations for OpenClaw cron jobs that maintain memory health. Each `.json` file can be used directly with `openclaw cron add` or as a template for the `cron.add` tool call.

## Jobs

| File | Schedule | Description |
|------|----------|-------------|
| [`dream-cycle.json`](dream-cycle.json) | Weekly (Sun 08:00) | Memory consolidation — summarize session logs, archive, deduplicate patterns |
| [`daily-progress-sync.json`](daily-progress-sync.json) | Daily (04:00) | Sync project progress from calendar to Blackboard |
| [`monthly-cleanup.json`](monthly-cleanup.json) | Monthly (1st, 03:00) | Archive old session logs from memory/ root |

> Schedules use `Asia/Shanghai` timezone. Adjust `tz` and `expr` for your timezone.

## Usage

### Add via CLI

```bash
# Read the JSON, then pass fields to the CLI
# Example: Dream Cycle
openclaw cron add \
  --name "Dream Cycle (Memory Consolidation)" \
  --cron "0 8 * * 0" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "Run memory consolidation: scan memory/ root for dated session logs, refine each to ≤30 lines, archive originals to memory/archive/YYYY-MM/, deduplicate patterns/." \
  --announce
```

### Add via tool call (agent)

```json
{
  "name": "Dream Cycle (Memory Consolidation)",
  "schedule": { "kind": "cron", "expr": "0 8 * * 0", "tz": "Asia/Shanghai" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Run memory consolidation: scan memory/ root for dated session logs, refine each to ≤30 lines, archive originals to memory/archive/YYYY-MM/, deduplicate patterns/."
  },
  "delivery": { "mode": "announce", "channel": "last" }
}
```

### Verify

```bash
openclaw cron list
openclaw cron run <jobId> --force   # manual test run
openclaw cron runs --id <jobId>     # view run history
```
