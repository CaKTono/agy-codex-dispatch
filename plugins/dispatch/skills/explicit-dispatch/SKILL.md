---
name: explicit-dispatch
description: "Use when the user's message contains explicit per-task agent tags like [agy] task, [codex] task, [claude] task, or [gemini] task (or the colon form, agy: task). Parse the tags, dispatch each task to the named target, and run them in parallel. Do not infer or override the user's chosen target."
---

# Explicit Dispatch

## When to use

You see a user message that contains lines like:

```
[agy] summarize https://example.com/post
[codex] refactor src/auth/login.ts to split JWT verify out
[claude] rename oldName to newName in tests/foo.test.ts
[gemini] map dependencies across the whole repo and list anything that imports `auth/`
```

…or the colon form:

```
agy: summarize https://example.com/post
codex: refactor src/auth/login.ts ...
```

…or a bullet variant (`- [agy] …`). The tags are the user's **explicit routing instruction**. This skill applies whether or not `/dispatch:fanout` was invoked — the same tagging works in plain prompts.

## The rule

**The user picks the target. You don't.** Do not infer, override, or "improve" the routing. If the user wrote `[claude]` for a task that "obviously belongs on codex", dispatch to Claude anyway. Routing is the user's job; execution is yours.

## Target → subagent_type

| Tag        | subagent_type                    | Per-target tweak                  |
|------------|----------------------------------|-----------------------------------|
| `[agy]`    | `agy:agy-task`                   | none                              |
| `[codex]`  | `codex:codex-rescue`             | append ` --fresh` to the prompt   |
| `[claude]` | `general-purpose`                | none                              |
| `[gemini]` | `cc-gemini-plugin:gemini-agent`  | none                              |
| `[<other>]`| as-written                       | none; must be a registered subagent |

Tags are case-insensitive. Whitespace inside the brackets is ignored.

## Parsing

1. Walk the message line by line. A line is a task if it matches `[<target>] <task>`, `<target>: <task>`, or one of those forms with a leading `- ` bullet.
2. Ignore lines that don't match (they're prose around the task list).
3. Preserve the original order — aggregation reports in that order.
4. If a tag refers to an unknown target / unregistered subagent, stop and ask the user to fix it. Do not substitute.

## Dispatch

Send **one response message** containing one `Agent` tool call per parsed task:

```
Agent(subagent_type: <mapped>, description: "<≤5 word label>", prompt: <task text + tweak>)
```

All in the same message so they run concurrently.

If the message contains exactly one tagged task, still dispatch it via `Agent` (single-task fan-out is degenerate but consistent — the user explicitly asked for a target).

## Crafting each prompt

The subagent doesn't see this session's history. For each task:

- Inline anything the subagent needs (URLs, file excerpts, decisions).
- Specify the output shape if the user didn't.
- Don't add your own meta-instructions; the tag is the only meta-instruction.

## Aggregate

When all return:

1. Header: `Dispatched N tasks (<target counts>)`.
2. Per task in original order, verbatim subagent output under a `### Task <i> · [<target>] <label>` heading.
3. Flag empty / errored returns at the top.
4. Do not synthesize or paraphrase unless the user explicitly asked for synthesis.

## Don't

- ❌ Re-routing because you think a different target would be better.
- ❌ Stripping tags and dispatching all to Claude.
- ❌ Adding `--fresh` to anything other than `[codex]` tasks.
- ❌ Adding `--background` to any task.
- ❌ Dispatching serially.
- ❌ Inheriting `--resume` or session state into the forwarded prompts.

## Relationship to other skills

- `superpowers:dispatching-parallel-agents` — parent discipline for *when* to fan out at all.
- `agy:parallel-agy-dispatch`, `cx:parallel-codex-dispatch` — sibling skills that pick a single target. This skill supersedes them when the user has already tagged.
- `/dispatch:fanout` — the explicit slash-command form. This skill handles the same syntax in plain prompts.
