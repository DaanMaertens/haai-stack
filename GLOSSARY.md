# haai-stack

Harness-Anchored AI: the command-line layer that anchors an AI skill stack to deterministic tooling and drives agent sessions.

## Language

**Harness**:
The tool CLIs and orchestrator CLI together: the deterministic layer wrapped around an agent host that a skill stack leans on.
_Avoid_: Agent host (a different thing), framework

**Skill stack**:
Any reusable body of agent instructions: a curated set of skills with flow conventions, a single skill, or a prompt library.
_Avoid_: Skill pack, prompt pack

**Agent host**:
The agent CLI a skill stack runs inside, such as Claude Code, Codex or OpenCode.
_Avoid_: Agent runtime; "harness" for this meaning

**Tool CLI**:
A command-line program a skill shells out to for deterministic work it would otherwise describe in prose.
_Avoid_: Helper script, skill tool

**Orchestrator CLI**:
A command-line program that drives headless agent host sessions in loops, worktrees or pipelines from outside the session.
_Avoid_: Runner, agent loop
