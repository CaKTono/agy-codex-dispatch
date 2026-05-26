# Claude Code plugins by CaKTono

Three small Claude Code plugins that work together to make **multi-model dispatch** ergonomic from inside a Claude Code session.

| Plugin | What it gives you | Depends on |
|---|---|---|
| **agy** | `/agy:setup`, `/agy:task`, `/agy:fanout`, `/agy:status`, `/agy:model` — wraps the [Antigravity](https://antigravity.google) (`agy`) CLI. | `agy` binary on `$PATH` |
| **cx** | `/cx:fanout` — fans out N tasks to OpenAI's `codex:codex-rescue` in parallel. | OpenAI's [codex plugin](https://github.com/openai/codex-plugin-cc) |
| **dispatch** | `/dispatch:fanout` + auto-loading skill — explicit per-task routing via `[agy]` / `[codex]` / `[claude]` / `[gemini]` tags. | nothing required; recognized tags need their respective plugins installed |

You can install **any subset**. Each plugin is self-contained and only references siblings as optional companions.

---

## Install — everything in one block

Copy-paste this once:

```bash
claude plugin marketplace add CaKTono/agy-codex-dispatch && \
  claude plugin install agy@caktono-plugins && \
  claude plugin install cx@caktono-plugins && \
  claude plugin install dispatch@caktono-plugins
```

Then restart Claude Code. All `/agy:*`, `/cx:*`, `/dispatch:*` commands appear in the `/` menu.

## Install — only what you want

Add the marketplace once, then pick whichever plugins you actually need:

```bash
claude plugin marketplace add CaKTono/agy-codex-dispatch

# Install any subset:
claude plugin install agy@caktono-plugins        # Antigravity wrapper
claude plugin install cx@caktono-plugins         # Codex parallel fan-out
claude plugin install dispatch@caktono-plugins   # explicit per-task routing
```

Restart Claude Code afterwards.

**Note:** `cx` will not function without OpenAI's `codex` plugin installed. The `dispatch` plugin works alone, but tags like `[codex]` or `[gemini]` only dispatch if those plugins are installed (see below).

## Local install from a clone

For testing or contributing:

```bash
git clone https://github.com/CaKTono/agy-codex-dispatch.git
claude plugin marketplace add ./agy-codex-dispatch
claude plugin install agy@caktono-plugins   # plus cx, dispatch as desired
```

---

## Optional companions

These plugins are designed to **compose with** — but do not require — other plugins. None are needed for the install commands above to succeed.

| Companion | What it provides | Used by |
|---|---|---|
| [`superpowers`](https://github.com/obra/superpowers-marketplace) | `superpowers:dispatching-parallel-agents` — the parent "when to fan out at all" discipline. | All three plugins compose with it; none require it. Skills here name-drop it as related reading. |
| [`codex` (OpenAI)](https://github.com/openai/codex-plugin-cc) | `codex:codex-rescue` subagent. | **Required by `cx`.** Also used by `dispatch` when you tag a task `[codex]`. |
| [`cc-gemini-plugin`](https://github.com/thepushkarp/cc-gemini-plugin) | `cc-gemini-plugin:gemini-agent` subagent. | Only used by `dispatch` when you tag a task `[gemini]`. |

---

## Commands reference

All commands are namespaced per plugin: `/agy:<name>`, `/cx:<name>`, `/dispatch:<name>`. They become available after `claude plugin install …` followed by a Claude Code restart.

### agy plugin

#### `/agy:setup`

Runs health checks for the `agy` CLI and its network path to Google. Reports a ✓/✗ punch list with suggested fixes on failure. Optionally probes mihomo-specific TUN routing if mihomo is detected on the host.

- **Arguments:** none
- **What it checks:** binary present → version → Google OAuth reachability → eligibility endpoint → active model self-report. Mihomo TUN routing + fake-IP DNS are checked only if mihomo is detected.
- **Output shape:** one line per check (`✓` / `✗` + name + one-line reason), then a single suggestion line if anything failed, or `All clear — try /agy:task to delegate work.` if everything passed.

#### `/agy:task [--continue] <prompt>`

Forwards a single prompt to `agy --print` and returns its response verbatim. Use `--continue` to resume the most recent agy thread.

- **Arguments:** optional `--continue`, then the prompt text.
- **Returns:** agy's stdout verbatim, no preamble.

```
/agy:task what is the gravitational acceleration on Mars?
/agy:task --continue keep going with the explanation about orbital mechanics
```

#### `/agy:fanout <tasks>`

Dispatches N independent tasks to agy in parallel. Three accepted input forms:

- **Multi-line bullet form** — one task per `- ` line. Indented (≥2 spaces or a tab) continuation lines under a bullet are joined into that task with newlines.
- **Single-line `--` separator form** — `task one -- task two -- task three`.
- **Single task** — degenerate; the command will confirm before running.

Bullets are recommended for anything beyond two short tasks.

```
/agy:fanout
- summarize https://example.com/post-1
  in 3 bullets, focused on healthcare implications
- summarize https://example.com/post-2
  same format
- find recent papers about long-context RAG benchmarks
```

Each agy response comes back verbatim under a `### Task <i>` header, in input order. If any task fails, it's flagged at the top of the aggregate.

#### `/agy:status`

Prints `agy`'s install path, version, and recent activity hints. Brief — under 8 lines.

- **Arguments:** none

#### `/agy:model`

Probes agy for the currently active model via a self-report query, and lists the known picker entries (Gemini 3.5 Flash Low/Medium/High, Gemini 3.1 Pro Low/High, Claude Sonnet/Opus 4.6 Thinking, GPT-OSS 120B Medium).

- **Arguments:** none
- **Cannot switch the model from this command.** `agy` has no `--model` flag, no env var, and no local config file for model selection — the picker is interactive-only and the choice is stored server-side per Google account.
- **To switch:** start agy interactively (`agy` with no flags, or `agy -i "<initial prompt>"`) and type `/model` to open the model picker. Use arrow keys to select, enter to confirm. The new choice persists across future `agy --print` calls until you change it again.

---

### cx plugin

#### `/cx:fanout <tasks>`

Same input forms as `/agy:fanout`, but every task is routed to `codex:codex-rescue` in parallel. Automatically appends `--fresh` to every prompt so each thread is independent. Performs a one-time preflight check that the codex plugin is installed before dispatching.

**Requires:** OpenAI's codex plugin (`claude plugin marketplace add openai/codex-plugin-cc && claude plugin install codex@openai-codex`).

```
/cx:fanout
- refactor src/auth/login.ts to extract JWT verify into its own module
- write a 5-bullet root-cause analysis of why CI flakes on macOS but not linux
- fix the race condition in tests/queue.test.ts (it expects 3 events but gets 2)
```

Each codex-rescue response comes back verbatim under a `### Task <i>` header, in input order.

---

### dispatch plugin

#### `/dispatch:fanout <tagged tasks>`

Per-task explicit routing. Each task carries a target tag, and they all dispatch in parallel.

| Tag | Routes to | Required plugin |
|---|---|---|
| `[agy]` or `agy:` | `agy:agy-task` | agy plugin |
| `[codex]` or `codex:` | `codex:codex-rescue` (auto-adds `--fresh`) | OpenAI codex plugin |
| `[claude]` or `claude:` | `general-purpose` Claude subagent | none (built-in) |
| `[gemini]` or `gemini:` | `cc-gemini-plugin:gemini-agent` | cc-gemini-plugin |
| `[<exact subagent name>]` | the named subagent | that subagent must be registered |

Tags are case-insensitive. Bullet form (`- [agy] task`) and colon form (`agy: task`) are both accepted. Indented continuation lines under a tagged task belong to that task. Missing-plugin errors are surfaced clearly with the install command to fix them.

```
/dispatch:fanout
[agy]    summarize https://example.com/blog/2026-llm-trends in 3 bullets
[codex]  refactor src/auth/login.ts to split JWT verify into its own module
         preserve the existing test coverage
[claude] rename oldName to newName in tests/foo.test.ts
[gemini] map every place in this repo that imports `auth/`
```

Header in the aggregate looks like `Dispatched 4 tasks (1 agy, 1 codex, 1 claude, 1 gemini)`.

---

## Skills (auto-loaded)

These don't require user invocation — they activate automatically when Claude notices the situation matches the trigger.

| Skill | Activates when | Purpose |
|---|---|---|
| `agy:parallel-agy-dispatch` | 2+ independent tasks suit agy (web/research/long-context summarization) | Tells Claude how to fan out agy tasks. Composes with `superpowers:dispatching-parallel-agents`. |
| `cx:parallel-codex-dispatch` | 2+ independent tasks suit Codex (substantial refactors, deeper debugging, second-opinion implementation) | Tells Claude how to fan out codex-rescue calls with `--fresh` per task. |
| `dispatch:explicit-dispatch` | Your message contains tagged tasks (`[agy]`, `[codex]`, etc.) — even in plain prose, no slash command needed | Parses tags and dispatches per the user's chosen targets without overriding the routing. |

## Subagents

| Subagent | Purpose |
|---|---|
| `agy:agy-task` | Thin forwarder that runs `agy --print '<task>'` and returns its stdout verbatim. Used by `/agy:task` and `/agy:fanout`. Hardened against shell injection: single-quoted prompts only, no `eval`, no double quotes. |

---

## Source layout

Each plugin is a directory under `plugins/` with this shape:

```
plugins/<plugin>/
├── .claude-plugin/plugin.json     ← manifest: name, version, description, license
├── commands/<name>.md             ← becomes /<plugin>:<name> slash command
├── agents/<name>.md               ← defines a subagent invocable via the Agent tool
└── skills/<name>/SKILL.md         ← auto-loadable skill — Claude loads it when its
                                     description matches the current situation
```

What each directory is for:

- **`commands/*.md`** — slash commands. The filename (minus `.md`) becomes the command name (`commands/setup.md` → `/<plugin>:setup`). YAML frontmatter declares the command's description, allowed tools, and argument hint; the body is the instruction Claude follows when the command runs. The token `$ARGUMENTS` interpolates whatever the user typed after the command, and `${CLAUDE_PLUGIN_ROOT}` resolves to the plugin's install directory.
- **`agents/*.md`** — subagent definitions. Frontmatter declares the subagent's name and which tools it can use; the body is its behavior spec. Other commands invoke them via the `Agent` tool with `subagent_type: "<plugin>:<name>"`. They run in isolated context, separate from the main session.
- **`skills/<name>/SKILL.md`** — auto-loadable skills. The frontmatter's `description` field is what triggers Claude to load the skill when the situation matches (e.g., the `dispatch:explicit-dispatch` skill triggers when a user message contains `[agy]`/`[codex]`/etc. tags). The body is the guidance Claude follows.

These `.md` files are the source of truth — if this README and a `.md` ever disagree, the `.md` wins.

| Plugin | What's inside |
|---|---|
| [plugins/agy/](plugins/agy/) | 5 commands (`setup`, `task`, `fanout`, `status`, `model`), 1 agent (`agy-task`), 1 skill (`parallel-agy-dispatch`) |
| [plugins/cx/](plugins/cx/) | 1 command (`fanout`), 1 skill (`parallel-codex-dispatch`) |
| [plugins/dispatch/](plugins/dispatch/) | 1 command (`fanout`), 1 skill (`explicit-dispatch`) |

---

## How fan-out works (one-paragraph version)

All three plugins use the same shape: a slash command parses your bullet/tag input into N independent tasks, then issues **N parallel `Agent` tool calls in a single message**, then aggregates the verbatim outputs in input order. The `dispatch` plugin adds a layer on top of that: per-task target tags so you can mix Claude, agy, codex, and gemini in one batch.

Bullet lists support **indented continuation lines** — anything indented 2+ spaces under a bullet joins the previous task's text, so multi-line task descriptions work as expected.

---

## License

MIT — see [LICENSE](LICENSE).
