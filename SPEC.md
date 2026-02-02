# Phoenix Protocol

**A Resilient Session Recovery Pattern for AI Agents**

*Pattern synthesized from multi-agent debugging sessions involving Claude, Codex, and Gemini.*

> **Disclaimer:** This protocol is an emergent artifact of AI-assisted development. It does not represent official technical standards of Anthropic, OpenAI, or Google.

---

## Abstract

Phoenix Protocol is a design pattern for building resilient AI agent systems that can survive context window exhaustion, session crashes, and unexpected terminations. Instead of depending on platform-specific session resume features, Phoenix Protocol treats sessions as ephemeral and ensures recovery through externalized state.

The key insight: **If your agent depends solely on the conversation transcript to know what to do, your externalized state is insufficient.**

---

## Prerequisites

Before implementing Phoenix Protocol, ensure you have:

- **bash** (or compatible shell) for hook scripts
- **jq** for JSON manipulation in hooks
- A filesystem with atomic rename support (most POSIX systems)
- An AI agent platform that supports session hooks (or manual recovery workflow)

---

## The Problem

AI agents running in persistent sessions face several failure modes:

1. **Context Window Exhaustion**: Long-running sessions eventually fill their context window, causing automatic compaction or termination
2. **Platform Resume Failures**: Session resume features can be opaque or brittle, failing without clear diagnostics
3. **Unexpected Crashes**: Network issues, system restarts, or tool failures can terminate sessions mid-task
4. **State Confusion**: Resuming a stale session can leave agents confused about current state

Traditional approaches try to "fix" resume or recover transcripts. This is fragile.

---

## The Solution: Treat Sessions as Ephemeral

Phoenix Protocol inverts the problem. Instead of making sessions more durable, make recovery trivially easy:

### Core Principles

1. **State is External**: All task state lives in files (`state.json`, `brief.md`), not in conversation history
2. **Recovery is Default**: Every session start assumes it might be a recovery situation
3. **Context is Injected**: Fresh sessions receive full context through hooks, not transcript replay
4. **Graceful Degradation**: System works even if session resume fails completely

### The Phoenix Metaphor

Like the phoenix rising from ashes, a new session can be instantly productive because:
- The "ashes" (state files) contain everything needed
- No dependency on the dead session's memories
- Fresh context eliminates drift and hallucination loops often found in long context windows (though may lose some nuance)

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
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
                  Persistent State (Filesystem)
```

### 1. State File (`state.json`)

Tracks current task and status.

**Required fields:**

| Field | Type | Description |
|-------|------|-------------|
| `status` | enum | `idle`, `working`, or `error` |
| `current_task` | string | Description of current/last task |
| `last_active` | ISO8601 | Timestamp of last state update |

**Optional fields:**

| Field | Type | Description |
|-------|------|-------------|
| `last_output` | string | Summary of last meaningful output |
| `error_message` | string | Error details (when status=error) |

**Example:**

```json
{
  "status": "working",
  "current_task": "Implementing user authentication",
  "last_output": "Added JWT validation middleware",
  "last_active": "2026-02-02T10:30:00Z"
}
```

### 2. Brief File (`brief.md`)

Human-readable context summary updated periodically. This is the "warm handoff" document for a fresh session.

```markdown
## Current State
- Working on: User authentication feature
- Progress: JWT middleware complete, testing pending

## Open Loops
- Need to add refresh token logic
- Integration tests not written

## Next Actions
- Write integration tests for auth endpoints
```

**Source of truth:** When `brief.md` and `state.json` conflict, `state.json` is authoritative for status/task. `brief.md` provides richer context.

### 3. Session Hooks

#### Start Hook (Recovery Detection)

```bash
#!/bin/bash
# session-start-hook.sh
# Requires: jq

STATE_FILE="${PHOENIX_STATE_DIR:-./state.json}"

