# Orchestrator CLIs that drive headless agent sessions

Research for [#5](https://github.com/DaanMaertens/haai-stack/issues/5) (map: [#2](https://github.com/DaanMaertens/haai-stack/issues/2)). Surveyed 2026-10-19. Star counts and last-push dates come from the GitHub API on that day.

## TL;DR

1. Six control models recur: the **Ralph loop** (a shell or TUI loop that starts a fresh headless session per task), the **tracker-polling daemon** (Symphony and its clones), the **workflow DAG engine** (Archon), **tmux/worktree multiplexers** around *interactive* sessions (claude-squad, Gas Town, agent farm), **event-triggered CI** (claude-code-action), and **inside-out** setups where a skill stack orchestrates from inside one session (ccpm, pstack and its ports, which lean on host schedulers plus bookkeeping Tool CLIs).
2. Isolation has converged on **one git worktree per task/issue**. Work selection has converged on **a file or tracker queue plus labels and priority** (prd.json, GitHub/Linear issues, Beads).
3. **Deterministic flow steps are rare.** Most loops stop when the model prints a sentinel (`<promise>COMPLETE</promise>`, `LOOP_COMPLETE`, `EXIT_SIGNAL`). Only Archon (YAML DAG with `bash`, `until_bash` and `wait` nodes) and, partly, Gas Town (formulas/molecules) and ralph-orchestrator (backpressure gates) put code in charge of the sequence.
4. **State and observability are fragmented.** Every tool has its own format (progress.txt, `.ralph/*.json`, `event_history.jsonl`, NDJSON receipts, SQLite, OTEL, in-memory only). Few link a run to the host's own session transcript, and resume usually means "re-poll the tracker", not "continue the session".
5. Gap for haai: no lightweight, Claude-Code-first Orchestrator CLI sequences an *existing* Skill stack's slash commands as deterministic steps with shell gates, plus a durable, queryable run record. `claude -p "/skill-name"` already expands user-invoked skills ([docs](https://code.claude.com/docs/en/headless)), so the primitive exists.

## Comparison table

| Project | Control model | How it picks work | Parallelism / isolation | State & resume | Observability | Agent hosts | Skill stack relation | Maturity |
|---|---|---|---|---|---|---|---|---|
| [snarktank/ralph](https://github.com/snarktank/ralph) | Bash loop, fresh `claude --print` / `amp` per iteration | Highest-priority story with `passes: false` in `prd.json` | None (one feature branch) | `prd.json`, `progress.txt`, git; old runs archived | stdout tee; `jq` on prd.json | Amp, Claude Code | Ships `/prd` and `/ralph` skills to *prepare* the queue | 22.0k stars, last push 2026-02-02 |
| [frankbria/ralph-claude-code](https://github.com/frankbria/ralph-claude-code) | Bash loop with exit gate, rate limit and circuit breaker | `.ralph/fix_plan.md`; `ralph-queue` over GitHub issues (P-labels, `depends on #N`) | Sequential | `.ralph/`, `queue.json` survives restarts; `--resume <session_id>` | `ralph.log`, tmux monitor, `ralph-stats` JSONL metrics | Claude Code | None found | 9.7k stars, push 2026-10-17, v0.11.5 |
| [mikeyobrien/ralph-orchestrator](https://github.com/mikeyobrien/ralph-orchestrator) | Rust loop with "hats" (personas) coordinating via events; PTY execution | Prompt or spec dir from `ralph plan` | Extra loops in `.worktrees/<loop-id>/`, merge queue | `.agent/` (memories, tasks.jsonl, event_history.jsonl), `.ralph/loops.json`; `ralph loops resume` | Web dashboard (alpha), MCP server, Telegram | 11 incl. Claude Code, Codex, Gemini, OpenCode, Amp, Pi | Has a `ralph-loop` skill; presets are its own | 3.2k stars, push 2026-10-15 |
| [subsy/ralph-tui](https://github.com/subsy/ralph-tui) | TUI loop: select task, build prompt, run agent, detect completion | prd.json or Beads epic | Not stated in README | Session persistence, `ralph-tui resume` | Real-time TUI, subagent tracing, `logs`, remote instances | Claude Code, OpenCode, Droid, Cursor, Gemini, Codex, Kiro | PRD/task skills | 2.5k stars, push 2026-09-04 |
| [openai/symphony](https://github.com/openai/symphony) | Long-running daemon: poll, dispatch, reconcile (language-neutral SPEC + Elixir reference) | Tracker issues in active states, required labels, priority then age | Per-issue workspace, bounded concurrency (global and per-state) | In-memory scheduler; restart recovers from tracker + workspaces | Structured `key=value` logs; optional Phoenix dashboard + JSON API | Codex app-server only | `WORKFLOW.md` prompt; optional `commit/push/land/linear` skills | 27.7k stars, push 2026-09-15, "engineering preview" |
| [mraza007/baton](https://github.com/mraza007/baton) | Symphony clone for Claude Code | `gh issue list` by labels/assignee | Worktree per issue, `max_concurrent` | In-memory + JSON persistence | Colour terminal logs, `baton status` | Claude Code (`claude -p`) | Per-issue `## Skills` section in the issue body | 26 stars, push 2026-03-27 |
| [coleam00/Archon](https://github.com/coleam00/Archon) | YAML workflow DAG engine (agent, bash, script, loop, wait, child-workflow nodes) | `archon workflow run`, or build/merge queues over GitHub issues | Worktree per repository run | SQLite/Postgres; event log + transcript per run; explicit `--resume` | Web console (`archon serve`), attention queue, cost per run | Claude Code (Agent SDK), Codex, Pi | `command` nodes, per-node `skills:` list; `archon skill add` lets the agent drive Archon | 23.7k stars, push 2026-10-19 |
| [gastownhall/gastown](https://github.com/gastownhall/gastown) | Multi-agent "town": a Mayor agent plus worker polecats in tmux, watchdogs, merge queue | `gt sling <bead>` onto Beads issues/convoys | Worktree-backed hooks per agent; scheduler caps polecats | Beads ledger (git/Dolt); `.events.jsonl`; `gt seance` queries old sessions | `gt feed` TUI, web dashboard, OpenTelemetry | Claude Code (default), Codex, Copilot, Gemini, Cursor | Formulas (TOML) to molecules; Claude hooks for mail injection | 18.4k stars, push 2026-10-18 |
| [smtg-ai/claude-squad](https://github.com/smtg-ai/claude-squad) | TUI multiplexer over *interactive* sessions | Human creates each session | tmux session + git worktree per task | Stored instances (`cs reset`); pause/resume | Preview and diff tabs | Claude Code, Codex, Gemini, Aider, any command | None | 8.6k stars, push 2026-08-20 |
| [Dicklesworthstone/claude_code_agent_farm](https://github.com/Dicklesworthstone/claude_code_agent_farm) | Python supervisor typing `cc` into 20+ tmux panes | Chunks of a generated problem/best-practice list; lock files | Shared tree + lock-based coordination | Monitor state JSON; auto-restart | tmux dashboard, HTML run reports | Claude Code (monitoring is Claude-specific) | None | 0.9k stars, push 2026-09-21 |
| [BloopAI/vibe-kanban](https://github.com/BloopAI/vibe-kanban) | Local web kanban that runs agents in workspaces | Human moves kanban issues | Branch + terminal + dev server per workspace | Local DB (sqlx) | Web UI, inline diff review | 10+ incl. Claude Code, Codex, Gemini, OpenCode | None found | 28.4k stars, push 2026-09-19; company shut down April 2026, project now community-maintained |
| [Vinix24/vnx-orchestration](https://github.com/Vinix24/vnx-orchestration) | T0 orchestrator, ephemeral headless workers, review gates | `vnx dispatch-agent` with a role | Worktree per dispatch (parallel tracks "designed, not built") | Append-only NDJSON receipts, report per dispatch | `receipt_query.py`, cost per gate | `claude -p`, codex, gemini, kimi, ollama | Own agent roles | 62 stars, push 2026-10-19 |
| [anthropics/claude-code-action](https://github.com/anthropics/claude-code-action) | GitHub Action triggered by `@claude`, assignment or schedule | GitHub event | One runner per job | Outputs `session_id` (usable with `--resume`) and `execution_file` | Progress-checkbox comment, Action logs | Claude Code via Agent SDK | `plugins` / `plugin_marketplaces` inputs | 9.6k stars, push 2026-10-18 |
| [automazeio/ccpm](https://github.com/automazeio/ccpm) | **Inside-out**: a skill that turns PRD → epic → GitHub issues and launches parallel sub-agents | "start working on issue N" | Worktree per epic, parallel streams inside it | `.claude/epics/` files + GitHub issue comments | Bash status scripts (standup, blocked, next) | Any Agent Skills host | *Is* the skill stack | 8.4k stars, push 2026-10-19 |

## Headless primitives the tools build on

- **Claude Code**: `claude -p` with `--output-format json|stream-json`, `--json-schema`, `--continue` / `--resume <session_id>`, `--permission-mode`, `--permission-prompts none` for unattended runs. User-invoked skills work in `-p`: "Include `/skill-name` in the prompt string and Claude Code expands it before running." `--bare` skips skill and plugin auto-discovery and "will become the default for `-p` in a future release". ([docs](https://code.claude.com/docs/en/headless))
- **Codex**: `codex exec` with `--json` (JSONL events `thread.started`, `turn.completed`, ...), `codex exec resume <SESSION_ID>`, `--output-schema`; read-only sandbox by default ([docs](https://learn.chatgpt.com/docs/non-interactive-mode)). The page does not cover skills in exec mode. Symphony instead drives `codex app-server` ([SPEC §10](https://github.com/openai/symphony/blob/main/SPEC.md)).
- Two tools deliberately avoid headless mode: Gas Town and agent farm drive interactive sessions in tmux ([gastown](https://github.com/gastownhall/gastown#runtime-configuration), [agent farm](https://github.com/Dicklesworthstone/claude_code_agent_farm)). VNX removed its tmux lane on 2026-09-18 and kept only `claude -p` ([VNX README](https://github.com/Vinix24/vnx-orchestration)).

## Per-project notes

### Ralph loops

- **Origin**: Geoffrey Huntley's pattern, cited by every Ralph tool ([ghuntley.com/ralph](https://ghuntley.com/ralph/)).
- **snarktank/ralph**: `ralph.sh` runs `claude --dangerously-skip-permissions --print < CLAUDE.md` (or `amp --dangerously-allow-all`) in a `for` loop and exits when output contains `<promise>COMPLETE</promise>` ([ralph.sh](https://github.com/snarktank/ralph/blob/main/ralph.sh)). Each iteration is a fresh context. Memory is git, `progress.txt` and `prd.json`; the agent also updates `AGENTS.md` with learnings ([README](https://github.com/snarktank/ralph)). Quality checks (typecheck, tests) are steps in the *prompt*, not enforced by the loop.
- **frankbria/ralph-claude-code**: a "dual-condition exit gate" needs completion indicators *and* `EXIT_SIGNAL: true`. It adds a rate limit (100 calls/hour), a circuit breaker, 5-hour-limit detection, `--resume <session_id>` instead of `--continue` to avoid session hijacking, `--backup`/`--rollback` git branches, and a `ralph-queue` that reads GitHub issues with P-labels and `depends on #N` parsing into `.ralph/queue.json` ([README](https://github.com/frankbria/ralph-claude-code)).
- **ralph-orchestrator**: hats (personas) coordinate through an event bus; "backpressure" gates reject incomplete work (tests, lint, typecheck) ([README](https://github.com/mikeyobrien/ralph-orchestrator)). Backends run through a PTY executor, and state lives in `.agent/` ([architecture](https://mikeyobrien.github.io/ralph-orchestrator/advanced/architecture/)). The first loop holds `.ralph/loop.lock`; more loops spawn into `.worktrees/<loop-id>/`, queue themselves for merge, and are tracked in `.ralph/loops.json` ([parallel loops](https://mikeyobrien.github.io/ralph-orchestrator/advanced/parallel-loops/)). Human-in-the-loop goes through Telegram: `human.interact` blocks the loop until answered.
- **ralph-tui**: the same loop with trackers as plugins (prd.json, Beads), session persistence and resume, and a remote-control mode ([README](https://github.com/subsy/ralph-tui)).

### Tracker-polling daemons

- **Symphony** is a spec first: "Tell your favorite coding agent to build Symphony ... according to the following spec" ([README](https://github.com/openai/symphony)). Key design points from [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md):
  - Policy lives in a repo-owned `WORKFLOW.md` (YAML front matter plus a prompt template) and is hot-reloaded (§5, §6.2).
  - An explicit claim state machine (`Unclaimed → Claimed → Running → RetryQueued → Released`), and run phases that end in distinct terminal reasons (`Succeeded`, `Failed`, `TimedOut`, `Stalled`, `CanceledByReconciliation`) (§7).
  - Each tick reconciles first: it stops runs whose issue went terminal and detects stalls (§8.5). Retries use exponential backoff (§8.4).
  - The orchestrator only *reads* the tracker. Ticket writes are done by the agent through tracker tools that Symphony runs with host-side credentials (§11.5).
  - Restart recovery is deliberately tracker- and filesystem-driven, with no durable DB (§14.3).
  - The Elixir reference ships adapters for Linear, GitHub Issues, Jira, Asana and GitLab, and an optional Phoenix dashboard with `/api/v1/state` ([elixir/README](https://github.com/openai/symphony/blob/main/elixir/README.md)). Runs that need operator input are marked "blocked", but "Blocked entries are in memory only" (same source).
- **Baton** reimplements the spec for Claude Code (`claude -p` per turn, GitHub via `gh`), and lets an issue body pick skills with a `## Skills` section ([README](https://github.com/mraza007/baton)). It shows the spec spreading to other hosts, but has low adoption.

### Workflow engine: Archon

- A workflow is a graph of nodes. A node can be an agent prompt, a `command`, `bash`, `script`, `loop` (with `until_bash` and `fresh_context`), `wait` (human approval form), `cancel` or a child workflow ([README](https://github.com/coleam00/Archon), [authoring guide](https://archon.diy/guides/authoring-workflows/)). "The workflow fixes the steps, the checks and the gates. The agent supplies the judgment inside each step."
- The bundled `sdlc` pack has an intake, a build queue (claims the next ready issue), a merge queue and discovery filing, all through `gh`. Archon reports that 363 of 524 PRs merged 1–19 Oct 2026 were built by Archon runs ([README](https://github.com/coleam00/Archon)).
- State is SQLite by default (Postgres optional), with an event log, transcript and cost for every run. Paused runs become "attention items" (`archon workflow attention`). Resume is explicit (`--resume` or `archon workflow resume <run-id>`) ([authoring guide](https://archon.diy/guides/authoring-workflows/)).
- Skill stack relation: a `skills` field gives "Exact Claude-native skill selection (omission/`[]` selects none)", and Codex nodes call skills with `$skill-name`. "A Claude workflow node loads none of the Claude Code plugins installed on the machine" (same source). The providers package depends on `@anthropic-ai/claude-agent-sdk` ([code search hit: packages/providers/package.json](https://github.com/coleam00/Archon/blob/main/packages/providers/package.json)).
- Telemetry is on by default (opt out with env vars) ([README](https://github.com/coleam00/Archon#telemetry)).

### Multiplexers over interactive sessions

- **claude-squad** uses "tmux to create isolated terminal sessions" and "git worktrees to isolate codebases". A human creates, attaches to, pauses and resumes each session; `-y` auto-accepts prompts ([README](https://github.com/smtg-ai/claude-squad)). It manages sessions without orchestrating them.
- **Gas Town** goes furthest on roles. A Mayor (a Claude Code instance) coordinates. Polecats are workers with "persistent identity but ephemeral sessions". Witness, Deacon and Dogs are watchdogs, a Refinery is a Bors-style merge queue, and escalation is routed by severity. Work is Beads issues bundled in convoys, and multi-step work is defined as TOML "formulas" that become "molecules" with checkpoint recovery. Observability comes from `gt feed` (TUI), `gt dashboard` and OpenTelemetry logs and metrics ([README](https://github.com/gastownhall/gastown)). Codex needs a "startup fallback" because it has no hooks.
- **Agent farm** types `cc` (an alias for `claude --dangerously-skip-permissions`) into tmux panes and parses the screen to detect readiness, context % and usage limits. With other CLIs, "auto-restart, context management, and health checks will be degraded or inert" ([README](https://github.com/Dicklesworthstone/claude_code_agent_farm)).
- **Vibe Kanban** (web UI over workspaces) is notable as a maturity signal. bloop shut the company down on 2026-04-10, and the open-source app continues "community maintained" and fully local ([announcement](https://www.vibekanban.com/blog/shutdown)).
- **Orchestrator.inc** ([OrchestratorInc/agent-orchestrator](https://github.com/OrchestratorInc/agent-orchestrator), 13.4k stars) is now a desktop app plus daemon ("no CLI is required"). It sets one pattern worth copying: kanban columns are *derived* from session, PR, CI and review facts (Working / Needs you / In review / Ready to merge) ([README](https://github.com/OrchestratorInc/agent-orchestrator)). It is not in the table because it is not a CLI.

### Governance-first: VNX

Every dispatch writes an append-only NDJSON receipt. A receipt refuses to report "done" without a PR, a review gate cannot be skipped, and a merge is pinned to the head SHA the gates approved. The README states maturity per tier and says parallel multi-track execution is "designed, not built" ([README](https://github.com/Vinix24/vnx-orchestration)). Small project (62 stars), but the clearest example of run records as an audit trail.

### Event-triggered CI: claude-code-action

Runs Claude Code through `@anthropic-ai/claude-agent-sdk` on a GitHub runner when someone writes `@claude`, assigns an issue, or a scheduled workflow fires. It exposes `session_id` and `execution_file` as outputs, and can install plugins from marketplaces per run ([README](https://github.com/anthropics/claude-code-action), [action.yml](https://github.com/anthropics/claude-code-action/blob/main/action.yml), [package.json](https://github.com/anthropics/claude-code-action/blob/main/package.json)). This is the pattern behind "issue assignment = dispatch".

### Inside-out: the skill stack orchestrates itself

- **ccpm** is "now an AGENT SKILL". It writes PRDs, epics and tasks into `.claude/epics/`, syncs them to GitHub issues and sub-issues, creates a worktree per epic, and "start working on issue N" launches parallel agents from the main conversation, which "becomes the conductor" ([README](https://github.com/automazeio/ccpm)). Nothing outside the session drives it, so it runs only while someone keeps a session open.
- **pstack** (poteto's skill stack, the map's starting point) lives in [cursor/plugins/pstack](https://github.com/cursor/plugins/tree/main/pstack). Its `orchestrate` playbook hands "a standing project" to one coordinator chat with fleets of subagents. It "works extremely well with cursor's `/loop` command". Its only AFK piece, the `benny` automation pack, runs on Cursor Automations and "is cursor-only". In Claude Code, steps needing Cursor-only features are skipped ([README](https://github.com/cursor/plugins/blob/main/pstack/README.md)).
- **pstack community ports** were checked for an Orchestrator CLI that replaces `/loop` and cloud agents. None has one. Each swaps those mechanisms for the *new* Agent host's own orchestration, plus Tool CLIs that the coordinating agent calls to keep state:
  - **[creedants/pstack-t3](https://github.com/creedants/pstack-t3)** (14 stars, created 2026-10-03, push 2026-10-19) adds `$brigade`, "a standing coordinator per project". The coordinator is "a pinned thread on the project root. It never writes code. Each unit runs in its own worktree thread." Scheduling, delegation and wake-ups come from T3 Code's orchestrator tools (`delegate_task`, `t3_thread_launch`, `schedule_task`). A worker's last step is `t3_thread_send` back to the coordinator. A liveness check runs every 10–30 minutes ([how it works](https://github.com/creedants/pstack-t3/blob/main/docs/how-it-works.md)).
    - The CLIs it ships are bookkeeping: `brigade.py` ("Bookkeeping for brigade restaurants": tickets, priorities, worker caps, build/check commands, spend), `land.py` (a landing queue that is "the only writer to trunk" and bounces a commit if the rebased result differs from the reviewed one), and `activity.py` (an activity page built from the brigade store plus T3 Code's state DB) ([CLI reference](https://github.com/creedants/pstack-t3/blob/main/docs/cli/README.md), [brigade.py](https://github.com/creedants/pstack-t3/blob/main/docs/cli/brigade.md)).
    - It is the most complete design among the ports for deterministic landing and observability, but the control loop is an agent thread inside [T3 Code](https://github.com/pingdotgg/t3code), not a CLI.
  - **[ScriptedAlchemy/pstack-codex](https://github.com/ScriptedAlchemy/pstack-codex)** (69 stars, push 2026-10-17) maps long-running loops to "supported heartbeat/project automations, durable ledgers" in Codex, and states that "detached sleepers are not persistence" ([PARITY.md](https://github.com/ScriptedAlchemy/pstack-codex/blob/main/PARITY.md)). It ships `orch` (a Bun/commander CLI over a store of units, tracks, verdicts and ledger entries) ([orch.ts](https://github.com/ScriptedAlchemy/pstack-codex/blob/main/plugins/pstack/skills/poteto-mode/scripts/orch/orch.ts)). The runtime contract says to keep run state in `<repo>/.codex/pstack-runs/<run-name>/`, and "On restart, read the ledger, inspect current branch/PR state, and recreate workers; old agent ids are not durable worker handles" ([CODEX.md](https://github.com/ScriptedAlchemy/pstack-codex/blob/main/plugins/pstack/CODEX.md)). It also ships a scheduled-polling Benny adapter with a durable ledger, but "neither starts automatically on installation" ([README](https://github.com/ScriptedAlchemy/pstack-codex)).
  - **[uzairansaruzi/p3-stack](https://github.com/uzairansaruzi/p3-stack)** (155 stars, created and last pushed 2026-10-05) ships skills only, rewired to T3 Code primitives: "`schedule_task` runs audits and overnight cadences even when no turn is active. No `/loop`", and `watch_pull_request` replaces polling ([README](https://github.com/uzairansaruzi/p3-stack)). No CLI.
  - Takeaway for haai: the pstack lineage settles on "the Agent host schedules and wakes, the Skill stack decides, Tool CLIs hold durable state (ledger, landing queue, activity view)". On Claude Code no such host-level coordinator exists outside the session, which is the slot an Orchestrator CLI would fill.
- **Ruflo** (formerly claude-flow, [ruvnet/ruflo](https://github.com/ruvnet/ruflo), 77k stars) installs as a Claude Code plugin and MCP server that adds swarms and memory inside the session ([README](https://github.com/ruvnet/ruflo)). It is not an outside-in driver. Its swarm and memory claims were not verified here.

## Recurring patterns

1. **Fresh context per unit of work, memory on disk.** Ralph tools restart the host every iteration and carry state in files and git ([snarktank](https://github.com/snarktank/ralph#critical-concepts)). Symphony instead keeps one thread across continuation turns within a worker run ([SPEC §7.1](https://github.com/openai/symphony/blob/main/SPEC.md)).
2. **Worktree per task** is the default isolation everywhere except agent farm (locks in one tree) and ccpm (one worktree per epic, parallel streams inside it).
3. **The tracker is the queue.** GitHub, Linear and Jira issues (Symphony, Baton, Archon, frankbria `ralph-queue`, ccpm), Beads (Gas Town, ralph-tui), or `prd.json`. Labels and P-labels set eligibility and priority, and `depends on #N` sets order.
4. **Repo-owned workflow file.** `WORKFLOW.md` (Symphony, Baton), `.archon/workflows/*.yaml`, `ralph.yml`, formulas (Gas Town). The policy is versioned with the code.
5. **Completion is model-declared.** Sentinel strings or status blocks decide when to stop (snarktank, frankbria, ralph-orchestrator). Code-declared completion (`until_bash`, CI green, PR exists) is the exception (Archon, VNX, ralph-orchestrator gates).
6. **Human attention as a first-class state.** Archon attention items, Symphony "blocked", ralph-orchestrator Telegram `human.interact`, Orchestrator.inc "Needs you", Gas Town escalation.
7. **Merge queue as a separate stage** (Archon merge queue, Gas Town Refinery, ralph-orchestrator merge queue, VNX merge door).
8. **Tool CLIs as the agent's durable memory.** Even where an agent coordinates, deterministic state lives in small CLIs the agent calls: `brigade.py`/`land.py` (pstack-t3), `orch` (pstack-codex), `bd` (Beads, Gas Town), `gt done`/`gt escalate`, and `ralph tools interact` (ralph-orchestrator).
9. **Multi-host via a thin adapter** (command plus prompt mode). Host-specific features (hooks, context %, session resume) degrade on other hosts ([Gas Town runtime notes](https://github.com/gastownhall/gastown#runtime-configuration), [agent farm](https://github.com/Dicklesworthstone/claude_code_agent_farm)).

## Gaps against haai's three pains

### 1. Deterministic flow steps

- Most tools put the *flow* inside the prompt ("run typecheck, commit, update prd.json"), so the model decides whether steps happened. Only Archon makes the sequence code: nodes, `depends_on`, `when`, `until_bash`, and `wait`.
- **Gap**: no tool sequences an existing Skill stack's slash commands as named steps (for example `/to-spec`, then `/implement`, then `/code-review`) with shell gates between them, and without adopting a new platform. Archon is closest: it supports `command` nodes and a `skills:` list. But it is a full platform with its own DB, web console, plugin registry and default-on telemetry, and it deliberately loads none of the user's installed Claude Code plugins.
- The primitive already exists: `claude -p "/skill-name ..."` expands user-invoked skills ([docs](https://code.claude.com/docs/en/headless)).

### 2. AFK orchestration

- This pain is well covered. Tracker polling with claims, reconciliation, stall detection and backoff is specified in detail by Symphony and cloned by others.
- **Gaps**:
  - Symphony's reference targets Codex app-server only.
  - Blocked or needs-human state is often in memory only (Symphony) or tied to one channel (Telegram).
  - Rate-limit and subscription-window handling is ad hoc (frankbria's 5-hour-limit detection, VNX's billing lanes).
  - Nothing reviewed here uses the tracker issue itself as the durable human-question channel, though Archon writes answers back into the issue "as the new scope" ([README](https://github.com/coleam00/Archon)).

### 3. Observability and state

- Every tool uses its own run-record format: `progress.txt`/`prd.json`, `.ralph/*.json` + `ralph.log`, `.agent/event_history.jsonl`, NDJSON receipts, SQLite, OTEL, or in-memory.
- Links back to the host's own transcript are rare. Symphony logs `session_id` (§13.1), claude-code-action outputs `session_id` + `execution_file`, and Gas Town's `seance` reads `.events.jsonl`.
- **Gaps**:
  - No shared, inspectable run record ("which step ran, with which skill, which session ID, which gate result, what cost") that is plain files in the repo or issue and works without a server.
  - "Resume" mostly means re-dispatching from tracker state, not continuing a step. Only frankbria (`--resume <session_id>`), Archon (`--resume`) and ralph-orchestrator (`loops resume`) resume explicitly.

## Unverified or not checked

- I did not run any of these tools. All behaviour comes from READMEs, specs, docs and a few source files linked above.
- ralph-tui parallelism and how it invokes each host: not stated in the README.
- Vibe Kanban internals (state format, how hosts are invoked): only the README and the shutdown post were read.
- Ruflo swarm and memory claims, and Orchestrator.inc's daemon internals: not verified.
- Codex skills in `codex exec`: the docs page does not cover it.
- `GLOSSARY.md` named in the map issue does not exist on `main` yet. Terms (Harness, Agent host, Skill stack, Tool CLI, Orchestrator CLI) are used here as named in [#2](https://github.com/DaanMaertens/haai-stack/issues/2).
