---
name: parallel-agy-dispatch
description: Use when 2+ independent tasks are suited for Antigravity (web/search/research/long-context summarization) — fan them out to agy:agy-task subagents in one message instead of running serially. Composes with superpowers:dispatching-parallel-agents.
---

# Parallel Dispatch via Antigravity

## When to use

You already know *how* to decide parallel-vs-serial — that's `superpowers:dispatching-parallel-agents`. This skill picks up after that decision, when the answer is "fan out" **and** the tasks happen to be a better fit for Antigravity than for Claude or Codex.

Good fits for Antigravity:
- Web research that benefits from Google integration.
- Long-context summarization where a different model is preferable.
- Tasks the user has explicitly asked to route through agy.
- Anything where the user said "use agy" in this session.

Bad fits (use Claude subagents instead):
- Tasks that need this repo's CLAUDE.md context.
- Tasks that need tools only Claude Code's harness has.
- Tasks that need to call back into Claude's plugin commands.

If a batch is mixed, split it: some tasks go to `agy:agy-task`, others stay with Claude or go to `codex:codex-rescue`.

## The dispatch pattern

Send **one message** containing **multiple `Agent` tool calls**, all with `subagent_type: "agy:agy-task"`. They run concurrently.

```
Agent(subagent_type: "agy:agy-task", prompt: <task 1, self-contained>)
Agent(subagent_type: "agy:agy-task", prompt: <task 2, self-contained>)
Agent(subagent_type: "agy:agy-task", prompt: <task 3, self-contained>)
```

Each prompt is the **literal task string** to forward to `agy --print`. No meta-instructions to the subagent — it is a thin forwarder, not a thinker. If you want agy to behave a certain way (style, format, length), bake those instructions into the task string itself.

## Crafting each task

Treat each prompt as a standalone request to an external model that sees zero of this session's context:

1. **Self-contained.** Inline every fact agy needs. URLs, file excerpts, names, dates. Do not say "the file we discussed" — paste the relevant lines.
2. **Specific output shape.** "Return a 3-bullet summary." "Return a JSON list with fields x, y, z." "Return the URL and 1 sentence why it matches." Without this, agy responses won't compose.
3. **Focused scope.** One question or one transform per task. If two questions share state, they aren't independent — merge them or use one task.
4. **No back-references.** agy can't ask follow-ups mid-task. If it needs a decision, give it the answer in the prompt.

## After dispatch

When all N subagents return:

1. Read each verbatim output.
2. If any failed (`[agy exited <code>]` line or "command not found"), surface that — do not silently retry. Tell the user what happened.
3. Aggregate into one response: a short header, then per-task results in the order they were dispatched.
4. If results conflict or duplicate, point that out — don't paper over it.

## Don't do these

- ❌ Dispatching agy subagents one-by-one across multiple messages (defeats the parallelism).
- ❌ Asking the subagent to "think about", "plan", or "decide" — it's a forwarder.
- ❌ Forwarding the entire chat history. Agy doesn't need it; condense to the task.
- ❌ Using agy for tasks that require editing files in this repo — agy runs in its own context, edits won't land here. Use Claude subagents for that.
- ❌ Mixing dispatch targets in a single fan-out batch unintentionally. If the user said "use agy", all branches go to `agy:agy-task`.

## Quick mental check before firing

- ≥2 tasks? ✓
- Independent (no shared state)? ✓
- Each prompt self-contained? ✓
- Specific output shape per task? ✓
- All clearly belong on agy (not Claude / Codex)? ✓

If all five, fire one message with N `Agent` calls. Otherwise stop and rethink.

## Relationship to other skills

- `superpowers:dispatching-parallel-agents` — the parent discipline. Read it first if you're unsure whether to fan out at all.
- `agy:agy-task` (subagent) — the actual forwarder this skill dispatches to.
- `/agy:fanout` (command) — user-facing way to *ask* for fan-out explicitly when you'd rather not infer the intent.
