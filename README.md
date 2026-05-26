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
claude plugin marketplace add CaKTono/claude-plugins && \
  claude plugin install agy@caktono-plugins && \
  claude plugin install cx@caktono-plugins && \
  claude plugin install dispatch@caktono-plugins
```

Then restart Claude Code. All `/agy:*`, `/cx:*`, `/dispatch:*` commands appear in the `/` menu.

## Install — only what you want

Add the marketplace once, then pick whichever plugins you actually need:

```bash
claude plugin marketplace add CaKTono/claude-plugins

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
git clone https://github.com/CaKTono/claude-plugins.git
claude plugin marketplace add ./claude-plugins
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

## Per-plugin docs

Reference docs and full usage live next to each plugin's source:

- **agy** → [plugins/agy/](plugins/agy/) — runs `agy --print` headless, single-task or parallel.
- **cx** → [plugins/cx/](plugins/cx/) — wraps `codex:codex-rescue` for fan-out.
- **dispatch** → [plugins/dispatch/](plugins/dispatch/) — explicit `[tag] task` routing, with continuation-indent support.

Each plugin's `.md` files (`commands/`, `agents/`, `skills/`) are the source of truth for behavior.

---

## How fan-out works (one-paragraph version)

All three plugins use the same shape: a slash command parses your bullet/tag input into N independent tasks, then issues **N parallel `Agent` tool calls in a single message**, then aggregates the verbatim outputs in input order. The `dispatch` plugin adds a layer on top of that: per-task target tags so you can mix Claude, agy, codex, and gemini in one batch.

Bullet lists support **indented continuation lines** — anything indented 2+ spaces under a bullet joins the previous task's text, so multi-line task descriptions work as expected.

---

## License

MIT — see [LICENSE](LICENSE).