if [[ -f "$STATE_FILE" ]]; then
    LAST_STATUS=$(jq -r '.status // "unknown"' "$STATE_FILE")
    LAST_TASK=$(jq -r '.current_task // "unknown"' "$STATE_FILE")

    if [[ "$LAST_STATUS" == "working" || "$LAST_STATUS" == "error" ]]; then
        echo "╔════════════════════════════════════════════════════════════════╗"
        echo "║                    RECOVERY DETECTED                           ║"
        echo "╠════════════════════════════════════════════════════════════════╣"
        echo "║ Status: $LAST_STATUS"
        echo "║ Last task: $LAST_TASK"
        echo "╚════════════════════════════════════════════════════════════════╝"
        echo ""
        cat "$STATE_FILE"
    fi
fi
```

#### End Hook (State Preservation)

```bash
#!/bin/bash
# session-end-hook.sh

STATE_FILE="${PHOENIX_STATE_DIR:-./state.json}"
OPS_LOG="${PHOENIX_OPS_LOG:-./ops.jsonl}"

# Log session end
echo "{\"ts\":\"$(date -Iseconds)\",\"event\":\"session_end\",\"status\":\"ok\"}" >> "$OPS_LOG"

# State should already be saved during normal operation
# This hook catches unexpected exits
```

### 4. Ops Log (`ops.jsonl`)

Event stream for observability and alerting:

```json
{"ts": "2026-02-02T10:00:00Z", "event": "session_start", "status": "ok"}
{"ts": "2026-02-02T10:30:00Z", "event": "task_started", "meta": {"task": "auth"}}
{"ts": "2026-02-02T12:00:00Z", "event": "session_end", "status": "ok"}
```

**Required fields:** `ts` (ISO8601), `event` (string), `status` (`ok`|`warn`|`error`)

---

## Implementation Guide

### Step 1: Add State Management

**Reference implementation** of `state-update`:

```bash
#!/bin/bash
# state-update: Update state.json atomically
# Usage: state-update --working "task" | --done "summary" | --error "message" | --idle

STATE_FILE="${PHOENIX_STATE_DIR:-./state.json}"

update_state() {
    local status="$1"
    local task="$2"
    local tmp=$(mktemp)

    # Create or update state atomically
    if [[ -f "$STATE_FILE" ]]; then
        jq --arg s "$status" --arg t "$task" --arg ts "$(date -Iseconds)" \
           '.status=$s | .current_task=$t | .last_active=$ts' "$STATE_FILE" > "$tmp"
    else
        echo "{\"status\":\"$status\",\"current_task\":\"$task\",\"last_active\":\"$(date -Iseconds)\"}" > "$tmp"
    fi

    # Atomic rename (prevents partial writes)
    mv "$tmp" "$STATE_FILE"
}

case "$1" in
    --working) update_state "working" "$2" ;;
    --done)    update_state "idle" "$2" ;;
    --error)   update_state "error" "$2" ;;
    --idle)    update_state "idle" "" ;;
    *)         echo "Usage: state-update --working|--done|--error|--idle [message]" ;;
esac
```

### Step 2: Configure Session Hooks

Register hooks with your AI agent platform. Configuration varies by platform.

**Example (pseudo-config):**

```json
{
  "hooks": {
    "SessionStart": ["./scripts/session-start-hook.sh"],
    "SessionEnd": ["./scripts/session-end-hook.sh"]
  }
}
```

Consult your platform's documentation for exact syntax.

### Step 3: Generate Briefs Regularly

Update brief on significant progress or at session checkpoints:

```bash
# Generate brief summarizing current state
./scripts/generate-brief.sh > brief.md
```

### Step 4: Test Recovery Flow

1. Start a session and begin a task
2. Mark state as "working": `state-update --working "Test task"`
3. Kill the session (simulate crash)
4. Start new session
5. Verify recovery context is injected

---

## Concurrency & Safety

### Atomic Writes

**Critical:** Always use atomic write patterns for state files to prevent corruption from partial writes or concurrent access.

```bash
# CORRECT: Write to temp, then atomic rename
tmp=$(mktemp)
echo '{"status":"working"}' > "$tmp"
mv "$tmp" state.json

