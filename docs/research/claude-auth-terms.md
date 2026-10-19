# May an open-source orchestrator drive a user's subscription-logged-in Claude Code?

Research for [#12](https://github.com/DaanMaertens/haai-stack/issues/12). Sources read on 2026-10-19. **This is not legal advice.** It is a reading of Anthropic's published text. That text has changed before and will change again. For a binding answer, ask Anthropic: the legal page points to [contact sales](https://www.anthropic.com/contact-sales).

## TL;DR

1. An API key (Claude Console) or a cloud provider (Bedrock, Vertex/Agent Platform, Foundry) is clearly allowed for every case: a shell-out **Orchestrator CLI**, one built on the Agent SDK, personal use, and distribution.
2. Personal use: a user may drive their own subscription-logged-in `claude` from scripts. Anthropic's docs list `claude -p` and `claude setup-token` for "scripts", and the support site says `claude -p`, Agent SDK and "third-party app" usage "still draw from your subscription limits".
3. Agent SDK as a product: a distributed product built on the Agent SDK may **not** offer claude.ai login "unless previously approved". It must use an API key or a cloud provider. This one is explicit.
4. Shell-out CLI, distributed: the text points both ways. The legal page says nothing "prevent[s] an end user from signing in to the unmodified Claude Code binary with their own Claude subscription". The support site says "If you're building a product, application, or tool for others, use API key authentication". This is the open question.
5. Hard lines in every case: do not modify the binary or strip its login methods, do not collect, store or intermediate claude.ai credentials or tokens, do not spoof Claude Code's identity, and do not pay for or resell usage on users' behalf.

## Terms used

Terms follow [GLOSSARY.md](../../GLOSSARY.md). The **Agent host** is Claude Code (the `claude` binary). The **Orchestrator CLI** is the haai-stack program that drives headless Agent host sessions from outside. The **Harness** is that orchestrator plus the tool CLIs. Two shapes are compared:

- **(a) Shell-out:** the Orchestrator CLI runs the user's installed `claude` binary (`claude -p ...`), and the binary authenticates with whatever the user has set up.
- **(b) Agent SDK:** the Orchestrator CLI imports `@anthropic-ai/claude-agent-sdk` or `claude-agent-sdk`. The SDK ships with a Claude Code binary and runs it ([Quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart): "Both the TypeScript and Python SDKs bundle a native Claude Code binary").

## Primary sources and exact passages

### 1. Claude Code docs: Legal and compliance

<https://code.claude.com/docs/en/legal-and-compliance>

Which terms apply:

> Your use of Claude Code is subject to: Commercial Terms of Service - for Team, Enterprise, and Claude API users; Consumer Terms of Service - for Free, Pro, and Max users

"Authentication and credential use" (the central passage):

> **OAuth authentication** is intended exclusively for purchasers of Claude Free, Pro, Max, Team, and Enterprise subscription plans and is designed to support ordinary use of Claude Code and other native Anthropic applications.

> **Developers** building products or services that interact with Claude's capabilities, including those using the Agent SDK, should use API key authentication through Claude Console or a supported cloud provider. Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens — sign-in to a Claude account must complete through Anthropic's own flow.

> This does not restrict how customers provision and manage their own API keys or third-party inference provider credentials [...] Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription, including where a platform hosts Claude Code as described under *Can customers offer Claude Code in their products?* above.

> Anthropic reserves the right to take measures to enforce these restrictions and may do so without prior notice.

"Can customers offer Claude Code in their products?":

> Unless we've mutually agreed otherwise, preinstalling or running Claude Code in your products or services (e.g. in hosted sandboxes or other agent infrastructure) requires agreeing to our Commercial Terms of Service and complying with the conditions below:
>
> - **The Claude Code binary must not be modified.** Claude Code must be installed and run as published by Anthropic, and customers may not remove, disable, or restrict any authentication method built into it (including methods that permit signing in with a Claude account or the user's own API key).
> - **Customers may not pay for, resell, or intermediate Claude usage on their end users' behalf.** Each end user must authenticate with their own Anthropic API key, Claude subscription plan credentials, or 3P inference provider credential (Amazon Bedrock, Google Cloud's Agent Platform, Microsoft Foundry). That usage is billed directly to the end user under their own agreement with Anthropic or, for third-party inference providers, with the applicable provider.

Usage policy section:

> Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK.

On naming: "You can accurately say, in plain text, that your product has Claude Code preinstalled or that it runs Claude Code. But you can't use the Claude Code or Anthropic names or logos as part of your own product [...] name".

### 2. Agent SDK docs: overview and quickstart

<https://code.claude.com/docs/en/agent-sdk/overview> (the same note appears in the [Quickstart](https://code.claude.com/docs/en/agent-sdk/quickstart)):

> Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK. Use the API key authentication methods described in the Quickstart instead.

> Use of the Claude Agent SDK is governed by Anthropic's Commercial Terms of Service, including when you use it to power products and services that you make available to your own customers and end users [...]

The overview also points to shape (a) as the non-Python/TS way to reach the same loop: "To drive the same agent loop from a language other than Python or TypeScript, run the CLI as a subprocess with the `-p` flag and `--output-format json`."

The Quickstart lists the SDK's supported auth as `ANTHROPIC_API_KEY`, or `CLAUDE_CODE_USE_BEDROCK=1`, `CLAUDE_CODE_USE_ANTHROPIC_AWS=1`, `CLAUDE_CODE_USE_VERTEX=1` or `CLAUDE_CODE_USE_FOUNDRY=1`.

### 3. Claude Code docs: Authentication

<https://code.claude.com/docs/en/authentication>

Credential precedence lists, among others, `CLAUDE_CODE_OAUTH_TOKEN` (item 5) and "Subscription OAuth credentials from `/login`. This is the default for Claude Pro, Max, Team, and Enterprise users." (item 7). On long-lived tokens:

> For CI pipelines, scripts, or other environments where interactive browser login isn't available, generate a one-year OAuth token with `claude setup-token` [...] This token authenticates with your Claude subscription and requires a Pro, Max, Team, or Enterprise plan.

The page also notes that the same credential sources reach the SDK: "`apiKeyHelper`, `ANTHROPIC_API_KEY`, and `ANTHROPIC_AUTH_TOKEN` apply to the CLI and the surfaces that wrap it, including the VS Code extension, the Agent SDK, and GitHub Actions." It also mentions "Terminal, VS Code extension, and Agent SDK logins" when describing org restrictions.

### 4. Claude Code docs: Headless and GitHub Actions

<https://code.claude.com/docs/en/headless>: the page is titled "Run Claude Code programmatically" and says "This page covers using the Agent SDK via the CLI (`claude -p`)". So Anthropic's own docs treat `claude -p` as one face of the Agent SDK. It also notes bare mode "doesn't use your subscription login" and recommends `--bare` "for scripted and SDK calls".

<https://code.claude.com/docs/en/github-actions>: Anthropic's own Action accepts either `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` ("an OAuth token that authenticates with your Claude subscription"). It adds: "For a secret shared across repositories, authenticate with an API key [...] rather than an OAuth token, since an OAuth token is tied to the subscription of the person who ran `claude setup-token`."

### 5. Consumer Terms (Free, Pro, Max)

<https://www.anthropic.com/legal/consumer-terms> (effective October 8, 2025; the copy served to this EU-located reader, Anthropic Ireland).

Section 3, "Use of our Services", prohibited uses:

> Except when you are accessing our Services via an Anthropic API Key or where we otherwise explicitly permit it, to access the Services through automated or non-human means, whether through a bot, script, or otherwise.

> To develop any products or services that compete with our Services, including to develop or train any artificial intelligence or machine learning algorithms or models or resell the Services.

Section 2, "Account creation and access":

> You may not share your Account login information, Anthropic API key, or Account credentials with anyone else or make your Account available to anyone else.

### 6. Commercial Terms (API keys, Team, Enterprise, Agent SDK)

<https://www.anthropic.com/legal/commercial-terms> (effective June 17, 2025). They "govern Customer's use of Anthropic API keys". A.1 allows use "to power products and services Customer makes available to its own customers and end users". D.4 bars using the Services "to build a competing product or service [...] or resell the Services except as expressly approved by Anthropic".

### 7. Usage Policy

<https://www.anthropic.com/legal/aup> (effective November 12, 2026). It says nothing about auth. It does say: "Users are responsible for ensuring that agents they build or deploy, including actions those agents take through tools, browsers, or connected systems, comply with this Usage Policy."

### 8. Anthropic support articles (official, but help-center text rather than terms)

[Log in to your Claude account](https://support.claude.com/en/articles/13189465-log-in-to-your-claude-account), section "Authenticating to subscription plans":

> Subscription plans can only be used by subscribers, and the usage included in these plans is designed to support ordinary use of native Anthropic applications, including the Claude web, desktop, and mobile applications and Claude Code.

> The preferred way to access Anthropic services using third-party software, tools, or services ("third-party tools"), including open-source projects, is through API key authentication through Claude Console or a supported cloud provider. Anthropic may at its discretion allow paid subscribers who have enabled usage credits to use certain third-party tools to access Anthropic services included in paid subscription plans, but reserves the right to draw use of such third-party tools from usage credits rather than subscription limits. [...] Use of third-party tools that misrepresent their identity to Anthropic's servers, attempt to route third-party traffic against subscription limits, or otherwise violate applicable terms or policies is prohibited and such use may be enforced against.

Section "Developers":

> If you're building a product, application, or tool for others, use API key authentication through Claude Console or a supported cloud provider. Applications that misrepresent their identity to Anthropic's servers, attempt to route third-party traffic against subscription limits, or otherwise violate applicable terms or policies are prohibited and may be enforced against.

[Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan):

> Update October 7, 2026: Claude Max and Team plans now include monthly API credits, which cover the Claude Agent SDK, `claude -p`, the Claude API, and Claude Managed Agents. You can still use the Claude Agent SDK, `claude -p`, and third-party apps with your subscription limits.

> Update June 15, 2026: We've paused the previously-announced changes to Claude Agent SDK usage. For now, nothing has changed: Claude Agent SDK, `claude -p`, and third-party app usage still draw from your subscription limits.

## What the text clearly says

- **API key and cloud providers are allowed everywhere.** Every source names them as the default or recommended path for developers, SDK products and third-party tools, and the Commercial Terms allow powering products for your own end users.
- **A product built on the Agent SDK may not offer claude.ai login** without prior approval from Anthropic (SDK overview and quickstart note; legal page "Developers" bullet).
- **No one may collect, store or intermediate claude.ai credentials or session tokens.** Sign-in must finish in Anthropic's own flow (legal page).
- **Do not route requests through Free, Pro or Max credentials on behalf of your users**, and do not pay for, resell or intermediate their usage (legal page).
- **If you run Claude Code inside your product, the binary stays unmodified.** None of its auth methods may be removed or restricted, each end user authenticates with their own credential, and the Commercial Terms apply (legal page).
- **An end user signing in to the unmodified Claude Code binary with their own subscription is expressly not prevented**, even where a platform hosts Claude Code (legal page).
- **Personal scripted use of your own subscription is supported.** `claude setup-token` is documented "for CI pipelines, scripts"; Anthropic's own GitHub Action takes a subscription token; the support site says `claude -p`, Agent SDK and third-party app usage "still draw from your subscription limits".
- **No spoofing.** Third-party tools that "misrepresent their identity to Anthropic's servers" are prohibited (support article).
- **Do not share credentials.** The Consumer Terms forbid sharing login information or making your account "available to anyone else". The Action docs warn that a setup-token is tied to the subscription of the person who minted it.

## What is ambiguous

1. **Is a distributed shell-out Orchestrator CLI a "tool for others" that must use API keys?** Arguments both ways:
   - *For allowed:* the orchestrator never offers login, never touches credentials and never intermediates usage. The user signs in to the unmodified binary through Anthropic's flow, which the legal page says is not prevented.
   - *Against:* the support article's "Developers" section says a "tool for others" should "use API key authentication". The "third-party tools" paragraph also calls API keys "preferred" for "open-source projects" and keeps the right to bill such use to usage credits.

   "Preferred" and "should" are weaker words than the SDK note's "does not allow", but no source addresses this exact shape.
2. **Does the Consumer Terms ban on automated access cover it?** The ban on access "through automated or non-human means, whether through a bot, script" exempts API keys and cases "where we otherwise explicitly permit it". Documented `claude -p` and `setup-token` use "for scripts" reads like that explicit permission for the user's own scripts. No source says whether that permission extends to a third-party orchestrator running unattended loops.
3. **Does the shell-out shape avoid the SDK restriction at all?** Anthropic's headless page describes `claude -p` as "using the Agent SDK via the CLI". A shell-out orchestrator may therefore count as an "agent built on the Claude Agent SDK" for the purposes of the overview note. The text does not settle this.
4. **What counts as "ordinary, individual usage"?** Pro and Max limits "assume ordinary, individual usage of Claude Code and the Agent SDK". Many parallel loops or worktrees driven by an orchestrator may fall outside that assumption. The text sets an expectation about limits, not a prohibition, and gives no threshold.
5. **Does a subscription token in the user's environment count as intermediating?** If a user exports `CLAUDE_CODE_OAUTH_TOKEN` and the orchestrator passes its environment to `claude`, is the orchestrator intermediating a session token? Plain inheritance looks like the documented script use. Any orchestrator code that reads, stores, prompts for or forwards the token is much closer to "collect, store, or intermediate".
6. **Does "personal" use of a public open-source tool count as distribution?** Running your own copy is personal use. Publishing the tool so others run it with their own subscriptions is where the "tool for others" sentence may apply.
7. **The wording has changed and could change again.** Secondary reports (not verified here, because the Wayback Machine was unreachable) say an early-2026 version of the legal page read: "Using OAuth tokens obtained through Claude Free, Pro, or Max accounts in any other product, tool, or service — including the Agent SDK — is not permitted". The current text quoted above is narrower. The support site also records an announced Agent SDK usage change that was later "paused". Re-check before relying on any of this.

## Auth paths by case

Key: **Clear** means a primary source expressly allows it. **Ambiguous** means the text points both ways or is silent. **No** means a primary source expressly forbids it.

| Auth path | (a) Shell-out, personal use | (a) Shell-out, distributed OSS | (b) Agent SDK, personal use | (b) Agent SDK, distributed product |
| - | - | - | - | - |
| Console API key (`ANTHROPIC_API_KEY`, `apiKeyHelper`) | Clear | Clear | Clear | Clear |
| Bedrock / Vertex (Agent Platform) / Foundry | Clear | Clear | Clear | Clear |
| User's own `/login` subscription in the unmodified binary | Clear (ordinary use; `claude -p` draws on plan limits) | Ambiguous, leaning allowed if the orchestrator never touches credentials (legal page "Nor does it prevent..." versus support "tool for others → API key") | Clear for your own use (support: "You can still use the Claude Agent SDK [...] with your subscription limits") | No, unless previously approved ("does not allow third party developers to offer claude.ai login [...] including agents built on the Claude Agent SDK") |
| `claude setup-token` → `CLAUDE_CODE_OAUTH_TOKEN` set by the user for their own runs | Clear (documented "for CI pipelines, scripts") | Ambiguous if only inherited from the user's environment. **No** if the orchestrator collects, stores or forwards the token | Clear for your own use | No (same SDK note; also "collect, store, or intermediate") |
| Orchestrator reads `~/.claude/.credentials.json`, runs its own OAuth flow, or presents itself as Claude Code | No | No | No | No |

## What this means for haai-stack (input for #9, not a decision)

The design that stays inside the clear text:

- Ship the Orchestrator CLI as a shell-out to the user's installed, unmodified `claude`, and leave authentication entirely to that binary. The orchestrator must have no login command, never read credential files or keychains, never ask for or store tokens, and never spoof a user agent.
- Document an API key or cloud provider as the recommended path, especially for CI or shared use, and say so in the README.
- Mention that a user may run it against their own subscription login at their own discretion, citing the legal page sentence and the ambiguity above. Do not market it as "use your Max plan".
- Do not build on the Agent SDK *and* advertise subscription login unless Anthropic approves it.
- Naming: plain-text "runs Claude Code" is fine. No "Claude Code" in the product name or logo.

## Sources

- Claude Code, Legal and compliance: <https://code.claude.com/docs/en/legal-and-compliance>
- Claude Code, Authentication: <https://code.claude.com/docs/en/authentication>
- Claude Code, Run Claude Code programmatically (headless): <https://code.claude.com/docs/en/headless>
- Claude Code, GitHub Actions: <https://code.claude.com/docs/en/github-actions>
- Agent SDK overview: <https://code.claude.com/docs/en/agent-sdk/overview>
- Agent SDK quickstart: <https://code.claude.com/docs/en/agent-sdk/quickstart>
- Consumer Terms of Service: <https://www.anthropic.com/legal/consumer-terms>
- Commercial Terms of Service: <https://www.anthropic.com/legal/commercial-terms>
- Usage Policy: <https://www.anthropic.com/legal/aup>
- Support, Log in to your Claude account: <https://support.claude.com/en/articles/13189465-log-in-to-your-claude-account>
- Support, Use the Claude Agent SDK with your Claude plan: <https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan>
- Support, Monthly API credits for Max and Team plans: <https://support.claude.com/en/articles/17154008-monthly-api-credits-for-max-and-team-plans>
