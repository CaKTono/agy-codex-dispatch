---
description: "Fan out N tasks in parallel where the user explicitly picks the dispatch target per task. Tags: [agy], [codex], [claude], [gemini], or [<exact subagent name>]."
argument-hint: "one task per line, each prefixed with [agy], [codex], [claude], or [gemini] (colon form 'agy: task' also accepted)"
allowed-tools: Agent, AskUserQuestion
---

The user wants explicit per-task agent routing.

Raw input:
$ARGUMENTS

## Target → subagent_type mapping

| Tag                          | Subagent type used                  | Required plugin                                      |
|------------------------------|--------------------------------------|------------------------------------------------------|
| `[agy]`                      | `agy:agy-task`                       | agy plugin (https://github.com/CaKTono/agy-codex-dispatch) |
| `[codex]`                    | `cx:cx-task`                         | cx plugin (which itself needs the codex CLI on `$PATH`) |
| `[claude]`                   | `general-purpose`                    | none (built-in)                                      |
| `[gemini]`                   | `cc-gemini-plugin:gemini-agent`      | cc-gemini-plugin (optional)                          |
| `[<exact subagent name>]`    | as-written                           | the named subagent must be registered                |

Targets are case-insensitive (`[AGY]` and `[agy]` are the same). Colon form (`agy: …`) is also accepted as a synonym for `[agy] …`.

## Parse $ARGUMENTS

Walk $ARGUMENTS line by line. Apply these rules in order:

1. A line starting with `- [<target>] <task>` or `- <target>: <task>` (with optional whitespace around `- `) **opens** a new tagged task. Strip the leading `- ` and the tag, keeping the rest as the task text. The same line without a leading `- ` (just `[<target>] <task>` or `<target>: <task>`) also opens a new task — bullets are optional but recommended.
2. Subsequent **indented** lines (start with ≥2 spaces or a tab) belong to the most recent task — append them to that task's text with a single `\n`, stripping the indent. The tag stays the one from the opening line.
3. Subsequent **non-indented, non-tagged** lines are **ambiguous**:
   - Before the first tagged line, or after the last tagged line with a blank-line separator, treat as prose around the list and ignore.
   - Interspersed between tagged lines without a blank-line separator, stop and use `AskUserQuestion` to ask whether the line is (a) a forgotten continuation that should be joined into the previous tagged task, (b) a stray note to drop, or (c) a tagged task missing its tag. Do not guess.
4. If `<target>` doesn't match the mapping table above and isn't a registered subagent name, list the offending lines and stop. Don't guess.
5. Blank lines are ignored.

If $ARGUMENTS is empty, ask the user once for tasks. Provide options "Enter tagged tasks now" and "Cancel".

After parsing, you have an ordered list of `(target, task)` pairs. If the list has > 6 entries, ask the user once to confirm fan-out at that size.

## Dispatch

Send **one response message** with **N parallel `Agent` tool calls**, one per pair:

```
Agent(
  subagent_type: <mapped subagent_type>,
  description:   "<short label, ≤5 words>",
  prompt:        <the task text, plus any per-target modifier from the table>
)
```

Per-target modifiers:
- `agy` — no modifier.
- `codex` — no modifier (`codex exec` is a fresh session by default).
- `claude` — no modifier; the task must be self-contained as the subagent does not inherit this session's context.
- `gemini` — no modifier.

All `Agent` calls go in **one** message so they run concurrently.

## Handling missing target plugins

If any `Agent` call fails with an "unknown subagent type" / "subagent not registered" error, do **not** retry and do **not** drop the failed task silently. Instead, in the aggregate response:

- Flag the failed target at the top, e.g.:
  > Task 2 [gemini] could not run: `cc-gemini-plugin` is not installed. Install with `claude plugin marketplace add thepushkarp/cc-gemini-plugin && claude plugin install cc-gemini-plugin@cc-gemini-plugin`, then restart Claude Code.
- Still present the successful tasks' outputs.

Common missing-plugin pointers:
- `[agy]` → install the agy CLI from https://antigravity.google/product/antigravity-cli, then `claude plugin marketplace add CaKTono/agy-codex-dispatch && claude plugin install agy@caktono-plugins`
- `[codex]` → install the codex CLI from https://developers.openai.com/codex/cli and run `codex login`, then `claude plugin marketplace add CaKTono/agy-codex-dispatch && claude plugin install cx@caktono-plugins`
- `[gemini]` → `claude plugin marketplace add thepushkarp/cc-gemini-plugin && claude plugin install cc-gemini-plugin@cc-gemini-plugin`

## Aggregate

After all subagents return:

1. Header line: `Dispatched N tasks (<target counts>)` — e.g. "Dispatched 3 tasks (1 agy, 1 codex, 1 claude)".
2. Per task in original input order:
   ```
   ### Task <i> · [<target>] <short label>
   <verbatim subagent output>
   ```
3. If any returned empty / errored, flag them at the top of the aggregate.
4. Do not paraphrase any subagent's output. Verbatim per task.
5. Do not add your own analysis unless the user explicitly asked for synthesis.

## Don't

- Don't infer a target if the user forgot one. Ask.
- Don't dispatch serially — the whole point is parallel.
- Don't merge outputs into a single prose block.
- Don't pass `--background` to any target; fan-out aggregates foreground stdout.
- Don't apply a per-target modifier to the wrong target.

## Example

Input:
```
- [agy] summarize https://example.com/blog/2026-llm-trends in 3 bullets
- [codex] refactor src/auth/login.ts to extract JWT verify into its own module
- [claude] rename oldName to newName in tests/foo.test.ts
```

Dispatched as 3 parallel `Agent` calls:
- `subagent_type: "agy:agy-task"`    ← task 1 prompt verbatim
- `subagent_type: "cx:cx-task"`      ← task 2 prompt verbatim
- `subagent_type: "general-purpose"` ← task 3 prompt verbatim
