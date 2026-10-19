# Mechanical steps in Matt's flows: candidates for tool CLI commands

Research for wayfinder ticket #7 (map #2). Sources are the installed skills under `.agents/skills/*/` (every `SKILL.md` and sibling `.md`) and `docs/agents/*.md`, as of commit `25bbf58`. Every claim cites `path:line`. The gh behaviour noted below was checked live against gh 2.97.0 on this repo.

## TL;DR

1. About 60 steps across the skills are deterministic tracker, git, or file-layout work written as prose. They cluster in **wayfinder**, **to-tickets**, **implement-spec**, and **triage**. The other skills are almost entirely judgement.
2. The biggest payoff is a **wayfinder/ticket core**: `frontier`, `claim`, `resolve`, and `chart`/`publish` with two-pass edge wiring. Every session runs these, and the prose currently hands them over as a multi-step `gh api` recipe with a database-id trap (`docs/agents/issue-tracker.md:43-44`).
3. Part of that prose is **already stale**. gh 2.97 has native `--blocked-by`, `--parent`, and `--add-sub-issue` flags, and `gh issue view --json blockedBy`. One GraphQL call returns the whole frontier, which removes the N+1 `gh api` loop the doc implies.
4. Three tracker docs (GitHub, GitLab, local markdown) spell out the same seven operations. A tool CLI with tracker adapters would let every skill name one command (`haai ticket claim 7`) no matter which tracker the repo uses.
5. Some steps that look mechanical are not: picking a ticket's type, fog-vs-ticket, out-of-scope rulings, matching `.out-of-scope/` entries "by concept, not keyword", and the triage state decision. The CLI should run the *write* that follows a decision and leave the decision itself to the agent.

### Top 10 by payoff (frequency x error-proneness x tokens saved)

| # | Command | Replaces | Why it pays |
|---|---------|----------|-------------|
| 1 | `haai map frontier <map> [--first] [--json]` | wayfinder `SKILL.md:69,123`, `issue-tracker.md:44` | Every work session runs it. Today it means listing sub-issues, then one blocker lookup and one assignee lookup per child. A single GraphQL query (verified below) replaces all of that. |
| 2 | `haai ticket resolve <n> --answer-file F --gist "…"` | wayfinder `SKILL.md:125`, `issue-tracker.md:46` | Every session. Three writes (comment, close, map-body edit). The map-body edit is a read-modify-write on a body that parallel sessions also edit (`SKILL.md:128`). |
| 3 | `haai map chart --plan plan.yaml` / `haai tickets publish --plan plan.yaml` | wayfinder `SKILL.md:113-114`, to-tickets `SKILL.md:63` | Two-pass create-then-wire, real ids rather than placeholder `#<n>` (`SKILL.md:114` warns a placeholder auto-links to an unrelated issue), plus the blocker's db-id lookup (`issue-tracker.md:43`). |
| 4 | `haai ticket claim <n>` / `haai ticket next --map <m> --claim` | wayfinder `SKILL.md:67,123`, `issue-tracker.md:45` | First write of every session. It needs a verify-after-write, which prose never does (see Open questions: same-user parallel sessions). |
| 5 | `haai integ start/merge/finish` + `haai wt new/sync/clean` | implement-spec `SKILL.md:25-40` | Branch, worktree, base check, reset, merge-tip, draft-PR timing, and cleanup are all deterministic and spread over 9 steps and several subagents. |
| 6 | `haai triage set <n> --category C --state S` | triage `SKILL.md:41,45,78-86` | It can enforce exactly one category and one state, check legal transitions, and add the AI disclaimer (`SKILL.md:13-17`) to every comment by itself. |
| 7 | `haai review prep <fixed-point>` | code-review `SKILL.md:21-32,36` | Validates the ref, checks the diff is non-empty, lists commits, pulls issue refs out of commit messages, finds spec and standards candidates. Saves a failure inside two parallel subagents. |
| 8 | `haai triage queue` | triage `SKILL.md:58-66`, `issue-tracker.md:24` | Three buckets, including "needs-info with reporter activity since the last triage notes", which is a timestamp comparison, plus the long external-PR `jq` filter. |
| 9 | `haai branch throwaway <kind>/<name> --push --link <issue>` | wayfinder `SKILL.md:115`, prototype `SKILL.md:26`, ask-matt `SKILL.md:87` | Branch from main, commit, push, never PR, and leave a context pointer on the issue. Easy to get wrong (branch off HEAD, open a PR). |
| 10 | `haai seq next <dir>` / `haai adr new <slug>` | `ADR-FORMAT.md:3,27`, teach `LEARNING-RECORD-FORMAT.md:3,27`, teach `SKILL.md:49`, to-tickets `SKILL.md:62` | "Scan for the highest number and increment" shows up four times. It is trivial but racy and easy to get off by one with parallel agents. |

---

## Method

