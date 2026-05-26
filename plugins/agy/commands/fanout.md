---
description: Dispatch N independent tasks to Antigravity in parallel and aggregate the results
argument-hint: "<task1> -- <task2> -- <task3>   (or one task per line, each prefixed with '- ')"
allowed-tools: Agent, AskUserQuestion
---

The user wants to fan out a batch of independent tasks to Antigravity in parallel.

Raw input:
$ARGUMENTS

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
3. **Single-task fallback** — if neither delimiter is present and the input is non-empty, treat it as a single task. Confirm with `AskUserQuestion` that the user really wants a single-task fan-out (degenerate), offering options "Run as single agy task" and "Cancel — I'll re-enter with multiple tasks". Don't proceed unless they confirm single-task.
4. **Empty input** — use `AskUserQuestion` once to ask for the tasks. Provide options "Enter tasks now" and "Cancel".

After parsing, you have an ordered list of N task strings. If N > 6, ask the user once whether to proceed with that many parallel calls (cost / rate-limit consideration).

## Dispatch

Construct each task prompt with these properties:

- Self-contained — the agy subagent sees zero session context, so inline anything it needs.
- Bake any desired output shape (length, format, JSON, etc.) into the task string itself.
- Do not add meta-commentary — the subagent is a forwarder, not a reasoner.

In a **single response message**, issue **N parallel** `Agent` tool calls:

```
Agent(subagent_type: "agy:agy-task", description: "<short label for task i>", prompt: "<task i>")
```

…one call per parsed task. All in the same message so they run concurrently.

## Aggregate

After all subagents return:

1. Display one short header summarizing what was dispatched (e.g. "Dispatched 3 tasks to agy in parallel.").
2. For each task in order, show:
   ```
   ### Task <i>: <short label>
   <verbatim agy stdout>
   ```
3. If any task returned an `[agy exited <code>]` line or "command not found", flag it at the top of the aggregate so the user sees failures immediately.
4. Do NOT paraphrase agy's output. Forward it verbatim per task.

## Don't

- Don't dispatch sequentially — the whole point is parallel.
- Don't merge or summarize agy outputs into one prose block. Keep them separated and verbatim.
- Don't add Claude's own analysis on top unless the user explicitly asked for synthesis.
- Don't include `--continue` for fan-out tasks — each task is a fresh thread.
