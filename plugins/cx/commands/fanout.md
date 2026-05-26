---
description: "Dispatch N independent tasks to the codex CLI in parallel via cx:cx-task and aggregate the results. Requires the codex CLI on $PATH."
argument-hint: "<task1> -- <task2> -- <task3>   (or one task per line, each prefixed with '- ')"
allowed-tools: Agent, AskUserQuestion, Bash(command:*)
---

The user wants to fan out a batch of independent tasks to the codex CLI in parallel.

Raw input:
$ARGUMENTS

## Preflight — codex CLI installed?

Run once:

```bash
command -v codex
```

If empty, stop and tell the user:

> The `cx` plugin requires OpenAI's Codex CLI to be installed and authenticated.
> Install from https://developers.openai.com/codex/cli, then run `codex login` once before retrying.

Do not attempt the fan-out until `codex` is on `$PATH`.

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

After parsing, you have an ordered list of N task strings. If N > 4, ask the user once whether to proceed — codex CLI tasks are heavier than agy and can chew rate-limit quickly.

## Dispatch

Construct each task prompt with these properties:

- Self-contained — codex sees zero session context, so inline anything it needs (file paths, error messages, decisions).
- Bake the desired output shape (length, format, JSON, etc.) into the task string itself.
- Each task is a fresh codex session by default (every `codex exec` invocation starts a new session). No flags needed for independence.
- Foreground only — do not pass any flag that would background the call.

In a **single response message**, issue **N parallel** `Agent` tool calls:

```
Agent(subagent_type: "cx:cx-task", description: "<short label for task i>", prompt: "<task i>")
```

…one call per parsed task. All in the same message so they run concurrently.

## Aggregate

After all subagents return:

1. One short header: e.g. "Dispatched 3 tasks to codex in parallel."
2. Per task in order:
   ```
   ### Task <i>: <short label>
   <verbatim codex stdout>
   ```
3. If any subagent returned an `[codex exited <code>]` line or "codex CLI is not installed", flag that task at the top of the aggregate.
4. Do NOT paraphrase codex output. Forward it verbatim per task.

## Don't

- Don't skip the preflight on a fresh session.
- Don't dispatch sequentially.
- Don't merge codex outputs into one prose block.
- Don't add your own analysis unless the user explicitly asked for synthesis.
- Don't pass extra flags (`-m`, `-s`, `resume`, etc.) to codex unless the user explicitly requests them — the subagent treats unexplained flags as an error.