I read every `.md` under `.agents/skills/` (27 skills plus their sibling references) and all three `docs/agents/*.md`. `docs/agents/*.md` are byte-identical to the seed templates in `.agents/skills/setup-matt-pocock-skills/` (checked with `diff`), so I cite the `docs/agents/` copies. For each prose step I checked whether, given the same inputs, there is exactly one correct output. If there is, the step is mechanical. If it needs taste, reading meaning, or a human, it is a judgement call.

The proposed command shapes use a placeholder binary name, `haai`. The namespace is a sketch for later tickets, not a decision.

Live checks (gh 2.97.0, this repo):

- `gh issue create --parent N --blocked-by A,B --blocking C` and `gh issue edit N --add-blocked-by A --add-sub-issue X --parent P` exist. The `gh api … dependencies/blocked_by -F issue_id=<db-id>` recipe in `docs/agents/issue-tracker.md:43` is no longer needed.
- `gh issue view N --json blockedBy,subIssues` works. `gh issue list` has **no** parent filter, so "scoped to the map's sub-issues" (`issue-tracker.md:44`) can't be one `gh issue list`.
- One GraphQL query returns the frontier inputs for the whole map:

```bash
gh api graphql -f query='{repository(owner:"DaanMaertens",name:"haai-stack"){issue(number:2){
  subIssues(first:50){nodes{number title state
    assignees(first:5){nodes{login}} labels(first:10){nodes{name}}
    blockedBy(first:20){nodes{number state}}}}}}}' \
  --jq '.data.repository.issue.subIssues.nodes[]
        | select(.state=="OPEN" and (.assignees.nodes|length)==0
                 and ([.blockedBy.nodes[]|select(.state=="OPEN")]|length)==0)
        | {number,title,type:[.labels.nodes[].name|select(startswith("wayfinder:"))][0]}'
```

Run against map #2 it correctly showed #3-#7 as claimed, #8 blocked by #3-#7, and #9 blocked by #8.

---

## Inventory by flow

Columns: **Where** (path:line) / **What the step does** / **Commands the prose implies** / **Error-proneness in prose** / **Proposed command**.