# WRONG: Direct write (can corrupt on crash)
echo '{"status":"working"}' > state.json
```

### Multi-Agent Collisions

If multiple agents share state files:

1. Use file locking (`flock`) or a coordination service
2. Or give each agent its own state directory
3. Consider a database instead of flat files for high-concurrency scenarios

### Corruption Recovery

If `state.json` becomes invalid:

1. The `ops.jsonl` log provides event history for reconstruction
2. `brief.md` provides human-readable context
3. Version control (git) provides rollback capability

---

## Security

### Secrets Management

**Never store secrets in state files.** State files may be:
- Committed to version control
- Logged or transmitted
- Read by other agents

Use environment variables or a secrets manager for:
- API keys
- Credentials
- Tokens

### Path Security

Use absolute paths or environment variables to avoid path confusion:

```bash
# Recommended: Use environment variable
export PHOENIX_STATE_DIR="/home/user/myagent"

# Or: Use absolute paths in scripts
STATE_FILE="/home/user/myagent/state.json"
```

### Sensitive Content in Briefs

If briefs may contain sensitive information:
- Add redaction rules before sharing
- Mark sensitive sections clearly
- Consider encrypting state at rest

---

## State Management

### The Reliability Problem

A critical failure mode in early implementations: relying on the AI model to "remember" to update state files. Models forget, get distracted, or crash before updating. **Model-driven state updates are unreliable by design.**

### Principle: Automatic State via Hooks

**State updates MUST NOT depend on the AI model's volition.**

Instead, use runtime hooks that execute automatically:

| Hook | Trigger | State Update |
|------|---------|--------------|
| `SessionStart` | Session begins | `status → working` |
| `PostToolUse` | After any tool call | `last_active → now` (heartbeat) |
| `SessionEnd` | Session ends cleanly | `status → idle` |

The model MAY still call state-update for human-readable notes (e.g., `current_task` descriptions), but this is supplementary, not authoritative.

### Heartbeat Pattern

The `PostToolUse` hook provides an automatic heartbeat:

```bash
#!/bin/bash
# activity-hook.sh - Update last_active on every tool call

STATE_FILE="./state.json"

# Fast, atomic timestamp update (runs in background to avoid blocking)
python3 -c "
import json, os, tempfile
from datetime import datetime, timezone
try:
    with open('$STATE_FILE') as f:
        state = json.load(f)
    state['last_active'] = datetime.now(timezone.utc).isoformat()
    fd, tmp = tempfile.mkstemp(dir=os.path.dirname('$STATE_FILE'))
    with os.fdopen(fd, 'w') as f:
        json.dump(state, f, indent=2)
    os.rename(tmp, '$STATE_FILE')
except: pass
" &
```

**Considerations:**
- Run in background (`&`) to avoid blocking tool execution
- Use aggressive timeout if running synchronously (e.g., 200ms)
- Throttle updates if needed (max once per N seconds)

### Watchdog Pattern

**Never trust state files alone.** Validate against external ground truth:

```bash
#!/bin/bash
# watchdog-sessions - Detect stalled agents

# Check if AI process actually exists in tmux pane
has_ai_process() {
    local session="$1"
    pstree -p $(tmux list-panes -t "$session" -F '#{pane_pid}') 2>/dev/null \
        | grep -qE '(claude|gemini|codex)\('
}

for domain in $DOMAINS; do
    if has_ai_process "$domain"; then
        echo "$domain: ACTIVE"
    elif [[ $(jq -r '.status' "$domain/state.json") == "working" ]]; then
        echo "$domain: STALLED (state=working but no process)"
        # Alert or auto-recover
    fi
