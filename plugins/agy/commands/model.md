---
description: Report agy's currently active model and list the known picker options. Switching requires agy's interactive model picker — there is no CLI flag for it.
allowed-tools: Bash(agy:*), Bash(command:*), Bash(timeout:*)
---

The `agy` CLI does not expose a `--model` flag, an `AGY_MODEL` env var, or a local config file for model selection — the model picker is interactive-only and the choice is stored server-side per Google account.

This command can **read** the active model, not switch it.

## Step 1 — Sanity check agy

Run `command -v agy`. If empty, report ✗ "agy not installed — run /agy:setup" and stop.

## Step 2 — Probe the active model

Run exactly this command (the prompt is crafted to discourage chattiness and to ask for the picker label, not the LLM's self-name). Do **not** add `--print-timeout` or `--` — they have triggered argv-parsing oddities in past agy versions and caused the prompt to be misinterpreted as a flag-docs query.

```bash
timeout 60 agy --print "Output exactly one line and nothing else: the model display label as it appears in your model picker (e.g. Gemini 3.5 Flash (High) or Claude Opus 4.6 (Thinking)). No quotes, no preamble, no explanation."
```

Trim the result. If it looks like a model label (matches one of the known picker entries listed below, or close to one), present it as: `Active model (best-effort): <label>`.

If the response is empty, doesn't look like a label, or the command failed with auth / network / `[agy exited <code>]`, say so explicitly. Suggest:
- Auth issue → run `agy` interactively once to refresh credentials.
- Network issue → run `/agy:setup` to confirm the proxy/TUN path.

## Step 3 — Show the known picker entries

Show this list verbatim so the user knows what they can switch to in the interactive picker:

```
Known agy model picker entries (subject to availability per account):
  Gemini 3.5 Flash (Low | Medium | High)
  Gemini 3.1 Pro   (Low | High)
  Claude Sonnet 4.6 (Thinking)
  Claude Opus 4.6  (Thinking)
  GPT-OSS 120B     (Medium)
```

## Step 4 — Switching instructions

End with a single line:

> To switch the active model, start agy interactively (`agy` with no flags, or `agy -i "<initial prompt>"`) and type `/model` to open the model picker. Arrow keys to select, enter to confirm. The new choice persists across future `agy --print` calls until you change it again. There is no headless / CLI / scripted way to change the model — agy stores the choice on Google's side, not locally.

## Don't

- Don't claim the model is something agy didn't actually report.
- Don't say the model can be changed via `--model` or env var. It can't.
- Don't infer the model from session history or memory — only from this command's probe.
- Don't run any additional `agy --print` calls; the probe in Step 2 is the only allowed forward call.