### 1. Wayfinder (`.agents/skills/wayfinder/SKILL.md`, `docs/agents/issue-tracker.md` "Wayfinding operations")

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `wayfinder/SKILL.md:25`, `issue-tracker.md:39-46` | Look up how *this* repo's tracker expresses map, child, blocking, frontier, claim, and resolve. Fall back to local markdown. | Read `docs/agents/issue-tracker.md`, find the section | Medium. The agent must read and interpret a doc each session. The fallback default is easy to miss. | Implicit: `haai` reads tracker config once (`haai config show`) |
| `wayfinder/SKILL.md:113`, `issue-tracker.md:41` | Create the map issue with the `wayfinder:map` label and a templated body (`SKILL.md:31-53`) | `gh issue create --label wayfinder:map --title … --body "$(cat <<EOF …)"` | Low-medium. Heredoc quoting. The section headings must match exactly for later edits to find them. | `haai map create --title T --destination-file F --notes-file F` (renders the template) |
| `wayfinder/SKILL.md:114`, `issue-tracker.md:10,42` | Create each ticket as a sub-issue with a `wayfinder:<type>` label and a `## Question` body (`SKILL.md:57-63`) | `gh issue create --parent <map> --label wayfinder:<type> …` (older gh: `gh api --method POST repos/o/r/issues/<map>/sub_issues -F sub_issue_id=<child-db-id>`) | Medium. The db-id vs number vs node_id confusion (`issue-tracker.md:10,43`). Type must be one of four. | `haai ticket create --map M --type research --title T --question-file F` |
| `wayfinder/SKILL.md:114`, `issue-tracker.md:43` | Second pass: wire blocking edges with real ids, and write cross-references with real ids | Per edge: `gh api repos/o/r/issues/<blocker> --jq .id`, then `gh api --method POST repos/o/r/issues/<child>/dependencies/blocked_by -F issue_id=<db-id>` | **High.** Two calls per edge. Using `#number` or `node_id` fails silently or mis-links. A placeholder `#<n>` auto-links to an unrelated issue (`SKILL.md:114`). Stale now that gh has `--add-blocked-by`. | `haai map chart --plan plan.yaml`: local handles (`a`, `b`) in the plan, real ids resolved and edges wired in one run, `{{ref:a}}` body placeholders swapped for real `#n` |
| `wayfinder/SKILL.md:115` | For each research ticket: subagent, findings on a throwaway `research/<name>` branch, push, no PR, context pointer from the ticket | `git switch -c research/<name> origin/main`, write, commit, `git push -u origin research/<name>`, `gh issue comment <n> --body <link>` | Medium. Branching off HEAD instead of main, accidentally opening a PR, forgetting the pointer. | `haai branch throwaway research/<name> --from main --push --link <n>` |
| `wayfinder/SKILL.md:122`, `SKILL.md:29` | Load the map at low resolution: body plus open children found by query, not listed | `gh issue view <map> --json body`, then the frontier query | Low-medium. Loading every ticket body by accident wastes context. | `haai map show <map>` (body, frontier, claimed, blocked counts, no child bodies) |
| `wayfinder/SKILL.md:69,123`, `issue-tracker.md:44` | Compute the frontier: open, unblocked (every blocker closed), unassigned children. First in map order wins. | List sub-issues, then for each: blockers (`issue_dependencies_summary.blocked_by`) and assignees, filter, order | **High.** N+1 calls. "Map order" is ambiguous (sub-issue order? number?). `issue_dependencies_summary.blocked_by` counts open blockers only, an easy subtlety to get wrong. | `haai map frontier <map> [--first] [--json]` (one GraphQL call, above) |
| `wayfinder/SKILL.md:67,123`, `issue-tracker.md:45` | Claim: assign to the driving dev, as the session's **first** write | `gh issue edit <n> --add-assignee @me` | Medium. A race: two sessions see the same frontier head, and `--add-assignee` adds rather than replaces, so both "win". Prose never re-reads to verify. | `haai ticket claim <n>` (assign, re-read, fail if someone else already holds it); `haai ticket next --map M --claim` |
| `wayfinder/SKILL.md:124` | Read the `wayfinder:<type>` label to decide how to resolve ("the body never states the type") | `gh issue view <n> --json labels --jq …` | Low, but the skill feels it has to warn about it, so agents do get it wrong | `haai ticket show <n>` prints `type:` first |
| `wayfinder/SKILL.md:124` | Zoom: fetch the full body of related or closed tickets on demand | `gh issue view <n> --json title,body,comments` | Low | `haai ticket show <n> [--comments]` |
| `wayfinder/SKILL.md:125`, `issue-tracker.md:46` | Resolve: post a resolution comment, close, append `- [title](url): gist` to Decisions-so-far | `gh issue comment`, `gh issue close`, `gh issue view <map> --json body`, edit the text, `gh issue edit <map> --body-file` | **High.** The map-body edit is a read-modify-write that parallel sessions race (`SKILL.md:128`). Section insertion by hand. The line must use the title, not a bare `#n` (`SKILL.md:17`). | `haai ticket resolve <n> --answer-file F --gist "…"` (fetches the title, formats the line, re-reads the body right before writing, checks after) |
| `wayfinder/SKILL.md:101,126` | Rule a ticket out of scope: close it, add a line to Out of scope (never to Decisions-so-far) | `gh issue close <n> --reason "not planned"`, map body edit | Medium. Wrong section, same body race. | `haai ticket out-of-scope <n> --why "…"` |
| `wayfinder/SKILL.md:126`, `SKILL.md:86,93` | Graduate fog: create the new tickets (create-then-wire), then remove that patch from Not yet specified | Same as create and wire, plus a body edit | Medium | `haai map fog list` / `haai map fog remove <match>` (the ticket creation goes through `chart --append`) |
| `wayfinder/SKILL.md:126` | "Update or delete" tickets the decision invalidates | `gh issue edit`/`gh issue close --reason "not planned"` | Low (the write). Deciding *which* is judgement. | `haai ticket edit`/`haai ticket drop <n> --why` |
| `wayfinder/SKILL.md:65` | Invariant: map and tickets carry **only** `wayfinder:` labels, never triage labels | none (prose rule) | Medium. Nothing checks it. `to-tickets` (`SKILL.md:63`) adds `ready-for-agent` by default, so mixing flows can break it. | `haai lint map <map>` (labels, sections present, every child typed) |
| `wayfinder/SKILL.md:105` | Policy: at most one non-research ticket resolved per session | none | Low-medium. A behavioural rule an orchestrator can enforce. | Orchestrator concern, not a tool CLI command |
| `issue-tracker-local.md:25-30` | Local-markdown versions of all of the above: `map.md`, `issues/NN-slug.md`, `Type:`/`Status:`/`Blocked by:` lines | File scan, line edits | Medium. Free-text `Blocked by:` parsing, numbering. | The same `haai` commands with a local adapter |
| `issue-tracker-gitlab.md:43-46` | GitLab: `/blocked_by` quick action as a note, links API for the frontier, close without a comment | `glab issue note <n> --message "/blocked_by #<b>"`, `glab api projects/:id/issues/<iid>/links` | High. Different verbs, close can't take a comment (`:12`), blocking is premium-only with a fallback. | The same `haai` commands with a GitLab adapter |

