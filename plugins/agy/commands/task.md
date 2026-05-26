---
description: Delegate a task to Antigravity (agy --print) and return its output verbatim
argument-hint: "[--continue] <task to send to agy>"
allowed-tools: Agent, AskUserQuestion
---

Invoke the `agy:agy-task` subagent via the `Agent` tool (`subagent_type: "agy:agy-task"`), forwarding the raw user request as the prompt.

Raw user request:
$ARGUMENTS

Operating rules:
- The subagent is a thin forwarder. It runs `agy --print` once and returns stdout verbatim.
- If the raw request includes the literal flag `--continue`, leave it in — the subagent will pass it to agy so the most recent conversation is resumed.
- Return the subagent's stdout verbatim to the user. Do not paraphrase, summarize, or wrap with commentary before or after.
- If `$ARGUMENTS` is empty, use `AskUserQuestion` once to ask what agy should work on, then forward.
- If the subagent reports that agy is missing or unauthenticated, stop and tell the user to run `/agy:setup`.
