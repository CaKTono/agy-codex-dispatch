# Claude Code plugins by CaKTono

Three small Claude Code plugins that work together to make **multi-model dispatch** ergonomic from inside a Claude Code session.

| Plugin | What it gives you | Depends on |
|---|---|---|
| **agy** | `/agy:setup`, `/agy:task`, `/agy:fanout`, `/agy:status` — wraps the [Antigravity](https://antigravity.google) (`agy`) CLI. | `agy` binary on `$PATH` |
| **cx** | `/cx:fanout` — fans out N tasks to the codex CLI in parallel via `cx:cx-task`. | `codex` binary on `$PATH` |
| **dispatch** | `/dispatch:fanout` — explicit per-task routing via `[agy]` / `[codex]` / `[claude]` / `[gemini]` tags. | nothing required; recognized tags need their respective plugins installed |

You can install **any subset**. Each plugin is self-contained and only references siblings as optional companions.

---

## Prerequisites

Each plugin wraps an external CLI. Install the corresponding tool **before** installing the plugin, and run the tool's `login` / interactive flow once so the OAuth tokens are cached.

| Plugin | External CLI required | Install / docs |
|---|---|---|
| `agy` | Antigravity CLI (`agy` on `$PATH`) | https://antigravity.google/product/antigravity-cli |
| `cx` | OpenAI Codex CLI (`codex` on `$PATH`) | https://developers.openai.com/codex/cli (run `codex login` once after install) |
| `dispatch` | No CLI prereq of its own. Tags only dispatch if the respective plugin is installed (see [Optional companions](#optional-companions) below). | — |

Without the listed CLI installed and authenticated, the corresponding plugin's commands will fail with a clear error pointing here.

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

**Note:** `cx` requires the `codex` binary to be installed and authenticated (`codex login`). It does **not** require OpenAI's `codex` plugin for Claude Code — `cx` wraps the codex CLI directly. The `dispatch` plugin works alone, but the `[codex]` tag routes through `cx:cx-task` (so it needs `cx` installed) and `[gemini]` tags need cc-gemini-plugin installed.

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
| [`superpowers`](https://github.com/obra/superpowers-marketplace) | `superpowers:dispatching-parallel-agents` — the parent "when to fan out at all" discipline. | Optional. Not required by any plugin here; useful related reading on parallel dispatch. |
| [`codex` (OpenAI)](https://github.com/openai/codex-plugin-cc) | `codex:codex-rescue` subagent and `/codex:*` slash commands. | Optional. **Not required** by `cx` or `dispatch` — both wrap the codex CLI directly. Install only if you want OpenAI's official `/codex:*` commands. |
| [`cc-gemini-plugin`](https://github.com/thepushkarp/cc-gemini-plugin) | `cc-gemini-plugin:gemini-agent` subagent. | Only used by `dispatch` when you tag a task `[gemini]`. |

---

## Commands reference

All commands are namespaced per plugin: `/agy:<name>`, `/cx:<name>`, `/dispatch:<name>`. They become available after `claude plugin install …` followed by a Claude Code restart.

### agy plugin

#### `/agy:setup`

Runs health checks for the `agy` CLI and its network path to Google. Reports a ✓/✗ punch list with suggested fixes on failure. Optionally probes mihomo-specific TUN routing if mihomo is detected on the host.

- **Arguments:** none
- **What it checks:** binary present → version → Google OAuth reachability → eligibility endpoint. Mihomo TUN routing + fake-IP DNS are checked only if mihomo is detected.
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

> **Model selection note.** `agy` has no `--model` CLI flag, no env var, and no local config file. To switch between Gemini / Claude / GPT-OSS models inside agy, run `agy` interactively (no flags) and type `/model` to open the model picker — arrow keys to select, enter to confirm. The choice persists across future `agy --print` calls. This plugin deliberately does **not** ship a `/agy:model` command, because the only programmatic probe available (asking the LLM to self-identify) reliably lies.

---

### cx plugin

#### `/cx:fanout <tasks>`

Same input forms as `/agy:fanout`, but every task is routed to `cx:cx-task` (a thin forwarder around `codex exec`) in parallel. Each `codex exec` invocation is a fresh session by default. Performs a one-time preflight (`command -v codex`) to confirm the CLI is on `$PATH`.

**Requires:** OpenAI Codex CLI installed and authenticated (https://developers.openai.com/codex/cli, then `codex login`). Does **not** require OpenAI's codex Claude Code plugin.

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
| `[codex]` or `codex:` | `cx:cx-task` (wraps `codex exec`) | cx plugin + codex CLI on `$PATH` |
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

## Subagents

| Subagent | Purpose |
|---|---|
| `agy:agy-task` | Thin forwarder that runs `agy --print '<task>'` and returns its stdout verbatim. Used by `/agy:task` and `/agy:fanout`. Hardened against shell injection: single-quoted prompts only, no `eval`, no double quotes. |
| `cx:cx-task` | Thin forwarder that runs `codex exec '<task>'` and returns the codex CLI's stdout verbatim. Used by `/cx:fanout` and by `/dispatch:fanout` when a task is tagged `[codex]`. Same shell-injection hardening as `agy:agy-task`. |

---

## Source layout

Each plugin is a directory under `plugins/` with this shape:

```
plugins/<plugin>/
├── .claude-plugin/plugin.json     ← manifest: name, version, description, license
├── commands/<name>.md             ← becomes /<plugin>:<name> slash command
└── agents/<name>.md               ← defines a subagent invocable via the Agent tool
```

What each directory is for:

- **`commands/*.md`** — slash commands. The filename (minus `.md`) becomes the command name (`commands/setup.md` → `/<plugin>:setup`). YAML frontmatter declares the command's description, allowed tools, and argument hint; the body is the instruction Claude follows when the command runs. The token `$ARGUMENTS` interpolates whatever the user typed after the command, and `${CLAUDE_PLUGIN_ROOT}` resolves to the plugin's install directory.
- **`agents/*.md`** — subagent definitions. Frontmatter declares the subagent's name and which tools it can use; the body is its behavior spec. Other commands invoke them via the `Agent` tool with `subagent_type: "<plugin>:<name>"`. They run in isolated context, separate from the main session.

> **Note:** these plugins intentionally ship **no auto-loadable skills** (`skills/<name>/SKILL.md`). All behavior is driven by explicit slash commands so nothing imposes opinions on Claude's default behavior. You can still add skills yourself in a fork — Claude Code will discover any `skills/<name>/SKILL.md` it finds.

These `.md` files are the source of truth — if this README and a `.md` ever disagree, the `.md` wins.

| Plugin | What's inside |
|---|---|
| [plugins/agy/](plugins/agy/) | 4 commands (`setup`, `task`, `fanout`, `status`), 1 agent (`agy-task`) |
| [plugins/cx/](plugins/cx/) | 1 command (`fanout`), 1 agent (`cx-task`) |
| [plugins/dispatch/](plugins/dispatch/) | 1 command (`fanout`) |

---

## How fan-out works (one-paragraph version)

All three plugins use the same shape: a slash command parses your bullet/tag input into N independent tasks, then issues **N parallel `Agent` tool calls in a single message**, then aggregates the verbatim outputs in input order. The `dispatch` plugin adds a layer on top of that: per-task target tags so you can mix Claude, agy, codex, and gemini in one batch.

Bullet lists support **indented continuation lines** — anything indented 2+ spaces under a bullet joins the previous task's text, so multi-line task descriptions work as expected.

---

## License

MIT — see [LICENSE](LICENSE).
