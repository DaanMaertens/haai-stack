# What Claude Code offers an orchestrator CLI

Research for [#6](https://github.com/DaanMaertens/haai-stack/issues/6), part of the map [#2](https://github.com/DaanMaertens/haai-stack/issues/2).

**TL;DR**

1. Claude Code is a usable **agent host** for an external **orchestrator CLI**: `claude -p` runs a full agent loop non-interactively, emits NDJSON events (`--output-format stream-json`), accepts NDJSON input (`--input-format stream-json`), and ends with a `result` event carrying `session_id`, cost, turns, `permission_denials` and an error subtype.
2. Sessions are addressable: `--session-id <uuid>` to pick the ID up front, `--resume <id>`/`--continue` to add turns, `--fork-session` to branch, `--worktree` for git isolation. Hooks (`PreToolUse` `defer`) let an orchestrator pause a run at a tool call and resume it later.
3. Unattended safety is explicit, not automatic: since v2.1.302, `-p` starts in Manual mode, so any call needing approval is denied unless `--allowedTools`, `--permission-mode` (`dontAsk`/`acceptEdits`/`auto`/`bypassPermissions`), a hook, or `--permission-prompt-tool` allows it. `--permission-prompts none` stops runs waiting on a host. Caps: `--max-turns`, `--max-budget-usd` (client-side estimate, can overshoot).
4. Skills work headlessly: put `/skill-name` in the `-p` prompt and Claude Code expands it, and forked-skill/subagent messages appear in the stream tagged by `parent_tool_use_id`. `--bare` skips all discovery (skills, hooks, CLAUDE.md, MCP), so it is unusable for a **skill stack** unless skills come in via `--add-dir` or `--plugin-dir`.
5. Observability from outside: the stream itself; hooks (≈30 events, command/HTTP/MCP/prompt/agent handlers; some can block); OpenTelemetry metrics/events/traces (traces beta, with `TRACEPARENT` propagation into `-p`); and `claude agents --json` for background sessions. On-disk transcripts (`~/.claude/projects/...jsonl`) exist but their format is explicitly unstable.

Sources: official docs at `code.claude.com/docs/en/*` (Markdown versions fetched 2026-10-19; the pages carry no per-page date), the [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) (latest entry 2.1.303 at time of fetch, matching the locally installed `claude --version`), npm `@anthropic-ai/claude-agent-sdk` 0.3.303, PyPI `claude-agent-sdk` 0.2.168. Many features carry "requires v2.1.x" notes; they are reproduced where they matter. The release cadence is fast, so pin a minimum version in the harness.

Terminology follows `GLOSSARY.md`: Claude Code is the **agent host**; what haai-stack would build is an **orchestrator CLI** (drives headless host sessions from outside) plus **tool CLIs** (called by skills from inside).

---

## 1. Headless mode (`claude -p`)

- `-p`/`--print` runs any `claude` invocation non-interactively. Exit code is 0 on success, non-zero on failure. Invalid flags go to stderr before the run. Failures inside the run (e.g. missing auth) are printed **as the result on stdout**. ([headless](https://code.claude.com/docs/en/headless#basic-usage))
- `--bg` is rejected with `-p`. Background sessions are a separate surface (see §11). ([headless](https://code.claude.com/docs/en/headless#basic-usage), [agent-view](https://code.claude.com/docs/en/agent-view#dispatch-an-agent-from-your-shell))
- Piped stdin is capped at 10 MB. ([headless](https://code.claude.com/docs/en/headless#pipe-data-through-claude))
- **Without `--bare`, `-p` loads everything an interactive session would**: hooks, skills, commands, subagents, plugins, MCP, auto memory, CLAUDE.md, from the cwd and `~/.claude`. It also **skips the workspace-trust dialog**, so a repo's `.claude/settings.json` hooks and `.mcp.json` servers run even in an untrusted folder. ([headless](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode))
- `--bare` skips all of that auto-discovery. It needs `ANTHROPIC_API_KEY` or `apiKeyHelper` because it never reads OAuth or keychain credentials, and it offers only Bash/read/edit tools. Context comes back in through flags: `--append-system-prompt[-file]`, `--settings`, `--mcp-config`, `--agents`, `--plugin-dir`/`--plugin-url`. Skills in an `--add-dir` directory's `.claude/skills/` still load, but there are no system reminders, so Claude isn't given the skill list, and there are no background tasks. The docs say `--bare` "will become the default for `-p` in a future release." ([headless](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode), [cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
  - **Implication for haai-stack:** a skill stack driven by an orchestrator either runs non-bare (and inherits whatever the user's `~/.claude` holds, which hurts reproducibility) or runs `--bare` and ships its skills as a plugin via `--plugin-dir`. Whether a `--plugin-dir` plugin's skills are listed to Claude under `--bare` is **not stated** in the docs; it needs an experiment.
- `--restricted` (v2.1.248+) is meant for evaluation harnesses. It removes command/code-running tools and WebFetch unless named in `--tools`, confines file tools to the working dirs, loads only managed settings and `--settings`, and refuses `bypassPermissions`. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- **Exit behaviour:** after the final turn, a `-p` run waits for background work (background commands, background subagents/workflows, Monitor watches, pending `/loop` wakeups). It drops idle waits after 10 minutes by default (`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`). ([headless](https://code.claude.com/docs/en/headless#background-tasks-at-exit))
- **SIGTERM** exits 143, leaves the in-progress turn unfinished with no result, kills running Bash process trees, and runs only `SessionEnd` hooks. To end a turn cleanly, send SIGINT (or call the SDK's `interrupt()`) first. ([headless](https://code.claude.com/docs/en/headless#stop-a-run-with-sigterm))

## 2. Output formats and the `stream-json` event schema

- `--output-format text | json | stream-json`. `json` returns one object with `result`, `session_id`, usage and `total_cost_usd` (plus a per-model breakdown). `stream-json` is newline-delimited JSON whose **last line is a `result` message**. ([headless](https://code.claude.com/docs/en/headless#get-structured-output))
- `stream-json` needs `--verbose` in the documented examples. `--include-partial-messages` adds token deltas (`type: "stream_event"`). ([headless](https://code.claude.com/docs/en/headless#stream-responses))
- `--json-schema '<schema>'` with `--output-format json` puts schema-validated output in `structured_output`. The `format` keyword is accepted but not enforced. ([headless](https://code.claude.com/docs/en/headless#get-structured-output))
- **Event schema:** the CLI stream carries the Agent SDK's `SDKMessage` union. The headless page links its events to the TS reference types (`SDKSystemMessage`, `SDKInformationalMessage`, `SDKHookStartedMessage`, …). The union currently has ~36 variants: `assistant`, `user`, user replay, `result`, `system/init`, partial assistant (`stream_event`), compact boundary, status, hook started/progress/response, plugin install, tool progress, auth status, task started/progress/updated/notification, background-tasks-changed, session-state-changed, permission denied, prompt suggestion, API retry, rate-limit event, and more. ([TS reference: SDKMessage](https://code.claude.com/docs/en/agent-sdk/typescript#sdkmessage))
  - `system/init` (first event, unless `SessionStart`/`Setup` hook events or `plugin_install` events come first) gives `session_id`, `cwd`, `model`, `permissionMode`, `tools`, `mcp_servers` (+`mcp_server_errors`), `slash_commands`, `skills`, `agents`, `plugins` (+`plugin_errors`), `claude_code_version` and a `capabilities` array for feature detection. ([headless](https://code.claude.com/docs/en/headless#read-session-metadata), [TS SDKSystemMessage](https://code.claude.com/docs/en/agent-sdk/typescript#sdksystemmessage))
  - `result` has `subtype` `success | error_max_turns | error_during_execution | error_max_budget_usd | error_max_structured_output_retries`, plus `is_error`, `num_turns`, `duration_ms`, `total_cost_usd`, `usage`, `modelUsage`, `permission_denials`, `stop_reason`, `structured_output`, `deferred_tool_use` and `terminal_reason`. `result` text is only on `success`. ([TS SDKResultMessage](https://code.claude.com/docs/en/agent-sdk/typescript#sdkresultmessage), [agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop#handle-the-result))
  - `system/api_retry` events carry attempt, delay, HTTP status and error category. ([headless](https://code.claude.com/docs/en/headless#handle-api-retries))
  - Permission denials stream as `permission_denied` system messages and are listed in `result.permission_denials`. ([headless](https://code.claude.com/docs/en/headless#turn-off-permission-prompts-in-unattended-runs))
  - **Subagents/forked skills:** their messages appear as `assistant`/`user` with `parent_tool_use_id` set. By default you get only their `tool_use`/`tool_result` blocks; add `--forward-subagent-text` for text and thinking blocks. A forked skill invoked as `/<skill>` in the prompt is tagged `forked-command-…`, and its messages arrive **after it finishes**. Minimum versions apply (v2.1.219 for all nesting depths, v2.1.287 for prompt-invoked forked skills). ([headless](https://code.claude.com/docs/en/headless#follow-subagent-messages))
  - `--include-hook-events` adds hook lifecycle events (SessionStart/Setup hooks are always included). ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- The docs warn that a slow stream consumer delays exit by up to 30 s. ([headless](https://code.claude.com/docs/en/headless#stream-responses))

## 3. Input streaming

- `--input-format stream-json` makes `-p` read NDJSON user messages from stdin (shape of `SDKUserMessage`: `{type:"user", message:{role,content}, parent_tool_use_id:null}`). This is the "streaming input mode" the SDK recommends: one long-lived process with queued messages, interrupts and images. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags), [streaming input](https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode))
- `--replay-user-messages` echoes stdin user messages to stdout for acknowledgement. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- With stream input, `--max-turns` resets per queued message, while the budget keeps accumulating. ([agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop#turns-and-budget))
- **Gap:** the stdin side also carries a **control protocol** (`control_request`/`control_response`, `initialize`, `interrupt`, `register_repo_root`, permission requests). The TS reference mentions it ("a client that drives the CLI's control protocol directly"), but there is **no standalone spec page** for it. Driving it from a non-SDK orchestrator means reverse-engineering the SDK. ([TS reference](https://code.claude.com/docs/en/agent-sdk/typescript#query-object))

## 4. Sessions: resume, continue, fork

- `--continue` takes the most recent session in the cwd (`-p --continue` includes `-p`/SDK sessions). `--resume <id|name|path.jsonl>` resumes by ID across all projects on the machine (v2.1.223+). `--session-id <uuid>` presets the ID. `--fork-session` with resume/continue creates a new ID. `--name` names the session. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags), [headless](https://code.claude.com/docs/en/headless#continue-conversations), [sessions](https://code.claude.com/docs/en/sessions#branch-a-session))
- **What resume does not restore:** `--mcp-config`, `--settings`, `--plugin-dir`, `--fallback-model` and `--add-dir` must be passed again. With `-p`, the stored permission mode is **not** restored either, so pass `--permission-mode` again. ([sessions](https://code.claude.com/docs/en/sessions#what-a-resumed-session-restores), [hooks: defer note](https://code.claude.com/docs/en/hooks#defer-a-tool-call-for-later))
- Resuming the same session in two processes without forking interleaves both into one transcript. ([sessions](https://code.claude.com/docs/en/sessions#branch-a-session))
- Interrupted tool calls are not re-run on resume; Claude is told to check whether they took effect (`CLAUDE_CODE_RESUME_INTERRUPTED_TURN=1` changes this). ([sessions](https://code.claude.com/docs/en/sessions#what-a-resumed-session-restores))
- `--no-session-persistence` disables writing to disk for one `-p` run. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- **Pause/resume at a tool call:** a `PreToolUse` hook returning `permissionDecision: "defer"` (honoured only in `-p`) makes the process exit with `stop_reason: "tool_deferred"` and a `deferred_tool_use` payload. The orchestrator later runs `claude -p --resume <id>`, and the same hook can then `allow` with `updatedInput`. This only works when Claude made a single tool call that turn. There is no timeout, and the session is subject to the 30-day `cleanupPeriodDays` sweep. ([hooks](https://code.claude.com/docs/en/hooks#defer-a-tool-call-for-later))
- The SDK adds `listSessions()`, `getSessionMessages()`, `forkSession()`, `tagSession()` and a `SessionStore` adapter to mirror transcripts to external storage so another host can resume them. ([TS reference](https://code.claude.com/docs/en/agent-sdk/typescript), [session storage](https://code.claude.com/docs/en/agent-sdk/session-storage))

## 5. Permissions for unattended runs

- **Default:** since v2.1.302, `-p` and SDK runs with no mode set start in **Manual** (`default`). Calls needing approval are **denied** unless an allow rule, hook, `canUseTool`, or `--permission-prompt-tool` approves them. ([permission-modes](https://code.claude.com/docs/en/permission-modes#which-mode-a-non-interactive-run-starts-in), [headless](https://code.claude.com/docs/en/headless#auto-approve-tools))
- Modes: `default`, `acceptEdits`, `plan`, `auto` (classifier reviews each action), `dontAsk` (deny anything that would prompt, "locked-down CI"), and `bypassPermissions` ("isolated containers and VMs only"). ([permission-modes](https://code.claude.com/docs/en/permission-modes#available-modes))
- `--allowedTools` / `--disallowedTools` / `--tools` use permission-rule syntax, e.g. `Bash(git diff *)`. The space before `*` matters. In `auto` mode a bare `Bash` allow is dropped. ([headless](https://code.claude.com/docs/en/headless#create-a-commit), [cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- `--permission-prompts none` (v2.1.259+) means: don't wait for a host, deny unresolved prompts, tell Claude not to retry, and remove `AskUserQuestion`. ([headless](https://code.claude.com/docs/en/headless#turn-off-permission-prompts-in-unattended-runs))
- `--permission-prompt-tool <mcp tool>` delegates approvals to an MCP tool, which is how an orchestrator could implement a human-in-the-loop queue. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- **Never auto-approved, even in bypass:** explicit `ask` rules, `AskUserQuestion`, MCP tools marked `requiresUserInteraction`, and `rm`/`rmdir` of critical paths. ([permission-modes](https://code.claude.com/docs/en/permission-modes#actions-no-mode-auto-approves))
- **Auto-mode gotchas in `-p`:** the 3-in-a-row / 20-total classifier block thresholds can't fall back to a prompt, so the action is skipped and Claude keeps working without stopping the run. Auto mode needs a supported model and can be disabled by org or server-side; the session then starts in Manual instead. ([permission-modes](https://code.claude.com/docs/en/permission-modes#repeated-block-thresholds), [permission-modes](https://code.claude.com/docs/en/permission-modes#which-mode-a-session-starts-in))
- `bypassPermissions` is refused as root/sudo outside a recognised sandbox. ([permission-modes](https://code.claude.com/docs/en/permission-modes#skip-all-checks-with-bypasspermissions-mode))
- A project skill's `allowed-tools` frontmatter is honoured in `-p` even in untrusted folders. ([skills](https://code.claude.com/docs/en/skills))

## 6. Hooks (control and observability)

- **Events:** `SessionStart`, `Setup`, `UserPromptSubmit`, `UserPromptExpansion`, `PreToolUse`, `PermissionRequest`, `PermissionDenied`, `PostToolUse`, `PostToolUseFailure`, `PostToolBatch`, `Notification`, `MessageDisplay`, `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `Stop`, `StopFailure`, `TeammateIdle`, `InstructionsLoaded`, `ConfigChange`, `CwdChanged`, `DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PreCompact`, `PostCompact`, `PreModelSwitch`, `PostModelSwitch`, `Elicitation`, `ElicitationResult`, `SessionEnd`. ([hooks](https://code.claude.com/docs/en/hooks#hook-lifecycle))
- **Can block (exit 2 or JSON decision):** `PreToolUse` (allow/deny/ask/defer, plus `updatedInput` to rewrite tool input), `UserPromptSubmit`, `UserPromptExpansion`, `Stop`/`SubagentStop` (forces Claude to continue, capped at 8 consecutive continuations), `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `ConfigChange`, `PostToolBatch` (stops the loop), `PreCompact`, `PreModelSwitch`, `Elicitation*`, `WorktreeCreate`/`Remove`. `PermissionRequest` decides through its JSON `decision` object, not exit 2. The rest are observe-only. ([hooks: exit code 2 table](https://code.claude.com/docs/en/hooks#exit-code-2-behavior-per-event), [Stop decision control](https://code.claude.com/docs/en/hooks#stop-decision-control))
- **Handler types:** `command` (JSON on stdin), `http` (POST JSON to a URL, which suits an orchestrator listening locally), `mcp_tool`, `prompt` (single-turn LLM judgment), and `agent` (experimental). Matching hooks run in parallel. The default timeout is 600 s for command/http. ([hooks](https://code.claude.com/docs/en/hooks#hook-handler-fields))
- **Common input fields:** `session_id`, `prompt_id` (matches OTel `prompt.id`), `transcript_path` (written asynchronously and may lag; use `last_assistant_message` on Stop), `cwd`, `permission_mode`, `agent_id`, `agent_type`. ([hooks](https://code.claude.com/docs/en/hooks#common-input-fields))
- **JSON output:** `continue:false` + `stopReason` halts Claude, `systemMessage` surfaces in stream-json as an informational message, and `additionalContext` injects context (capped at 10,000 chars). ([hooks](https://code.claude.com/docs/en/hooks#json-output))
- **Where hooks live:** user/project/local settings, managed policy, plugin `hooks/hooks.json`, and skill or subagent frontmatter. Entries merge across levels. Hooks also fire inside subagents, tagged with `agent_id`. ([hooks](https://code.claude.com/docs/en/hooks#hook-locations))
- `/goal` is a built-in session-scoped prompt-based Stop hook and works in `-p` (`claude -p "/goal <condition>"` loops until the condition is judged met). ([hooks](https://code.claude.com/docs/en/hooks#stop), [goal](https://code.claude.com/docs/en/goal))
- **Implication:** hooks are the in-session seam where a skill stack can enforce **deterministic flow steps** (`PreToolUse` gates, `Stop` "run the tests first"). They are also the cheapest observability tap (an `http` hook to the orchestrator).

## 7. Settings scopes

- Precedence, highest first: **managed** (file, MDM, or server-managed) > **command line** (`--settings` JSON, per session) > **project local** `.claude/settings.local.json` > **shared project** `.claude/settings.json` > **user** `~/.claude/settings.json`. List keys such as `permissions.allow` **merge** across levels. ([settings](https://code.claude.com/docs/en/settings#settings-precedence))
- `--setting-sources user,project,local` restricts which file sources load. The SDK equivalent is `settingSources`/`setting_sources`, and skills only load if `user`/`project` are included. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags), [SDK skills](https://code.claude.com/docs/en/agent-sdk/skills))
- `CLAUDE_CONFIG_DIR` (+ `CLAUDE_CODE_PROJECT_DIR_NAME`, v2.1.234+) relocates all state per tenant or run. That is useful for isolating an orchestrator's runs from the user's own `~/.claude`. ([sessions](https://code.claude.com/docs/en/sessions#name-the-project-directory-yourself))

## 8. Subagents and skills, headlessly

- **Slash commands/skills in `-p`:** "User-invoked skills and custom commands work. Include `/skill-name` in the prompt string and Claude Code expands it before running." Terminal-only built-ins (`/login`) are unavailable, while `/model x`, `/config k=v`, etc. accept arguments. ([headless note](https://code.claude.com/docs/en/headless#create-a-commit))
- `system/init.slash_commands` and `.skills` list what is invocable in that session. ([SDK skills](https://code.claude.com/docs/en/agent-sdk/skills#commands-in-agent-sdk-sessions))
- Skill frontmatter relevant to orchestration: `disable-model-invocation: true` makes a skill manual-only, `context: fork` runs it in a subagent, `agent:` picks the subagent type, and `background:` sets wait vs background. ([skills](https://code.claude.com/docs/en/skills))
- Subagents: `--agents '<json|file>'` defines them per run. `--agent <name>` runs a subagent as the main thread. Nesting goes 3 layers deep by default (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`). `isolation: worktree` in frontmatter gives each subagent its own worktree. `--append-subagent-system-prompt` exists for `-p`. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags), [sub-agents](https://code.claude.com/docs/en/sub-agents#let-subagents-spawn-their-own-subagents), [worktrees](https://code.claude.com/docs/en/worktrees#isolate-subagents-with-worktrees))
- A skill that downloads at sync time may not be ready in a short `-p` run. ([skills](https://code.claude.com/docs/en/skills))

## 9. Worktrees

- `claude -w <name>` / `--worktree` creates `<repo>/.claude/worktrees/<name>`. `-p` skips the trust check. You can also branch from a PR (`-w '#123'`). The base is the remote default branch (`worktree.baseRef: "fresh"`) or `"head"`, but **not an arbitrary branch** (use `git worktree add` yourself for that). ([worktrees](https://code.claude.com/docs/en/worktrees), [cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- **`-p` does not clean up its worktrees** and leaves a `git worktree lock` until a later stale-lock sweep. The orchestrator must remove them itself. ([worktrees](https://code.claude.com/docs/en/worktrees#clean-up-worktrees))
- Resume returns to the worktree (v2.1.212+). In `-p`, any worktree refusal other than "gone" aborts the resume with a stderr error. ([worktrees](https://code.claude.com/docs/en/worktrees#the-session-resumes-outside-its-worktree))
- **Isolation enforcement:** inside a worktree session, Claude Code refuses edits to the main checkout, commands whose cwd is the main checkout, git redirects into it, and **any Bash command whose git usage it can't statically verify** (computed command names, loops and the like). Skills that shell out to git in compound commands will see refusals and must use plain, separate commands. (Observed first-hand while writing this note.) ([worktrees](https://code.claude.com/docs/en/worktrees#how-claude-code-enforces-isolation))
- `WorktreeCreate`/`WorktreeRemove` hooks replace git worktree creation, for example for non-git VCS or custom layouts. ([worktrees](https://code.claude.com/docs/en/worktrees#non-git-version-control))

## 10. Cost and turn limits

- `--max-turns N` (no default limit) ends with `error_max_turns`. `--max-budget-usd X` ends with `error_max_budget_usd`. Both are print-mode only. ([cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- The budget is checked **after** each response, so spend can overshoot by one response plus running subagents ("leave headroom"). Subagent spend counts, and at the cap new subagents fail and background ones stop (v2.1.217+). On resume, earlier runs' totals don't count toward the cap. ([agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop#budget-headroom), [cli-reference](https://code.claude.com/docs/en/cli-reference#cli-flags))
- `total_cost_usd` is a client-side estimate, not billing. With `--resume` it reports the conversation's whole total. ([headless](https://code.claude.com/docs/en/headless#pipe-data-through-claude), [cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking))
- On subscription plans the cost figure "isn't relevant for billing"; session and weekly limits apply instead. ([costs](https://code.claude.com/docs/en/costs))

## 11. Background sessions and agent view (first-party orchestration)

- `claude --bg "<prompt>"` starts a supervised background session (not `-p`), moves it into a worktree before edits, and prints a short ID. `claude agents --json [--all]` is "the supported way to read session state from outside Claude Code": `state` (`working | blocked | done | failed | stopped`), `status`, `waitingFor`, `sessionId`. `claude logs | attach | stop | respawn | rm <id>` manage sessions. The files in `~/.claude/jobs/<id>/` are **not** a stable interface. **Research preview.** ([agent-view](https://code.claude.com/docs/en/agent-view#manage-sessions-from-the-shell))
- `--bg` in an untrusted directory without a TTY exits with `Workspace not trusted`. ([agent-view](https://code.claude.com/docs/en/agent-view#dispatch-an-agent-from-your-shell))
- **Implication:** this overlaps with what an orchestrator CLI would do (dispatch, track, isolate). haai-stack should decide whether to build on it, wrap it, or ignore it.

## 12. Agent SDK vs shelling out to the CLI

- The Agent SDK (TS `@anthropic-ai/claude-agent-sdk`, Python `claude-agent-sdk`) is "a library that runs the Claude Code binary". The TS SDK **bundles** a native binary as an optional dependency, and its version tracks the CLI (SDK 0.3.191 ↔ CLI 2.1.191). Python takes a `cli_path`. ([overview](https://code.claude.com/docs/en/agent-sdk/overview), [TS reference](https://code.claude.com/docs/en/agent-sdk/typescript), [Python reference](https://code.claude.com/docs/en/agent-sdk/python))
- Anthropic's own guidance: for languages other than Python/TS, "run the CLI as a subprocess with `-p` and `--output-format json`." ([overview](https://code.claude.com/docs/en/agent-sdk/overview))
- **SDK-only extras:** in-process `canUseTool` callback, programmatic hooks, in-process MCP tools, `interrupt()`, typed messages, `listSessions`/`forkSession`, the `SessionStore` adapter, file checkpointing/rewind, and `spawnClaudeCodeProcess` (run the CLI in a VM or container). ([TS reference](https://code.claude.com/docs/en/agent-sdk/typescript#options), [sessions](https://code.claude.com/docs/en/agent-sdk/sessions))
- **Auth caveat:** "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK." Use API keys instead. How this applies to a personal or open-source orchestrator that drives the user's own logged-in `claude` CLI is **not stated** in the docs. ([overview](https://code.claude.com/docs/en/agent-sdk/overview))
- For CI, `claude setup-token` produces a long-lived subscription OAuth token (`CLAUDE_CODE_OAUTH_TOKEN`). ([cli-reference](https://code.claude.com/docs/en/cli-reference), [env-vars](https://code.claude.com/docs/en/env-vars))

## 13. GitHub Actions

- `anthropics/claude-code-action@v1` has two modes. **Interactive** (no `prompt`) responds to `@claude` mentions. **Automation** (`prompt:` given) runs on any event, including cron. `prompt` can be a skill invocation, e.g. `/code-review:code-review …`. `claude_args` passes any CLI flag (`--max-turns`, `--allowedTools`, `--model`). Auth is `anthropic_api_key` or `claude_code_oauth_token`. Other inputs: `settings`, `plugins`, `plugin_marketplaces`. ([github-actions](https://code.claude.com/docs/en/github-actions), [action inputs](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md#inputs))
- Guardrails: the triggering user needs write access, and bot actors are rejected unless listed in `allowed_bots` (loop prevention). ([github-actions](https://code.claude.com/docs/en/github-actions#who-can-trigger-runs))
- With a plain-text prompt in automation mode, Claude has no shell or GitHub API access until tools are granted. ([github-actions](https://code.claude.com/docs/en/github-actions#run-on-a-schedule))

## 14. What is observable from outside a session

| Channel | What you get | Stability |
| - | - | - |
| `stream-json` stdout | Every message, tool call/result, subagent messages (`parent_tool_use_id`), retries, denials, final cost/turns | Typed via SDK reference; versioned features |
| Hooks (`http`/`command`) | Lifecycle events with `session_id`, `prompt_id`, `agent_id`, tool input | Documented contract |
| OpenTelemetry | Metrics (`claude_code.session.count`, `cost.usage`, `token.usage`, `lines_of_code.count`, `commit.count`, `pull_request.count`, `active_time.total`, …), log events (`user_prompt`, `tool_result`, `api_request`, `tool_decision`, `skill_activated`, …), traces (**beta**: `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`) with span tree interaction → llm_request / tool / hook. Prompt and tool content redacted by default (`OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`). `-p` honours inbound `TRACEPARENT`, so an orchestrator can parent Claude's spans; Bash subprocesses inherit `TRACEPARENT`. | Documented; traces beta |
| `claude agents --json` | State of background sessions | Supported interface, research preview |
| Transcripts `~/.claude/projects/<cwd-slug>/<session>.jsonl` | Full history; resumable by path | **Format is internal and "changes between versions"**; 30-day retention default |

Sources: [monitoring](https://code.claude.com/docs/en/monitoring-usage), [monitoring: traces](https://code.claude.com/docs/en/monitoring-usage#traces-beta), [sessions: transcripts](https://code.claude.com/docs/en/sessions#where-transcripts-are-stored), [agent-view](https://code.claude.com/docs/en/agent-view#read-session-state-from-a-script).

## 15. AFK gotchas (summary)

1. `-p` without permission flags now denies by default (v2.1.302). Silent denials show up only in `permission_denials`, so check them. ([permission-modes](https://code.claude.com/docs/en/permission-modes#which-mode-a-non-interactive-run-starts-in))
2. Non-bare `-p` runs repo hooks and MCP servers without a trust prompt, which is a supply-chain surface when orchestrating over arbitrary repos. ([headless](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode))
3. `--bare` drops skills/CLAUDE.md/hooks, which conflicts with a skill stack. ([headless](https://code.claude.com/docs/en/headless#start-faster-with-bare-mode))
4. Resume with `-p` doesn't restore the permission mode or the per-launch flags. ([sessions](https://code.claude.com/docs/en/sessions#what-a-resumed-session-restores))
5. Worktrees made by `-p` are not cleaned up. ([worktrees](https://code.claude.com/docs/en/worktrees#clean-up-worktrees))
6. Budget overshoots, and cost is an estimate. ([agent-loop](https://code.claude.com/docs/en/agent-sdk/agent-loop#budget-headroom))
7. Auto mode's classifier fallback can't prompt in `-p`, so blocked actions are skipped silently while the run continues. ([permission-modes](https://code.claude.com/docs/en/permission-modes#repeated-block-thresholds))
8. SIGTERM loses the in-flight turn; send SIGINT first. ([headless](https://code.claude.com/docs/en/headless#stop-a-run-with-sigterm))
9. A `-p` run may linger up to 10 minutes waiting for background work. ([headless](https://code.claude.com/docs/en/headless#background-tasks-at-exit))
10. Transcripts and `~/.claude/jobs` files are not stable APIs. ([sessions](https://code.claude.com/docs/en/sessions#where-transcripts-are-stored), [agent-view](https://code.claude.com/docs/en/agent-view#read-session-state-from-a-script))
11. Subscription auth for third-party SDK products is restricted (see §12).

## Unverified / open

- Whether `--plugin-dir` skills are surfaced to Claude in `--bare` (no system reminders) or only invocable via explicit `/name`. Not documented; needs an experiment.
- The CLI stdin control protocol has no public spec (§3).
- Licensing/ToS for a personal or open-source orchestrator driving a user's subscription-authenticated CLI (§12).
- No per-page doc dates; the facts are pinned to CLI 2.1.303 / TS SDK 0.3.303 as of 2026-10-19.
