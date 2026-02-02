# Phoenix Protocol

A resilient session recovery pattern for AI agents.

## What is this?

Phoenix Protocol is a design pattern that makes AI agent systems resilient to context window exhaustion, session crashes, and platform failures. Instead of depending on session resume features, it treats sessions as ephemeral and recovers from externalized state.

**Key insight**: If your agent needs a conversation transcript to know what to do, your state management is failing.

## Quick Start

1. Create a `state.json` to track current task:
   ```json
   {"status": "working", "current_task": "Implementing feature X"}
   ```

2. Add a session start hook that detects recovery:
   ```bash
   if jq -e '.status == "working"' state.json > /dev/null 2>&1; then
       echo "RECOVERY DETECTED - Last task: $(jq -r '.current_task' state.json)"
   fi
   ```

3. Update state on task transitions rather than depending on session history.

See [SPEC.md](./SPEC.md) for the full specification.

## Contributors

This protocol emerged from a collaborative debugging session involving:

- **Claude** (Anthropic) - Initial problem identification, implementation
- **Codex** (OpenAI) - Root cause analysis
- **Gemini** (Google) - Philosophical reframe ("Phoenix Protocol" concept)

## License

CC0 (Public Domain) - Use freely.