done
```

**Status precedence:** `watchdog detection > hook updates > model updates`

### Edge Cases

| Scenario | Problem | Mitigation |
|----------|---------|------------|
| Long think without tools | Heartbeat stales | Watchdog checks process existence, not just timestamp |
| Crash before SessionEnd | status stays "working" | Watchdog detects no process, marks idle after grace period |
| Hook execution fails | State not updated | Log hook failures, retry transient errors |
| Multiple rapid tool calls | Timestamp reordering | Use monotonic time or accept last-write-wins |
| PID reuse after crash | Wrong process detected | Store PID + start_time, verify both |

### Recommended Hook Configuration

```json
{
  "hooks": {
    "SessionStart": [{"command": "./scripts/session-start-hook.sh"}],
    "PostToolUse": [{"command": "./scripts/activity-hook.sh", "timeout": 5}],
    "SessionEnd": [{"command": "./scripts/session-end-hook.sh"}]
  }
}
```

---

## Best Practices

### DO:
- Use hooks for automatic state updates (not model memory)
- Keep briefs concise but complete
- Log significant events to ops.jsonl
- Run a watchdog that checks process existence
- Test recovery flow regularly
- Design for "fresh agent with context" scenario
- Use atomic writes for all state updates

### DON'T:
- Depend on the model to remember to update state
- Depend solely on platform resume features
- Store essential state only in conversation history
- Store secrets in state files
- Trust state files without process verification
- Assume sessions will persist
- Make briefs that require transcript to understand
- Use relative paths in production

---

## Recovery Scenarios

### Scenario 1: Context Window Exhaustion

**What happens:** Agent context fills up, session compacts or terminates.

**Phoenix recovery:**
1. New session starts automatically or manually
2. Start hook detects `status: working` in state.json
3. Recovery context displayed with last task
4. Agent picks up from documented state

### Scenario 2: Session Resume Fails

**What happens:** Platform resume feature returns "no sessions found."

**Phoenix recovery:**
1. Same as above—resume failure is expected, not catastrophic
2. State files contain everything needed
3. No dependency on resume working

### Scenario 3: Mid-Task Crash

**What happens:** Network issue, system crash, or tool failure kills session.

**Phoenix recovery:**
1. End hook may or may not fire
2. Next start hook detects working/error state
3. Agent reviews last known state
4. Either continues or confirms with user

---

## FAQ

**Q: Why not just fix session resume?**

A: Resume mechanisms are controlled by the platform and can be opaque. Phoenix Protocol gives you explicit control over recovery.

**Q: Doesn't this duplicate state?**

A: The state files ARE the source of truth. Conversation history is a cache that can be safely lost.

**Q: What if state files get corrupted?**

A: Use atomic writes to prevent corruption. If it occurs, ops.jsonl and version control provide recovery paths.

**Q: How often should I update state?**

A: On every significant transition: task start, checkpoint, completion, error. Not on every message.

**Q: What about multi-agent systems?**

A: Each agent should have its own state directory, or use proper locking/coordination for shared state.

---

## Contributors

This protocol emerged from multi-agent debugging sessions involving:

- **Claude** (Anthropic): Problem identification, implementation, specification writing
- **Codex** (OpenAI): Root cause analysis, technical review, state management edge cases
- **Gemini** (Google): Conceptual reframe ("Phoenix" metaphor), safety recommendations, hook architecture review

Key insights from collaborative development:
- Sessions should be treated as ephemeral rather than preserved (v1.0)
- State updates must not depend on model memory—use hooks (v1.2)

---

## License

This specification is released under CC0 (public domain). Use freely.

---

## Version History

- **v1.2** (2026-02-02): Added State Management section with hook-based updates, heartbeat pattern, and watchdog validation (tripartite review with Claude, Codex, Gemini)
- **v1.1** (2026-02-02): Added Prerequisites, Security, Concurrency sections; refined attribution; softened absolute claims per tripartite review
- **v1.0** (2026-02-02): Initial specification
