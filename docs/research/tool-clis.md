# Which open-source skill stacks ship tool CLIs that their skills shell out to?

Research for [#4](https://github.com/DaanMaertens/haai-stack/issues/4), part of map [#2](https://github.com/DaanMaertens/haai-stack/issues/2). Terms follow `GLOSSARY.md`: **Harness**, **Agent host**, **Skill stack**, **Tool CLI**, **Orchestrator CLI**. Stars and last push are GitHub API snapshots taken 2026-10-19.

## TL;DR

1. Pairing a skill stack with a tool CLI is now normal. Most of the large stacks do it: spec-kit, OpenSpec, GSD, BMAD, Superpowers, beads, Backlog.md, ccpm and pstack.
2. Two ways to ship it: **scripts bundled inside skill folders** (Superpowers, ccpm, BMAD, pstack, spec-kit, Anthropic skills), or a **standalone installed binary** that the skills or `AGENTS.md` point at (beads `bd`, `openspec`, `backlog`, `task-master`, `gsd-sdk`).
3. The CLIs that hold up have the same contract: `--json` output, one call per step, the CLI owns the state file and the model never edits it by hand, and the CLI never spawns agents. pstack's `orch` states that last rule outright.
4. Orchestration usually lives in a separate component: Sandcastle, Gas Town, `ralph.sh`, `humanlayer launch`, `gsd-sdk auto`, `specify workflow run`. Only GSD, spec-kit and beads with Gas Town share one state store between the tool CLI and the orchestrator CLI.
5. Gaps for haai: Matt's skills ship almost no tool CLIs; no stack has a tool CLI over GitHub Issues with triage labels plus a matching orchestrator; and observability rarely goes beyond a state file or a log. A run history tied to tracker issues is missing everywhere.

## Comparison

| Project | Tool CLI (what it does) | How skills call it | Distribution | Agent hosts | Orchestrator? | Maturity (stars / last push) |
|---|---|---|---|---|---|---|
| [pstack](https://github.com/cursor/plugins/tree/main/pstack) (Cursor) | `orch` (unit, ledger, inbox and gate state as TSV/JSON), `watch-pr`, `check-plan`, `worktree-audit`, `log.sh` | `bun scripts/orch/orch.ts …` from playbooks | Bundled in skill folder, Bun with self-install | Cursor, Claude Code (generated copy) | No: `orch` "never spawns, waits, or wakes anything" | in `cursor/plugins` 13.0k / 2026-10-19 |
| [beads](https://github.com/gastownhall/beads) | `bd`: dependency-aware issue graph in Dolt, `ready`, `claim`, `close`, `prime`, `remember` | `AGENTS.md`/skill tells agent to run `bd …`; SessionStart hook runs `bd prime` | Go binary (curl script, npm `@beads/bd`, Homebrew) | Claude Code, Codex, Cursor, Gemini, Copilot, Factory, … | Yes, separately: [Gas Town](https://github.com/gastownhall/gastown) `gt` | 28.0k / 2026-10-19 |
| [spec-kit](https://github.com/github/spec-kit) (GitHub) | `scripts/{bash,powershell,python}/*` (create feature branch and dirs, setup plan, check prerequisites) plus `specify` | Command frontmatter `scripts: sh: scripts/bash/setup-plan.sh --json` | `uv tool install specify-cli`, scripts copied in by `specify init` | Many integrations (Claude, Copilot, Gemini, …) | Yes: `specify workflow run/resume/status` with persisted `state.json` | 141.9k / 2026-10-19 |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | `openspec`: `status`, `instructions`, `new change`, `validate`, `archive`, all `--json` | Skills run `openspec instructions <artifact> --change X --json` and write to the `resolvedOutputPath` it returns | npm `@fission-ai/openspec` | Claude Code, Codex, Cursor, Copilot, Amazon Q, … | No | 72.4k / 2026-10-19 |
| [GSD](https://github.com/gsd-build/get-shit-done) | `gsd-tools.cjs` (STATE.md, roadmap, phases, commits, validation), being replaced by `gsd-sdk query` | Commands and workflows shell out to `node gsd-tools.cjs …` | npm installer plus `@gsd-build/sdk` | Claude Code first | Yes: `gsd-sdk run/auto` built on the Claude Agent SDK | 64.3k / 2026-05-31 |
| [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD) | PEP 723 Python scripts per skill: `tickets.py` (ticket tree, `next`/`status`), `memlog.py` (append-only log), and others | `uv run {skill-root}/scripts/tickets.py …` | Bundled in skill folders; `npx bmad-method install` | Claude Code, Codex plugins, others | No (unverified) | 54.3k / 2026-10-19 |
| [Superpowers](https://github.com/obra/superpowers) | `sdd-workspace`, `task-brief`, `review-package`, `task-start`, `task-done` (plan workspace, brief extraction, review diff packages, progress ledger) | `bash scripts/review-package PLAN BASE HEAD` from SKILL.md | Bundled in skill folders | About 15 hosts listed in README | No (subagent-driven inside one session) | 302.0k / 2026-10-10 |
| [ccpm](https://github.com/automazeio/ccpm) | 14 bash scripts over GitHub Issues and local files: `next`, `status`, `blocked`, `standup`, `validate`, … | Table in SKILL.md: "always run the bash script directly" | Bundled in skill folder | Any Agent Skills host | No (parallel agents via worktrees, in-session) | 8.4k / 2026-10-19 |
| [Backlog.md](https://github.com/MrLesk/Backlog.md) | `backlog`: markdown task files, board, search, `--plain`/`--json` | `AGENTS.md` tells agent to run `backlog instructions overview`, then `backlog task …` | npm/bun, Homebrew, Nix; optional MCP | Claude Code, Codex, Gemini, Kiro, Cursor | No | 7.1k / 2026-10-19 |
| [Task Master](https://github.com/eyaltoledano/claude-task-master) | `task-master`: `parse-prd`, `next`, `show`, `move`, `research` | Mainly via MCP; CLI is option 2 | npm `task-master-ai` | Cursor, Windsurf, VS Code, Claude Code, Q, … | No | 28.2k / 2026-04-28 (slowing) |
| [HumanLayer](https://github.com/humanlayer/humanlayer) `.claude/commands` | `humanlayer thoughts sync`, `hack/create_worktree.sh`, `humanlayer launch` | Slash commands call them directly | Repo-local; README now says the code is "pretty much all deprecated" | Claude Code | Command *calls* an orchestrator (`humanlayer launch … -w <worktree>`) | 11.7k / commands last touched 2025-11-12 |
| [Ralph](https://github.com/snarktank/ralph) | `ralph.sh` loop plus `prd.json`/`progress.txt` state | `prd`/`ralph` skills produce the `prd.json` the loop reads | Copy files, or Claude Code marketplace | Amp, Claude Code | Yes (the loop is the product) | 22.0k / 2026-02-02 |
| [Sandcastle](https://github.com/mattpocock/sandcastle) | (orchestrator library, plus `sandcastle init` scaffolder) | Not called by skills; runs agents with a `promptFile` | npm `@ai-hero/sandcastle` | Claude Code, Codex, Cursor, OpenCode, Pi | Yes: `run()` with sandboxes, worktrees, iterations, logs | 8.5k / 2026-10-19 |
| [anthropics/skills](https://github.com/anthropics/skills) | Per-skill Python/bash: PDF form fill, docx/pptx edits, skill-creator eval loop | SKILL.md names `scripts/x.py` | Bundled (Agent Skills `scripts/` convention) | Agent Skills hosts | No | 182.2k / 2026-10-09 |
| [cursor-team-kit `control-cli`](https://github.com/cursor/plugins/blob/main/cursor-team-kit/skills/control-cli/SKILL.md) | **None shipped.** Teaches the agent to build a throwaway tmux or PTY harness | n/a | n/a | Cursor | No | in `cursor/plugins` |

## Per project

### pstack (Cursor, poteto): reference point, covered elsewhere

Covered in depth in [`research/pstack-cli`](https://github.com/DaanMaertens/haai-stack/blob/research/pstack-cli/docs/research/pstack-cli.md); repeated here only for comparison. The tool CLIs sit inside the `poteto-mode` skill: [`scripts/orch/orch.ts`](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/scripts/orch/orch.ts), `watch-pr/`, `check-plan.mjs`, `worktree-audit.sh`, plus `show-me-your-work/scripts/log.sh` ([tree](https://github.com/cursor/plugins/tree/main/pstack/skills/poteto-mode/scripts)). The orchestrate playbook defines the contract: "State reads and writes go through `scripts/orch/orch.ts` at drain points, one command in and one line out. The CLI never spawns, waits, or wakes anything", and the "canonical plain TSV and JSON stay readable without the CLI" ([orchestrate.md](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/playbooks/orchestrate.md)). `orch` keeps `units.tsv`, `ledger.tsv` keyed by PR+SHA, `frontier.json` and a pid lock file `.orch.lock` ([store.ts](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/scripts/orch/store.ts)). Dependencies install on first run through a lockfile-hash bootstrap ([bootstrap.ts](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/scripts/bootstrap.ts)).

### cursor-team-kit `control-cli`: not a tool CLI

The skill teaches the agent to "build or adapt a local harness to drive, inspect, and profile an interactive CLI or TUI". It does that by reusing the repo's own harness or assembling a temporary one from `tmux` (`new-session`, `send-keys`, `capture-pane`) or a short Python PTY script ([SKILL.md](https://github.com/cursor/plugins/blob/main/cursor-team-kit/skills/control-cli/SKILL.md)). The kit's tree contains no executable for this skill ([tree](https://github.com/cursor/plugins/tree/main/cursor-team-kit/skills/control-cli)). Under the glossary it does **not** count as a Tool CLI: it describes in prose how to improvise one. It is still a useful pattern for haai's live checks: a candidate for a real `haai`-style tool CLI that wraps tmux sessions with deterministic waits.

### beads (`bd`) and Gas Town (`gt`)

`bd` is a "distributed graph issue tracker for AI agents, powered by Dolt". Its core loop is `bd ready` → `bd update <id> --claim` → `bd close`, with JSON output and hash IDs to avoid merge collisions ([README](https://github.com/gastownhall/beads)). Skills reach it in one of two ways. `bd setup <host>` writes a skill, an `AGENTS.md` section and hooks per host (Cursor and Codex get `.agents/skills/beads/`). For Claude Code, a **SessionStart hook runs `bd prime --hook-json`**, which injects about 1–2k tokens of workflow context. The docs call `bd prime` "the single source of truth for operational workflow commands" ([ide-setup.md](https://github.com/gastownhall/beads/blob/main/docs/getting-started/ide-setup.md)). Distribution: a Go binary via curl installer, npm `@beads/bd` or PyPI `beads-mcp` ([README](https://github.com/gastownhall/beads)). The orchestrator is a separate project: Gas Town is a "multi-agent orchestration system" that keeps work state "in Beads ledger", runs roles in tmux, and supports Claude Code by default plus Codex, Copilot, Gemini and Cursor ([gastown README](https://github.com/gastownhall/gastown)).

### spec-kit (GitHub)

Each command template declares the script to run in its frontmatter, per shell (`sh: scripts/bash/setup-plan.sh --json`, `ps: …`, `py: …`) ([plan.md](https://github.com/github/spec-kit/blob/main/templates/commands/plan.md)). The scripts (`create-new-feature`, `setup-plan`, `setup-tasks`, `check-prerequisites`, `resolve-template`) exist in bash, PowerShell and Python ([scripts/](https://github.com/github/spec-kit/tree/main/scripts)). `specify` installs via `uv tool install specify-cli` ([README](https://github.com/github/spec-kit)). spec-kit also has an orchestrator: `specify workflow run <source>` chains "commands, prompts, shell steps, and human checkpoints", can be "paused and resumed from the exact point of interruption", emits `--json` (`run_id`, `status`, `current_step_id`) and stores state in `.specify/workflows/runs/<run_id>/` (`state.json`). `specify workflow status` lists runs ([workflows.md](https://github.com/github/spec-kit/blob/main/docs/reference/workflows.md)). Workflow steps take an `integration` input ("claude, copilot, gemini") (same source). How each integration is launched headlessly was not verified.

### OpenSpec

The CLI marks a set of commands as agent-facing, all with `--json`: `list`, `show`, `validate`, `status`, `instructions` ("Used by AI agents to understand what to create next"), `templates`, `new change`, `archive` (`--yes` for non-interactive use) ([cli.md](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)). The skills call it step by step. For example, the onboard skill runs `openspec instructions proposal --change "<name>" --json` and writes to the returned `resolvedOutputPath`, and it is told to read the JSON instead of retrying when the CLI exits non-zero ([openspec-onboard/SKILL.md](https://github.com/Fission-AI/OpenSpec/blob/main/skills/openspec-onboard/SKILL.md)). Distributed via `npm install -g @fission-ai/openspec`. Slash-command names are adapted per host (Cursor, Copilot, Amazon Q, Codex) ([README](https://github.com/Fission-AI/OpenSpec)).

### GSD (Get Shit Done)

`gsd-tools.cjs` was written explicitly to replace prose: it "replaces repetitive inline bash patterns across ~50 GSD command/workflow/agent files" and centralizes "config parsing, model resolution, phase lookup, git commits, summary verification". Subcommands cover `state` (including `signal-waiting`/`signal-resume` writing `WAITING.json`), `phase`, `roadmap`, `validate health --repair`, `progress json` and `audit-open --json`. It is now marked `@deprecated` in favour of `gsd-sdk query` ([gsd-tools.cjs](https://github.com/gsd-build/get-shit-done/blob/main/get-shit-done/bin/gsd-tools.cjs)). The SDK advertises "deterministic query/mutation handlers, plan execution, and event-stream telemetry so agents focus on judgment, not shell plumbing". Its CLI is `gsd-sdk query|run|init|auto`, and unknown queries fall back to `gsd-tools.cjs` ([sdk/README.md](https://github.com/gsd-build/get-shit-done/blob/main/sdk/README.md)). It depends on `@anthropic-ai/claude-agent-sdk` ([sdk/package.json](https://github.com/gsd-build/get-shit-done/blob/main/sdk/package.json)). This is the clearest example of one shared registry serving both the tool CLI and the orchestrator CLI. The main repo's last push was 2026-05-31; a successor [`gsd-build/gsd-2`](https://github.com/gsd-build/gsd-2) (7.8k stars) also stopped on 2026-05-22. Activity is unclear after that.

### BMAD-METHOD

Skills bundle PEP 723 Python scripts. The PO skill defines "`tickets.py` is `uv run {skill-root}/scripts/tickets.py --project-root {project-root}`" and tells the agent to "take [state] from the `tickets.py` row, never derive it" ([workflow.md](https://github.com/bmad-code-org/BMAD-METHOD/blob/main/skills/bmad-agent-po/workflow.md)). `tickets.py` reads a ticket tree and "answer[s] what is next" ([tickets.py](https://github.com/bmad-code-org/BMAD-METHOD/blob/main/skills/bmad-agent-po/scripts/tickets.py)). `memlog.py` is an append-only, write-only log whose every command "echoes the new state as one line of JSON, so the caller never re-reads the file" ([memlog.py](https://github.com/bmad-code-org/BMAD-METHOD/blob/main/skills/bmad/scripts/memlog.py)). Installed via `npx bmad-method install` or as Claude Code and Codex plugins ([README](https://github.com/bmad-code-org/BMAD-METHOD)). No orchestrator CLI was found in the tree; this is not exhaustively verified.

### Superpowers

The bundled scripts are small, single-purpose bash. `sdd-workspace` resolves a per-plan workspace (`.superpowers/sdd/<plan>/`) holding "task briefs, implementer reports, review packages, and the progress ledger". `task-brief` extracts one task's text "so the task text never has to be pasted through the controller's context". `task-start` merges two steps "because every call in an inline session is a turn that re-reads the whole context" ([scripts](https://github.com/obra/superpowers/tree/main/skills/subagent-driven-development/scripts), [task-start](https://github.com/obra/superpowers/blob/main/skills/executing-plans/scripts/task-start)). SKILL.md calls them as `bash scripts/review-package PLAN_FILE BASE HEAD` ([SKILL.md](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md)). The README lists install paths for about 15 hosts, including Claude Code, Codex, Cursor, Gemini CLI, Copilot CLI, OpenCode and Pi ([README](https://github.com/obra/superpowers)).

### ccpm

The skill sets the rule outright: "For deterministic operations — anything that reads and reports without needing reasoning — always run the bash script directly rather than doing the work manually". A table maps intents to `bash references/scripts/{status,standup,next,blocked,validate,…}.sh` ([SKILL.md](https://github.com/automazeio/ccpm/blob/main/skill/ccpm/SKILL.md)). State lives in GitHub Issues (using the `gh-sub-issue` extension for parent/child) and in git worktrees for parallel agents. ccpm is "harness-agnostic" via Agent Skills ([README](https://github.com/automazeio/ccpm)). It is the closest existing match to haai's GitHub-Issues-as-tracker setup.

### Backlog.md

Markdown task files managed by the `backlog` CLI. `backlog init` writes a short instruction file "that tells agents to run `backlog instructions overview`", with MCP as an alternative. The README says to "prefer Backlog.md commands (CLI/MCP/Web) over hand-editing task files, so field types and metadata stay consistent" ([README](https://github.com/MrLesk/Backlog.md)). Distributed via npm/bun, Homebrew and Nix (same source).

### Task Master

`task-master parse-prd | list | next | show | move | research`. The README recommends MCP as "Option 1", with the CLI as "Option 2" ([README](https://github.com/eyaltoledano/claude-task-master)). Last commit 2026-04-23 (GitHub API); the docs now live on `tryhamster.com`. Treat it as a pattern-setter that is moving to a commercial product (inference, not verified).

### HumanLayer `.claude/commands`

Historically influential. `ralph_impl.md` creates a worktree via `./hack/create_worktree.sh`, then **launches a new headless session** with `humanlayer-nightly launch --model opus … -w ~/wt/humanlayer/ENG-XXXX "/implement_plan …"`. `create_handoff.md` ends with `humanlayer thoughts sync` ([ralph_impl.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/ralph_impl.md), [create_handoff.md](https://github.com/humanlayer/humanlayer/blob/main/.claude/commands/create_handoff.md)). It is the only surveyed example where a *skill* calls the *orchestrator CLI*. The README now says "the code here is pretty much all deprecated" ([README](https://github.com/humanlayer/humanlayer)).

### Ralph (snarktank)

"An autonomous AI agent loop that runs AI coding tools (Amp or Claude Code) repeatedly until all PRD items are complete. Each iteration is a fresh instance with clean context. Memory persists via git history, `progress.txt`, and `prd.json`." The `prd` and `ralph` skills produce the input that `ralph.sh` consumes ([README](https://github.com/snarktank/ralph)).

### Sandcastle (Matt Pocock)

A TypeScript orchestrator library, not a tool CLI. `run({ agent, sandbox, promptFile, maxIterations, completionSignal, logging })` runs agents in Docker, Podman, Vercel or no sandbox, on branches or worktrees, and returns `iterations`, `commits` and `logFilePath`. Logs default to `.sandcastle/logs/`. Agent providers are `claudeCode`, `codex`, `cursor`, `opencode` and `pi`. `sandcastle init` asks for an issue tracker (GitHub Issues, Beads or Custom) and a template ([README](https://github.com/mattpocock/sandcastle)). It is the natural orchestrator companion to Matt's skills, which themselves ship only a template script ([`hitl-loop.template.sh`](https://github.com/mattpocock/skills/blob/main/skills/engineering/diagnosing-bugs/scripts/hitl-loop.template.sh)) and a git-guardrail hook ([tree](https://github.com/mattpocock/skills)).

### anthropics/skills and the Agent Skills spec

The spec makes `scripts/` an optional directory of "executable code that agents can run", which should "be self-contained or clearly document dependencies" and "include helpful error messages" ([agentskills.io/specification](https://agentskills.io/specification)). Anthropic's own skills follow it: PDF form-filling, docx/pptx editing, and skill-creator's `run_eval.py`/`run_loop.py` ([repo](https://github.com/anthropics/skills)). This convention is why "bundle the script in the skill folder" has become the default.

## Recurring patterns

1. **Tool CLI as single writer of state.** The rule is "use the CLI, don't hand-edit": pstack `orch` ("every file has exactly one writer"), BMAD ("take it from the `tickets.py` row, never derive it"), Backlog.md, beads, GSD `state …`.
2. **JSON-first contract.** `--json` on every agent-facing command (OpenSpec, spec-kit scripts, Backlog, `bd`, GSD `progress json`), often with "one command in, one line out" (pstack, BMAD memlog).
3. **The CLI tells the agent what to do next.** `openspec instructions`, `bd prime`, `backlog instructions overview`, `tickets.py next`, `task-master next`, ccpm `next.sh`. The CLI becomes the source of workflow prose, so skills stay short.
4. **Context injection by hook.** beads wires `bd prime` into SessionStart, so the tool CLI rather than a static file primes the session ([ide-setup.md](https://github.com/gastownhall/beads/blob/main/docs/getting-started/ide-setup.md)).
5. **Fewer turns.** Superpowers merges calls "because every call … re-reads the whole context"; GSD exists to remove repeated inline bash.
6. **Two distribution modes.** Bundled scripts in bash, Python with uv/PEP 723, or Bun/Node with self-install travel with the skill and need no install step. Standalone binaries (Go, npm) give a stable command name, versioning and cross-repo use.
7. **Orchestrator kept apart from the tool CLI.** pstack forbids `orch` from spawning. Orchestration is a separate product (Gas Town, Sandcastle, Ralph) or a separate subcommand family (`gsd-sdk auto`, `specify workflow run`).

## Gaps measured against haai's three pains

**Deterministic flow steps.** The pattern is solved within each stack, but every stack reinvents it, and Matt's skills (haai's base) ship almost no tool CLIs. The closest fit for haai's tracker (GitHub Issues plus the triage-label vocabulary) is ccpm's bash scripts, which are read-mostly reporting. None of the surveyed stacks ships a typed tool CLI for issue-tracker *mutations* (claim, label transitions, close-with-comment, link to parent map) as the counterpart of `bd` for GitHub Issues. That is unverified as an exhaustive claim.

**AFK orchestration.** Orchestrators exist (Sandcastle, Gas Town, Ralph, `gsd-sdk auto`, `specify workflow run`, `humanlayer launch`), but only GSD, spec-kit and beads with Gas Town share state with their own tool CLIs. Sandcastle has tracker templates but no tool CLI for the agent inside the sandbox. pstack has the tool CLI but deliberately no external orchestrator; it orchestrates in-chat with subagents. Nothing surveyed pairs Matt-style skills, a GitHub-Issues tool CLI and an outside orchestrator around one shared state model.

**Observability and state.** State stores vary: Dolt (beads), markdown or frontmatter (Backlog, GSD STATE.md, OpenSpec), TSV/JSON (pstack), GitHub Issues (ccpm), run `state.json` (spec-kit), log files (Sandcastle). Resumable runs exist only in spec-kit (`workflow resume`) and partly in GSD (`WAITING.json` signals); GSD's "event-stream telemetry" is advertised but its format was not verified. No project offers a cross-session **run history keyed to tracker issues** (which agent touched which issue, with which verdict and transcript) that both a human and the orchestrator can read. pstack's PR+SHA verdict ledger comes closest.

## Unverified or open

- How spec-kit launches each integration headlessly inside `workflow run` (the docs mention a "prompt step's CLI subprocess" but this was not traced in code).
- Whether BMAD has any orchestrator CLI; only a tree scan was done.
- GSD's maintenance status after 2026-05; the format of its telemetry event stream.
- Task Master's future as open source (docs moved to tryhamster.com).

## Sources

All links inline above. Repo metadata (stars, created, last push) came from `gh api repos/<owner>/<repo>` on 2026-10-19.
