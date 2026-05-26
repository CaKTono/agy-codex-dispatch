---
description: "Dispatch N independent tasks to Codex in parallel via codex:codex-rescue and aggregate the results. Requires the openai-codex plugin to be installed."
argument-hint: "<task1> -- <task2> -- <task3>   (or one task per line, each prefixed with '- ')"
allowed-tools: Agent, AskUserQuestion
---

The user wants to fan out a batch of independent tasks to Codex in parallel.

Raw input:
$ARGUMENTS

## Preflight — codex plugin installed?

If a previous turn already confirmed `codex:codex-rescue` is registered, skip this check.

Otherwise, attempt one `Agent(subagent_type: "codex:codex-rescue", description: "preflight", prompt: "echo READY")` call. If it errors with "unknown subagent type" or similar (i.e. the codex plugin is not installed), stop and tell the user:

> The `cx` plugin depends on OpenAI's `codex` plugin and its `codex:codex-rescue` subagent. Install it first:
>
> ```
> claude plugin marketplace add openai/codex-plugin-cc
> claude plugin install codex@openai-codex
> ```
>
> Then restart Claude Code and re-run `/cx:fanout`.

Do not attempt the real fan-out until the preflight passes.

## Parse tasks from $ARGUMENTS

Pick the format based on what the input looks like:

1. **Multiline bullet form** — active when any non-empty line in $ARGUMENTS starts with `- ` (dash space). Apply these rules in order:
   - A line beginning with `- ` **opens** a new task. Strip the leading `- `.
   - Subsequent **indented** lines (start with ≥2 spaces or a tab) belong to the most recent task — append them to that task's text with a single `\n`, stripping the indent.
   - Subsequent **non-indented, non-bullet** lines are **ambiguous**:
     - If a line appears *before* the first bullet, or *after* the last bullet with a blank line separating it, treat it as prose framing the list and ignore.
     - If a line appears *between* bullets (no blank line separating it from neighboring bullets), stop and use `AskUserQuestion` to ask whether the line is (a) a forgotten continuation that should be joined into the previous task, (b) a stray note to drop, or (c) another task missing its dash. Do not guess.
   - Blank lines are ignored.
2. **Single-line `--` separator form** — otherwise, split the whole input on the literal `--` token (surrounded by whitespace). Trim each chunk.
3. **Single-task fallback** — if neither delimiter is present and the input is non-empty, treat it as a single task. Confirm with `AskUserQuestion` that the user really wants a single-task fan-out (degenerate), offering "Run as single codex task" and "Cancel — I'll re-enter with multiple tasks". Don't proceed unless they confirm.
4. **Empty input** — use `AskUserQuestion` once to ask for the tasks. Provide "Enter tasks now" and "Cancel".

After parsing, you have an ordered list of N task strings. If N > 4, ask the user once whether to proceed — codex-rescue tasks are heavier than agy and can chew rate-limit quickly.

## Dispatch

Invoke the `Skill` tool first to load `cx:parallel-codex-dispatch`. Follow its guidance for prompt construction:

- Make each task string self-contained — codex sees zero session context.
- Bake the desired output shape into the task string itself.
- Each task is a **fresh thread** — append the literal token `--fresh` to every forwarded prompt so codex-rescue does not resume a prior thread.
- Prefer foreground execution (do not add `--background`) so this command can aggregate outputs in one response.

Then in a **single response message**, issue **N parallel** `Agent` tool calls:

```
Agent(subagent_type: "codex:codex-rescue", description: "<short label for task i>", prompt: "<task i> --fresh")
```

…one call per parsed task. All in the same message so they run concurrently.

## Aggregate

After all subagents return:

1. One short header: e.g. "Dispatched 3 tasks to Codex in parallel."
2. Per task in order:
   ```
   ### Task <i>: <short label>
   <verbatim codex stdout>
   ```
3. If any subagent returned nothing (codex-rescue returns empty on failure per its contract), flag that task at the top of the aggregate.
4. Do NOT paraphrase codex output. Forward it verbatim per task.

## Don't

- Don't skip the preflight on a fresh session.
- Don't dispatch sequentially.
- Don't merge codex outputs into one prose block.
- Don't add your own analysis unless the user explicitly asked for synthesis.
- Don't pass `--resume` to fan-out tasks — they must be independent.
- Don't pass `--effort` or `--model` unless the user requested it; let codex-rescue use defaults.
- Don't grab review / status / result / cancel — codex-rescue's contract is `task`-only.
