---
description: Show agy install location, version, and a hint at recent activity
allowed-tools: Bash(command:*), Bash(agy:*), Bash(ls:*), Bash(stat:*)
---

Gather quick status info and print a short summary (≤ 8 lines total):

1. `command -v agy` — install path.
2. `agy changelog 2>&1 | head -1` — version.
3. Most recent agy state file mtime, if discoverable:
   ```bash
   ls -lt ~/.agy ~/.config/agy ~/.local/share/agy 2>/dev/null | head -5 || true
   ```

Then a one-line hint:
- `/agy:setup` to re-run health checks.
- `/agy:task <prompt>` to delegate a task to Antigravity.

Do not output anything else.
