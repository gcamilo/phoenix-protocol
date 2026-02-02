# Phoenix Protocol

**A Resilient Session Recovery Pattern for AI Agents**

*Collaboratively designed by Claude (Anthropic), Codex (OpenAI), and Gemini (Google)*

---

## Abstract

Phoenix Protocol is a design pattern for building resilient AI agent systems that can survive context window exhaustion, session crashes, and unexpected terminations. Instead of depending on platform-specific session resume features, Phoenix Protocol treats sessions as ephemeral and ensures recovery through externalized state.

The key insight: **If your agent needs a conversation transcript to know what to do, your state management is failing.**

---

## The Problem

AI agents running in persistent sessions face several failure modes:

1. **Context Window Exhaustion**: Long-running sessions eventually fill their context window, causing automatic compaction or termination
2. **Platform Resume Failures**: Session resume features are black boxes that can fail silently
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
- Fresh context often performs better than accumulated baggage

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

Tracks current task and status:

```json
{
  "status": "working",
  "current_task": "Implementing user authentication",
  "last_output": "Added JWT validation middleware",
  "last_active": "2026-02-02T10:30:00Z"
}
```

Status values:
- `idle`: No active task
- `working`: Task in progress
- `error`: Task failed, needs attention

### 2. Brief File (`brief.md`)

Human-readable context summary updated periodically:

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

### 3. Session Hooks

#### Start Hook (Recovery Detection)

```bash
#!/bin/bash
# session-start-hook.sh

STATE_FILE="./state.json"

if [[ -f "$STATE_FILE" ]]; then
    LAST_STATUS=$(jq -r '.status' "$STATE_FILE")
    LAST_TASK=$(jq -r '.current_task' "$STATE_FILE")

    if [[ "$LAST_STATUS" == "working" ]]; then
        echo "╔════════════════════════════════════════╗"
        echo "║         RECOVERY DETECTED              ║"
        echo "╠════════════════════════════════════════╣"
        echo "║ Previous task: $LAST_TASK"
        echo "╚════════════════════════════════════════╝"
        cat "$STATE_FILE"
    fi
fi
```

#### End Hook (State Preservation)

```bash
#!/bin/bash
# session-end-hook.sh

# Log session end for observability (implement log-event for your system)
# log-event session_end ok '{"trigger":"hook"}'
echo "Session ended at $(date -Iseconds)" >> ./ops.jsonl

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

---

## Implementation Guide

### Step 1: Add State Management

Create utilities to update state on task transitions:

```bash
# Mark task as started
state-update --working "Implementing feature X"

# Mark task as complete
state-update --done "Feature X complete"

# Mark error state
state-update --error "Build failed: missing dependency"
```

### Step 2: Configure Session Hooks

Register hooks with your AI agent platform:

```json
{
  "hooks": {
    "SessionStart": ["./scripts/session-start-hook.sh"],
    "SessionEnd": ["./scripts/session-end-hook.sh"]
  }
}
```

### Step 3: Generate Briefs Regularly

Update brief on significant progress or at session checkpoints:

```bash
# Generate brief summarizing current state
./scripts/generate-brief.sh > brief.md
```

### Step 4: Test Recovery Flow

1. Start a session and begin a task
2. Mark state as "working"
3. Kill the session (simulate crash)
4. Start new session
5. Verify recovery context is injected

---

## Best Practices

### DO:
- Update `state.json` at every task transition
- Keep briefs concise but complete
- Log significant events to ops.jsonl
- Test recovery flow regularly
- Design for "fresh agent with context" scenario

### DON'T:
- Depend solely on platform resume features
- Store essential state only in conversation history
- Assume sessions will persist
- Make briefs that require transcript to understand

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
1. Same as above - resume failure is expected
2. State files contain everything needed
3. No dependency on resume working

### Scenario 3: Mid-Task Crash

**What happens:** Network issue, system crash, or tool failure kills session.

**Phoenix recovery:**
1. End hook may or may not fire
2. Next start hook detects working state
3. Agent reviews last known state
4. Either continues or confirms with user

---

## FAQ

**Q: Why not just fix session resume?**

A: Resume is a black box controlled by the platform. Phoenix Protocol gives you control over recovery.

**Q: Doesn't this duplicate state?**

A: The state files ARE the source of truth. Conversation history is a cache that can be safely lost.

**Q: What if state files get corrupted?**

A: Keep state simple and atomic. Version control helps. Ops log provides event history for reconstruction.

**Q: How often should I update state?**

A: On every significant transition: task start, checkpoint, completion, error. Not on every message.

---

## Contributors

This protocol emerged from a collaborative debugging session involving:

- **Claude (Anthropic)**: Initial problem identification, implementation
- **Codex (OpenAI)**: Root cause analysis (cleanup settings)
- **Gemini (Google)**: Philosophical reframe ("Phoenix Protocol" concept)

The insight that sessions should be treated as ephemeral rather than preserved came from Gemini's observation that attempting to resurrect dead sessions is the wrong approach—instead, start fresh with preserved state.

---

## License

This specification is released under CC0 (public domain). Use freely.

---

## Version History

- **v1.0** (2026-02-02): Initial specification