### 2. to-spec / to-tickets

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `to-tickets/SKILL.md:17` | Fetch a referenced spec or issue: full body **and comments** | `gh issue view <n> --json title,body,comments` | Low. Comments are easy to forget. | `haai ticket show <n> --comments` |
| `to-spec/SKILL.md:19`, `issue-tracker.md:7,31` | Publish the spec as an issue with the `ready-for-agent` label | `gh issue create --label ready-for-agent --body-file …` | Low | `haai spec publish --file spec.md` |
| `to-spec/SKILL.md:21-75` | Spec body template (section headings) | none | Low. Headings drift. | Template rendered or checked by `haai spec publish` (lint: required sections present) |
| `to-tickets/SKILL.md:63` | Publish tickets **in dependency order** so edges can name real ids. Native blocking. Sub-issue of the source issue. `ready-for-agent` label. | Topological sort by hand, then N × (`gh issue create --parent S --label ready-for-agent --blocked-by …`) | **High.** Manual topo-sort, the id bookkeeping across N creates, the db-id trap on older gh. Same shape as wayfinder chart. | `haai tickets publish --plan tickets.yaml [--parent S] [--label ready-for-agent]` (topo-sorts, creates, wires, prints a handle-to-number table) |
| `to-tickets/SKILL.md:62,69-82`, `issue-tracker-local.md:7-11` | Local: `.scratch/<slug>/issues/NN-<slug>.md`, numbered from 01 in dependency order, one file per ticket, template | `mkdir -p`, write N files | Medium. Numbering, "never a single combined file", slugging. | `haai tickets publish` with the local adapter |
| `to-tickets/SKILL.md:65`, ask-matt `SKILL.md:26` | Work the frontier: any ticket whose blockers are done. Local trackers do it "blockers-first by hand". | Same as the wayfinder frontier, but with no map: the frontier of the spec's children | High (same as wayfinder) | `haai tickets frontier --parent S` (shares its implementation with `map frontier`) |
| `to-tickets/SKILL.md:67` | Do **not** close or modify the parent issue | none | Low. A guard rail. | `tickets publish` never writes the parent's body or state |
| `to-tickets/SKILL.md:101` | Omit the "Blocked by" section when native edges are set | none | Low | Handled by the template inside `tickets publish` |

