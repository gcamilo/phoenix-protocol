# Phoenix Protocol v2.0

**A Resilient Session Recovery Pattern for AI Agents**

*Battle-tested across 7 autonomous agents running 24/7 for 30+ days.*

> **Disclaimer:** This protocol is an emergent artifact of AI-assisted development. It does not represent official technical standards of Anthropic, OpenAI, or Google.

---

## Abstract

Phoenix Protocol is a design pattern for building resilient AI agent systems that survive context window exhaustion, session crashes, and unexpected terminations. Instead of depending on platform-specific session resume features, it treats sessions as ephemeral and recovers from externalized state.

**Key insight**: If your agent depends solely on the conversation transcript to know what to do, your externalized state is insufficient.

**v2.0 insight**: Don't ask agents to maintain their own state. They forget. Automate it.

---

## Prerequisites

- **bash** (or compatible shell) for hook scripts
- **jq** for JSON manipulation in hooks
- **python3** for atomic state updates
- A filesystem with atomic rename support (most POSIX systems)
- An AI agent platform that supports session hooks (or manual recovery workflow)
- **tmux** (for multi-agent deployments with watchdog)

---

## The Problem

AI agents running in persistent sessions face several failure modes:

1. **Context Window Exhaustion**: Long-running sessions fill their context window, causing automatic compaction or termination
2. **Platform Resume Failures**: Session resume features can be opaque or brittle
3. **Unexpected Crashes**: Network issues, system restarts, or tool failures can terminate sessions mid-task
4. **State Confusion**: Resuming a stale session can leave agents confused about current state
5. **Stale Open Loops**: Without structured tracking, resolved items persist indefinitely in briefs (v2.0)
6. **Watchdog/Wrapper Conflicts**: Recovery systems can fight each other without coordination signals (v2.0)

---

## Core Principles

1. **State is External**: All task state lives in files (`state.json`, `brief.md`), not in conversation history
2. **Recovery is Default**: Every session start assumes it might be a recovery situation
3. **Context is Injected**: Fresh sessions receive full context through hooks, not transcript replay
4. **Graceful Degradation**: System works even if session resume fails completely
5. **Automation Over Ceremony**: State updates must not depend on the AI model remembering to do them (v2.0)
6. **Hybrid Resume**: Use platform resume as primary, Phoenix as fallback — not either/or (v2.0)

---

## Architecture

### Required Components

```
┌─────────────────────────────────────────────────────────┐
│                    AI Agent Session                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │ Start Hook  │───▶│   Agent     │───▶│  End Hook   │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                  │                  │         │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │
          ▼                  ▼                  ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │state.json│      │ brief.md │      │ ops.jsonl│
    └──────────┘      └──────────┘      └──────────┘
          │                                    │
          ▼                                    ▼
    ┌──────────────┐                   ┌──────────────┐
    │resolved.jsonl│                   │  watchdog    │
    └──────────────┘                   └──────────────┘
```

### 1. State File (`state.json`)

Tracks current task, open loops, and key metrics.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `agent` | string | Agent/domain name |
| `last_active` | ISO8601 | Timestamp of last activity (hook-updated) |

**Recommended fields (v2.0):**

| Field | Type | Description |
|-------|------|-------------|
| `open_loops` | array | Tracked items awaiting resolution |
| `resolved` | array | Recently resolved items (7-day retention) |
| `numbers` | object | Key metrics as string key-value pairs |

**Open loop schema:**

```json
{
  "id": "kebab-case-stable-id",
  "text": "Human-readable description",
  "added": "2026-02-15",
  "stale": false
}
```

Items older than 14 days are automatically flagged `"stale": true`.

**Resolved item schema:**

```json
{
  "id": "kebab-case-stable-id",
  "reason": "Why this was resolved",
  "resolved_date": "2026-02-17"
}
```

**Full example:**

