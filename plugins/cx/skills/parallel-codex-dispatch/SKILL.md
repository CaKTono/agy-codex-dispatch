---
name: parallel-codex-dispatch
description: Use when 2+ independent tasks are suited for Codex (deeper debugging, second-opinion implementation, substantial refactors, rescue work) — fan them out to codex:codex-rescue subagents in one message instead of running serially. Composes with superpowers:dispatching-parallel-agents.
---

# Parallel Dispatch via Codex

## When to use

`superpowers:dispatching-parallel-agents` already tells you *when* to fan out. This skill picks up after that decision, when the answer is "fan out" **and** the tasks are a better fit for Codex than for Claude or agy.

Good fits for Codex:
- Substantial coding tasks: refactors, multi-file edits, complex bug investigations.
- Second-opinion / rescue passes when Claude got stuck.
- Tasks where a different model's perspective is wanted.
- Anywhere the user has said "use codex" in this session.

Bad fits (use Claude or agy instead):
- Short edits Claude can finish in seconds (codex-rescue's own contract says don't grab these).
- Web research / Google integration (agy is the right target).
- Tasks needing to call back into Claude's own plugin commands or harness tools.

If the batch is mixed, split it: some tasks to `codex:codex-rescue`, some to `agy:agy-task`, some kept on Claude.

## The dispatch pattern

Send **one message** with **multiple `Agent` tool calls**, all `subagent_type: "codex:codex-rescue"`:

```
Agent(subagent_type: "codex:codex-rescue", prompt: "<task 1> --fresh")
Agent(subagent_type: "codex:codex-rescue", prompt: "<task 2> --fresh")
Agent(subagent_type: "codex:codex-rescue", prompt: "<task 3> --fresh")
```

All run concurrently.

Each prompt is the **literal task string** to forward, with `--fresh` appended so codex starts a clean thread (no resume-last). codex-rescue itself parses out `--fresh`, `--background`, `--wait`, `--model`, `--effort`, `--resume` from its prompt — so those are the only routing flags that survive the forwarding hop.

## Crafting each task

Codex sees none of this session's context. For each task:

1. **Self-contained.** Inline every fact codex needs — file paths, function names, error messages, repro steps. Codex *can* read the repo, but it shouldn't have to guess what you mean by "the bug we discussed".
2. **Specific output shape.** "Return a patch", "Return a 5-line root-cause summary then the diff", "Return JSON with fields x, y". Without this, results won't compose.
3. **Focused scope.** One subsystem or one file per task. If two tasks share state, they aren't independent — merge or keep serial.
4. **No back-references.** Codex can't ask follow-ups mid-task.
5. **`--fresh` always.** Fan-out tasks must be independent threads. Never use `--resume` in a fan-out.

## Foreground vs background

Default to foreground (no `--background`). The whole point of fan-out is collecting all results in one response. `--background` returns immediately with a job ID and would defeat aggregation. Only use `--background` if the user explicitly asked for it on a long task.

## After dispatch

When all N subagents return:

1. Read each verbatim output. codex-rescue returns the codex-companion stdout exactly as-is; it returns **nothing** on failure rather than an error message — handle empty results explicitly.
2. If any returned empty, surface it at the top: "Task N failed silently — run `/codex:status` to investigate."
3. Aggregate: short header + per-task sections in dispatch order.
4. If results conflict (e.g. two patches touch the same file differently), point it out — don't paper over it.

## Don't do these

- ❌ Dispatching codex subagents serially across messages.
- ❌ Letting any task inherit `--resume` from a prior chat — fan-out must be `--fresh`.
- ❌ Mixing `--background` and foreground tasks in one fan-out batch.
- ❌ Asking the subagent to "think" or "decide" — codex-rescue is a forwarder.
- ❌ Pumping the entire session transcript into a task. Condense.
- ❌ Mixing dispatch targets unintentionally — if the user said "use codex", all branches go to `codex:codex-rescue`.

## Quick mental check before firing

- ≥2 tasks? ✓
- Independent (no shared state, no shared files being edited)? ✓
- Each prompt self-contained with specific output shape? ✓
- Each prompt ends with `--fresh`? ✓
- All belong on Codex (not Claude / agy)? ✓

If all five, fire one message with N `Agent` calls.

## Relationship to other skills

- `superpowers:dispatching-parallel-agents` — parent discipline. Read first if unsure whether to fan out at all.
- `codex:codex-rescue` (subagent) — the forwarder this skill dispatches to.
- `codex:gpt-5-4-prompting` — codex-rescue may load this to tighten prompts; you can lean on it indirectly by writing crisp task strings.
- `/cx:fanout` (command) — user-facing way to ask for codex fan-out explicitly.
- `agy:parallel-agy-dispatch` — sibling skill for the agy target. Same shape, different model.