### 3. implement / implement-spec

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `implement/SKILL.md:9` | Fetch the ticket and state its title before starting | `gh issue view <n> --json title,body,comments` | Low | `haai ticket show <n>` |
| `implement/SKILL.md:13` | Run typecheck regularly, single test files regularly, the full suite at the end | Repo-specific (`npm run typecheck`, `vitest path`, …) found by reading `package.json` and CI | Medium. Rediscovered every session. retro `SKILL.md:18` names the same lookup. | `haai checks list` (discovers commands from `package.json`, Makefile, CI); `haai checks run typecheck\|test [file]\|all` |
| `implement/SKILL.md:17` | Commit to the current branch | `git add`, `git commit` | Low | none needed |
| `implement-spec/SKILL.md:21`, `SKILL.md:13` | Read the spec plus its tickets as a task graph | Spec view, list children, blockers each | High (same as the frontier) | `haai tickets graph --parent S [--json\|--mermaid]` |
| `implement-spec/SKILL.md:23` | The exploration subagent saves notes **outside the repo**, reachable by all later subagents | `mktemp -d`, then pass the path along | Low-medium. A path every subagent must agree on. | `haai scratch dir <spec>` (a stable per-spec path, e.g. `$TMPDIR/haai/<repo>/<spec>`) |
| `implement-spec/SKILL.md:25` | Create the integration branch. If the tracker closes via PRs, open a draft PR **after the first merge** (it can't open with zero commits ahead), marked as closing the spec and tickets. | `git switch -c <integ> origin/main`, `git push -u`, later `gh pr create --draft --body "Closes #S, Closes #t1…"` | Medium-high. The timing rule, and building the "Closes" list by hand from the tickets. | `haai integ start <spec>`; `haai integ pr` (opens the draft PR once ahead, closing every child) |
| `implement-spec/SKILL.md:27-28` | Per ticket: its own worktree and branch, check it is based on the integration branch, reset onto it if not | `git worktree add -b <t> ../wt-<t> <integ>`, `git merge-base --is-ancestor <integ> HEAD` or `git reset --hard <integ>` | **High.** `reset --hard` is destructive, the base check is subtle, and path and branch naming drift. Each subagent repeats it. | `haai wt new <ticket> --base <integ>` (idempotent; verifies the base) |
| `implement-spec/SKILL.md:30` | Before reporting done, merge the integration tip into the ticket branch | `git fetch`, `git merge <integ>` | Medium | `haai wt sync` |
| `implement-spec/SKILL.md:32` | The merger subagent merges the ticket branch into the integration branch | `git switch <integ>`, `git merge --no-ff <t>`, push | Medium (mechanical when clean; conflicts are judgement) | `haai integ merge <ticket>` (fails clearly on conflict and hands it to an agent) |
| `implement-spec/SKILL.md:34` | Recompute the frontier after each merge, start new implementers | Frontier query, but "done" here means *merged into integ*, not tracker-closed | **High.** Tickets close only at the end (`:38`), so the tracker's blocked-by edges stay open the whole run. The frontier has to come from local merge state. | `haai integ frontier <spec>` (tracker graph minus merged ticket branches) |
| `implement-spec/SKILL.md:38` | Mark the draft PR ready, **or** close each ticket the way the tracker closes work, then report the branch | `gh pr ready`, or N × `gh issue close <t> --comment …` | Medium | `haai integ finish <spec>` |
| `implement-spec/SKILL.md:40` | Clean up all implementer worktrees | `git worktree list`, `git worktree remove …`, `git branch -d …` | Medium. Leftover worktrees and branches. | `haai wt clean [--spec S]` |
| *(absent)* | Neither implement skill **claims** a ticket, though parallel implementers could pick up the same one | none | Gap | `haai ticket claim` applies here too |

### 4. Triage (`triage/SKILL.md`, `AGENT-BRIEF.md`, `OUT-OF-SCOPE.md`, `docs/agents/triage-labels.md`)

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `triage/SKILL.md:13-17` | Every comment or issue posted during triage **must** start with the AI disclaimer | Prefix every `gh issue comment` body by hand | Medium. Easy to forget on the third comment. | Built into `haai triage comment` / `triage set` |
| `triage/SKILL.md:43`, `triage-labels.md:5-13` | Map canonical role names to this repo's label strings | Read the table, substitute | Low-medium | `haai` reads the mapping; commands take canonical role names |
| `triage/SKILL.md:11`, `issue-tracker.md:27` | Resolve a bare `#42` to an issue or PR | `gh pr view 42` falling back to `gh issue view 42` | Low | `haai ref resolve 42` (or built into `triage show`) |
| `triage/SKILL.md:58-64`, `issue-tracker.md:9,24` | Needs-attention buckets, oldest first: unlabeled; `needs-triage`; `needs-info` with reporter activity since the last triage notes; plus external PRs when enabled | `gh issue list --json …` with the long `--jq` at `issue-tracker.md:9`; the `gh api --paginate 'repos/{owner}/{repo}/pulls?state=open' --jq '.[] \| select(.author_association \| IN("OWNER","MEMBER","COLLABORATOR") \| not) …'` at `issue-tracker.md:24`; per-issue comment timeline compared against the last "## Triage Notes" comment | **High.** Two long `jq` incantations. "Reporter activity since last notes" means joining comment authors and timestamps against the issue author. | `haai triage queue [--json]` |
| `triage/SKILL.md:70,112` | Read the full issue or PR (body, comments, labels, author, dates, diff for a PR). Parse prior triage notes. | `gh issue view --json …`, `gh pr view --comments`, `gh pr diff` | Low-medium | `haai triage show <n>` (prior notes pulled out into "established" and "outstanding") |
| `triage/SKILL.md:70`, `OUT-OF-SCOPE.md:72` | Read every `.out-of-scope/*.md` | `cat .out-of-scope/*.md` | Low | `haai oos list` (concept name, decision line, prior requests). The **matching** stays with the agent (see judgement calls). |
| `triage/SKILL.md:41,45` | Invariants: exactly one category role and one state role; legal transitions (unlabeled→needs-triage→{needs-info, ready-for-agent, ready-for-human, wontfix}; needs-info→needs-triage); flag unusual ones | Manual label arithmetic with `gh issue edit --add-label/--remove-label` | **High.** Swapping states means removing the old label and adding the new one, conflicts are possible, and nothing validates. | `haai triage set <n> [--category bug\|enhancement] --state S [--force]` (removes the old state, checks the transition, non-zero exit on an illegal one) |
| `triage/SKILL.md:78-81,94-106`, `AGENT-BRIEF.md:41-68` | Post an agent brief / ready-for-human brief / needs-info notes from fixed templates | `gh issue comment --body-file` | Low (the write). The content is judgement. | `haai triage set … --brief-file F` (checks the template's required headings: Category, Summary, Acceptance criteria, Out of scope) |
| `triage/SKILL.md:82-85`, `OUT-OF-SCOPE.md:90-97` | Wontfix flow: check for an existing `.out-of-scope/<concept>.md`; append to Prior requests or create a file; comment linking it; close with the `wontfix` label | File append/create, `gh issue comment`, `gh issue edit --add-label wontfix`, `gh issue close` | Medium. Five steps in order, kebab-case naming (`OUT-OF-SCOPE.md:58`), and "already implemented must NOT write here" (`OUT-OF-SCOPE.md:88`). | `haai triage wontfix <n> --reason rejected --concept dark-mode [--why-file F]` / `--reason implemented --where "…"` |
| `AGENT-BRIEF.md:15-16` | Briefs must not reference file paths or line numbers | none | Low-medium | A lint inside `triage set --brief-file` (warn on path-like or `:NN` tokens) |

### 5. Code review (`code-review/SKILL.md`)

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `code-review/SKILL.md:21` | Capture the diff and commit list once | `git diff <fp>...HEAD`, `git log <fp>..HEAD --oneline` | Medium. Three dots vs two dots is a real trap, and the skill uses both on purpose. | `haai review prep <fp>` prints both commands and their outputs or paths |
| `code-review/SKILL.md:23` | Fail early if the ref doesn't resolve or the diff is empty | `git rev-parse <fp>`, `git diff --quiet` | Low, but the skill exists to stop it failing inside two subagents | Same command; non-zero exit |
| `code-review/SKILL.md:29` | Spec source 1: issue refs in commit messages (`#123`, `Closes #45`, GitLab `!67`), fetched | `git log --format=%B <fp>..HEAD \| grep -oE …`, then `gh issue view` each | Medium. Regex coverage. | `review prep` lists `spec_candidates` |
| `code-review/SKILL.md:31` | Spec source 3: a spec file under `docs/`, `specs/`, `.scratch/` matching the branch or feature | `git branch --show-current`, `find docs specs .scratch -name …` | Medium (the lookup is mechanical; "matching" is fuzzy) | `review prep` lists candidates; the agent picks |
| `code-review/SKILL.md:36` | Standards sources: `CODING_STANDARDS.md` and `CONTRIBUTING.md` must be included | `ls`/`find` | Low (the named files); the rest is judgement | `review prep` lists the named files that exist |

### 6. Retro (`retro/SKILL.md`)

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `retro/SKILL.md:13` | Find the session's primary source: "searching through session logs on this machine", default the current one | `ls -t ~/.claude/projects/<slug>/*.jsonl`, … (host-specific) | **High.** The agent has to know where the host stores logs, and the slug encoding. | `haai session log [--current\|--id X] [--path-only]` (host adapter) |
| `retro/SKILL.md:18` | Read the repo's own check command and CI; note when no guardrail (pre-commit or CI) exists | `cat package.json`, `ls .github/workflows`, `ls .husky .git/hooks` | Medium | `haai checks list --guardrails` (reports "no guardrail" as a finding) |
| `retro/SKILL.md:19` | A mechanical violation gets a deterministic check (lint rule, pre-commit hook, CI job) | n/a | n/a. This is the retro-side twin of this whole ticket: retro already says "mechanical → tool, judgement → prose". | none; a design note for haai |

### 7. Others

| Where | Step | Implied commands | Error-proneness | Proposed |
|---|---|---|---|---|
| `domain.md:7-11`, `GLOSSARY-FORMAT.md:54-60`, wait-what `SKILL.md:7` | Find the glossary or glossaries: root `GLOSSARY.md`, or `GLOSSARY-MAP.md` → per-context; ADRs in `docs/adr/` and `src/<ctx>/docs/adr/` | `ls`, read the map, follow links | Low-medium. Repeated by tdd `:10`, diagnosing-bugs `:10`, pr `:37`, improve-arch `:25`. | `haai domain paths [--context X]` |
| `ADR-FORMAT.md:3,27` | ADR file `docs/adr/NNNN-slug.md`: scan for the highest number, add one, create the dir lazily | `ls docs/adr \| sort \| tail -1` | Medium. Off-by-one, parallel collisions. | `haai adr new <slug>` (prints the path) |
| teach `LEARNING-RECORD-FORMAT.md:3,27`, teach `SKILL.md:49` | Same numbering for learning records and lessons | same | Medium | `haai seq next <dir> --slug S [--ext html]` |
| teach `SKILL.md:55`, improve-arch `SKILL.md:39` | Open a generated HTML file for the user (OS-dependent: `xdg-open`/`open`/`start`). improve-arch also builds `<tmpdir>/architecture-review-<timestamp>.html`. | as listed | Low-medium (OS branching) | `haai open <file>`; `haai tmpfile architecture-review --ext html` |
| handoff `SKILL.md:8` | Save to the OS temp dir (`$TMPDIR`, `/tmp`, `%TEMP%`), not the workspace | `${TMPDIR:-/tmp}` | Low | `haai tmpfile handoff --ext md` |
| to-questionnaire `SKILL.md:16` | Write `to-questionnaire-<slug>.md` in the cwd | none | Low | none (not worth a command) |
| prototype `SKILL.md:26`, ask-matt `SKILL.md:87` | Capture the prototype on a throwaway `prototype/<name>` branch out of main; context pointer on the implementation issue | `git switch -c prototype/<name> origin/main`, commit, push, `gh issue comment` | Medium (same as the research branch) | `haai branch throwaway prototype/<name> --push --link <n>` |
| improve-arch `SKILL.md:23` | Hot spots from the commit history | `git log --oneline --name-only -n N \| sort \| uniq -c \| sort -rn` | Low-medium. Agents improvise this. | `haai hotspots [--since 90d]` |
| diagnosing-bugs `SKILL.md:110,136` | Tag debug logs with a unique prefix `[DEBUG-xxxx]`, grep it out at cleanup | `openssl rand -hex 2`; `grep -rn "\[DEBUG-a4f2\]"` | Low | `haai debug-tag new` / `haai debug-tag sweep <tag>` |
| diagnosing-bugs `SKILL.md:125` | Diff against a pristine copy to prove a forced mutation landed | `git diff`/`git stash` | Low | none |
| wizard `SKILL.md:41-43` | `bash -n`, `shellcheck`, `chmod +x`; check every `set_secret` name matches a `secrets.*` reference in `.github/workflows/*` | as listed, plus a grep cross-reference | Medium. The secret-name cross-check is pure string matching done "statically" by eye. | `haai wizard lint <script>` |
| setup `SKILL.md:23-30` | Exploration probes: git remote host, CLAUDE.md / AGENTS.md, glossary, ADR dirs, `docs/agents/`, `.scratch/`, triage installed, monorepo signals | `git remote -v`, `ls`, `jq .workspaces package.json` | Low-medium | `haai setup detect --json` |
| setup `SKILL.md:76-82` | Pick CLAUDE.md if it exists, else AGENTS.md, else ask; update `## Agent skills` in place | file tests, section replace | Medium (in-place section replace) | part of `haai setup write` |
| setup `SKILL.md:104` | Create each configured triage label the tracker lacks | `gh label list`, `gh label create …` | Low | `haai labels ensure` (also covers the `wayfinder:*` labels, which nothing currently creates) |
| git-guardrails `SKILL.md:26-35,81,89-95` | Copy the hook script, chmod, merge into `hooks.PreToolUse` without clobbering, verify exit code 2 | `cp`, `chmod`, `jq` merge, pipe a test payload | Medium (JSON merge) | Already half-scripted (`scripts/block-dangerous-git.sh`). A `haai hooks install git-guardrails` is possible but low frequency. |
| research `SKILL.md:12` | Save where the repo keeps such notes; otherwise somewhere sensible | `ls docs` | Low | covered by `branch throwaway` + a `docs/research/` convention |

Scanned and found **no** mechanical tracker or git steps worth a command: `grilling`, `grill-me`, `grill-with-docs`, `codebase-design` (+`DEEPENING.md`, `DESIGN-IT-TWICE.md`), `tdd` (+`tests.md`, `mocking.md`), `pr`, `writing-for-agents` (+`SKILL-MECHANICS.md`), `prototype/LOGIC.md`, `prototype/UI.md`, `improve-codebase-architecture/HTML-REPORT.md`, `teach` format files other than numbering, `ask-matt/PHASE-BOUNDARIES.md`, `domain-modeling/SKILL.md` beyond file placement. `wizard/template.sh` and `diagnosing-bugs/scripts/hitl-loop.template.sh` are already scripts; they are prior art for the tool CLI approach.

---

## Looks mechanical, but is a judgement call

| Where | Step | Why it stays with the agent or human |
|---|---|---|
| `wayfinder/SKILL.md:65,77-81` | Choosing a ticket's type (research / prototype / grilling / task) | Depends on what the question needs (outside facts vs a human decision vs an artifact). Only *reading* the label back is mechanical. |
| `wayfinder/SKILL.md:88-93` | Fog vs ticket | The test, "can you state the question precisely now", is semantic. |
| `wayfinder/SKILL.md:97-101,126` | Ruling a ticket out of scope; deciding which tickets a decision invalidates | Scope is set by the destination's meaning. The CLI only does the close-and-annotate write. |
| `wayfinder/SKILL.md:44`, `SKILL.md:17` | The one-line gist in Decisions-so-far | Summarising is judgement. Formatting the link from the title is mechanical. |
| `wayfinder/SKILL.md:123` | "First frontier ticket in order" vs a user-named ticket | Mechanical as written. But "map order" is undefined for GitHub (sub-issue order? number?), so a human should *decide the rule once*, after which it is mechanical. |
| `to-tickets/SKILL.md:38-40` | Blocking edges, slice boundaries, expand-contract batching for wide refactors | Design decisions, approved by the user at `:42-56`. Publishing them is mechanical. |
| `implement-spec/SKILL.md:32` | Merging with conflicts | A clean merge is mechanical. Conflict resolution needs code understanding. `integ merge` should stop and hand over. |
| `triage/SKILL.md:70` | Redundancy check ("by domain concept, not just the request's wording") | Semantic search. |
| `OUT-OF-SCOPE.md:75` | Matching against `.out-of-scope/` | "By concept similarity, not keyword: 'night theme' matches `dark-mode.md`." Explicitly not string matching. |
| `triage/SKILL.md:72,74,45` | Which state to move to; verifying a bug repro; "flag transitions that look unusual" | The maintainer decides. The CLI can only reject *illegal* transitions, not unusual ones. |
| `OUT-OF-SCOPE.md:56-58` | Naming the concept file | Picking the concept is judgement. Kebab-casing it is not. |
| `code-review/SKILL.md:19,31,36` | The fixed point if the user gave none; which spec file "matches the feature"; "every file that documents how code should be written" | The skill says to ask. Fuzzy matches. Discovering standards beyond the two named files means reading. |
| `code-review/SKILL.md:38-56,78` | Smell detection; keeping the two axes un-reranked | Labelled as "always a judgement call" (`:41`). |
| `retro/SKILL.md:19` | Classifying a violation as mechanical vs judgement | This is the classification this ticket performs. It needs a human or agent. |
| `ADR-FORMAT.md:29-37`, `domain-modeling/SKILL.md:66-74` | Whether to offer an ADR | Three-part semantic test. Numbering is mechanical. |
| `ask-matt/PHASE-BOUNDARIES.md:53-55` | Continue / clear / handoff / subagent / compact | Stated outright: "These are judgement calls." An orchestrator CLI can *offer* the tree but not decide it. |
| `implement/SKILL.md:11`, `tdd/SKILL.md:22` | Which seams to test | Pre-agreed with the user. |
| `wizard/SKILL.md:29` | Each stage's UI journey | Third-party UIs change. The skill forbids inventing steps. |

---

## Cross-cutting findings

1. **The tracker doc is the real API, and it is a prose API.** Skills say "the tracker doc's operation" (`to-tickets/SKILL.md:63`, `wayfinder/SKILL.md:25`), and three near-identical docs implement it (`issue-tracker.md:39-46`, `issue-tracker-gitlab.md:37-46`, `issue-tracker-local.md:21-30`). A tool CLI with adapters (`github`, `gitlab`, `local`) is the natural shape: the skills keep their prose about *when*, and the CLI owns *how*.
2. **Stale incantations.** `issue-tracker.md:10,43` documents `gh api … sub_issues` and `… dependencies/blocked_by` with database ids. gh 2.97 has `--parent`, `--add-sub-issue`, `--blocked-by`, and `--add-blocked-by` by number or URL. A CLI moves this staleness into one versioned place.
3. **Concurrency is the hidden cost.** Wayfinder expects parallel sessions (`SKILL.md:128`), but claim (`--add-assignee` adds) and the Decisions-so-far append (body read-modify-write) both race. GitHub issue bodies have no compare-and-swap. A CLI can narrow the window (re-read right before writing, verify after) but can't close it. That points to an open question.
4. **Two kinds of "done".** Wayfinder and to-tickets read "unblocked" from the tracker (blocker closed). implement-spec closes tickets only at the very end (`SKILL.md:38`), so its frontier (`:34`) has to come from merged branches. One `frontier` implementation needs a pluggable "is done" predicate.
5. **Label invariants are never checked.** Wayfinder-only labels (`wayfinder/SKILL.md:65`), one-category-one-state (`triage/SKILL.md:41`), and the triage disclaimer (`triage/SKILL.md:13`) are rules no code enforces. `haai lint` and label-aware write commands would turn them into guarantees.
6. **Nothing creates the `wayfinder:*` labels.** Setup creates only the triage labels (`setup-matt-pocock-skills/SKILL.md:104`). `gh issue create --label wayfinder:map` fails on a fresh repo. `haai labels ensure` should cover both sets.

## Open questions for later wayfinder tickets

- **Claim identity for same-user parallel sessions.** A claim is "assign to the dev driving the map" (`wayfinder/SKILL.md:67`). When one dev runs several sessions in parallel (`:128`), every session is `@me`, so assignee can't tell sessions apart. Should a claim carry a session id (a comment or label `claimed-by:<session>`), or should the orchestrator CLI own the lease?
- **Where Decisions-so-far lives.** Keep it as an edited map-body section (race-prone), or derive it from closed children plus their resolution comments at read time (`haai map show` renders it)? The second removes the write entirely but changes the map's "index" contract (`wayfinder/SKILL.md:23`).
- **"Map order".** Sub-issue order (UI-draggable), issue number, or an explicit rank? This needs a decision before `frontier --first` can be deterministic.
- **Tool CLI vs orchestrator CLI split.** The one-ticket-per-session rule (`wayfinder/SKILL.md:105`), the research fan-out (`:115`), and implement-spec's implementer and merger loop (`implement-spec/SKILL.md:27-34`) are *orchestration*, not tool calls. Which of these move to the orchestrator CLI, and what does the skill prose say once they have?
- **Session-log access is host-specific** (`retro/SKILL.md:13`). Is `haai session log` a tool CLI concern or part of the agent-host adapter that the orchestrator ticket (#6) is researching?
- **How skills should call the CLI.** Rewrite Matt's skill prose to name `haai` commands (a fork), or keep upstream skills untouched and point `docs/agents/issue-tracker.md` at `haai` (the tracker doc is already the indirection layer)? The second keeps upstream merges painless.