```json
{
  "agent": "trades",
  "last_active": "2026-02-17T02:12:53Z",
  "open_loops": [
    {
      "id": "schwab-oauth",
      "text": "Schwab OAuth connection not set up",
      "added": "2026-02-01",
      "stale": true
    },
    {
      "id": "scotus-ieepa",
      "text": "SCOTUS IEEPA ruling ~Feb 20 — binary event",
      "added": "2026-02-14",
      "stale": false
    }
  ],
  "resolved": [
    {
      "id": "snapshot-failure",
      "reason": "Fixed cron schedule, Feb 16 run confirmed",
      "resolved_date": "2026-02-16"
    }
  ],
  "numbers": {
    "nav": "$266,691",
    "mtd_pnl": "-$3,846",
    "active_crons": "4"
  }
}
```

### 2. Brief File (`brief.md`)

Human-readable context summary. Updated daily by automated brief generation, or manually on significant changes.

```markdown
trades — 2026-02-16

**Changed:**
- Built multi-asset Merton share analysis across 11 asset classes
- IBKR snapshot cron restored

**Open loops:**
- Portfolio rebalance: 92% VTI concentration vs optimal 50-60%
- SCOTUS IEEPA ruling ~Feb 20

**Next:** Process Feb 16 snapshot, update NAV.

**Risk:** 92% VTI with margin into SCOTUS week.

**Numbers:** NAV $266,691 | MTD PnL -$3,846
```

**Source of truth hierarchy:** `state.json` > `brief.md` > conversation history

### 3. Resolution Log (`resolved.jsonl`) — v2.0

Append-only log for marking items done between brief generation cycles.

```jsonl
{"id": "snapshot-failure", "reason": "Fixed cron", "ts": "2026-02-16T10:00:00Z", "domain": "trades"}
```

A `resolve` command provides a convenient interface:

```bash
resolve "loop-id" "reason"
# Appends to resolved.jsonl, updates state.json immediately
```

Processed during daily brief generation, then archived.

### 4. Session Hooks

#### Start Hook (Recovery Detection + Brief Injection)

```bash
#!/bin/bash
INPUT=$(cat)
CWD=$(echo "$INPUT" | jq -r '.cwd // ""')

# Determine domain from cwd...
STATE_FILE="$HOME/$DOMAIN/state.json"

# Recovery detection
if [ -f "$STATE_FILE" ]; then
    LAST_STATUS=$(jq -r '.status // "unknown"' "$STATE_FILE" 2>/dev/null)
    if [ "$LAST_STATUS" = "working" ]; then
        LAST_TASK=$(jq -r '.current_task // "unknown"' "$STATE_FILE" 2>/dev/null)
        OPEN_LOOPS=$(jq -r '.open_loops | length // 0' "$STATE_FILE" 2>/dev/null)
        echo "=== RECOVERY DETECTED ==="
        echo "You were interrupted mid-task. Last known state:"
        echo "  Task: $LAST_TASK"
        echo "  Open loops: $OPEN_LOOPS"
        echo "Review your brief below and continue where you left off."
        echo "==========================="
    fi
fi

# Inject brief
BRIEF="$HOME/$DOMAIN/brief.md"
[ -f "$BRIEF" ] && [ -s "$BRIEF" ] && cat "$BRIEF"
```

#### Activity Hook (Heartbeat)

Updates `last_active` on every tool call. Runs in background to avoid blocking.

```bash
#!/bin/bash
# Triggered by PostToolUse hook

STATE_FILE="$HOME/$DOMAIN/state.json"
if [ -f "$STATE_FILE" ]; then
    python3 -c "
import json, os, tempfile
from datetime import datetime, timezone
f = '$STATE_FILE'
try:
    s = json.load(open(f))
    s['last_active'] = datetime.now(timezone.utc).isoformat()
    fd, t = tempfile.mkstemp(dir=os.path.dirname(f))
    os.fdopen(fd,'w').write(json.dumps(s, indent=2))
    os.rename(t, f)
except: pass
" &
fi
```

#### End Hook (Event Logging)

```bash
#!/bin/bash
echo "{\"ts\":\"$(date -Iseconds)\",\"event\":\"session_end\",\"status\":\"ok\"}" >> ops.jsonl
```

---

## Agent Wrapper (Crash Recovery)

The wrapper manages agent lifecycle with hybrid resume + Phoenix fallback:

1. **Try `--resume`** with saved session ID (primary path)
2. **On crash**: clear session ID, wait cooldown, restart fresh
3. **On clean exit (code 0)**: write `.clean-exit` marker, enter safe mode
4. **After max crashes**: stop retrying, enter safe mode, alert

**Key design decisions:**
- Always rebuild command from `BASE_CMD` to prevent `--resume` flag accumulation
- Safe mode = `while read` loop (not `exec bash` — prevents shell injection)
- Fresh-start marker (`.fresh`) allows intentional clean-slate restarts

---

## Watchdog (External Monitor)

Runs on cron (every 15 minutes). Validates agent health by checking **process existence**, not state files.

```bash
has_ai_process() {
    local pane_pid=$(tmux list-panes -t "$1" -F '#{pane_pid}' 2>/dev/null | head -1)
    [ -z "$pane_pid" ] && return 1
    pstree -p "$pane_pid" 2>/dev/null | grep -qE '(claude|gemini|codex)\('
}
```

**Clean-exit coordination (v2.0):**

| Scenario | `.clean-exit` exists? | Watchdog action |
|----------|----------------------|-----------------|
| Agent running | N/A | Skip (healthy) |
| Agent dead, clean exit | Yes | Skip (intentional) |
| Agent dead, no marker | No | Auto-restart |
| Intentional restart | Cleared by restart script | Restart proceeds |

---

## Automated Brief Generation (v2.0)

**Core v2.0 innovation: Don't ask agents to write their own briefs.**

A daily cron job reads agent transcripts and generates both `brief.md` and `state.json`:

1. Read previous state.json + resolved.jsonl for context
2. LLM reads recent transcript
3. LLM outputs brief text + JSON separated by `===STATE_JSON===` marker
4. Script parses, validates JSON, merges with existing state (preserving `last_active`)
5. Archives processed resolved.jsonl entries

**Why this works:** The LLM doing brief generation has access to the full transcript and can accurately identify what changed, what's still open, and what got resolved. The agent itself often forgets to update state because it's focused on the task, not on meta-tracking.

---

## What Didn't Work (Lessons Learned)

### `state-update` CLI (Removed in v2.0)

v1.x included `state-update --working "task"` for agents to declare status. **Agents didn't use it.** They'd forget, get distracted, or crash before calling it.

**Fix:** Hook-based heartbeat + watchdog process checking.

### Context Exhaustion Warnings (Dropped)

v1.x proposed warning at 85% context. **Hooks don't receive context percentage in their payload.** Modern platforms handle compaction internally.

**Fix:** Make recovery so fast that compaction is a non-event.

### Pure Phoenix (No Resume)

v1.0 proposed abandoning `--resume` entirely. In practice, `--resume` **works ~80% of the time** and provides much better continuity.

**Fix:** Hybrid approach — resume first, Phoenix as fallback.

---

## Best Practices

### DO:
- Use hooks for automatic state updates
- Automate brief generation externally
- Run a watchdog that checks process existence
- Use atomic writes for all state updates
- Use `--resume` as primary, Phoenix as fallback
- Coordinate clean exit vs crash with marker files
- Track open loops with stable IDs and staleness flags
- Keep briefs concise but complete

### DON'T:
- Depend on agents to maintain their own state
- Depend solely on platform resume features
- Store essential state only in conversation history
- Trust state files without process verification
- Use `exec bash` in crash handlers (shell injection risk)
- Skip the clean-exit handshake between wrapper and watchdog

---

## Version History

- **v2.0** (2026-02-17): Production-hardened rewrite. Structured state (open_loops, resolved, numbers), automated brief generation, resolve command, staleness detection, clean-exit coordination, hybrid resume. Removed state-update CLI and context warnings.
- **v1.2** (2026-02-02): Hook-based state updates, heartbeat pattern, watchdog validation
- **v1.1** (2026-02-02): Prerequisites, Security, Concurrency sections
- **v1.0** (2026-02-02): Initial specification

---

## Contributors

- **Claude** (Anthropic) — Implementation, v2.0 spec
- **Codex** (OpenAI) — Root cause analysis, code review
- **Gemini** (Google) — "Phoenix" metaphor, automated brief generation design

## License

CC0 (Public Domain) — Use freely.
