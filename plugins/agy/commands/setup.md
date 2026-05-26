---
description: Verify Antigravity is installed and that the network path to Google's API is healthy.
allowed-tools: Bash(command:*), Bash(which:*), Bash(agy:*), Bash(curl:*), Bash(timeout:*)
---

Run these checks and report a punch list (✓ / ✗ per item) at the end. Do not stop on a failure — collect every result first, then summarize.

1. **Binary present.** Run `command -v agy`. If empty, report ✗ "agy not installed — install from https://antigravity.google/product/antigravity-cli" and skip the rest.
2. **agy version.** Run `agy changelog 2>&1 | head -3 || agy --help 2>&1 | head -1`. Note the version line.
3. **Google OAuth reachability.** Run:
   ```bash
   timeout 12 curl -sS -o /dev/null -w "%{http_code} %{time_total}s remote=%{remote_ip}\n" \
     -X POST https://oauth2.googleapis.com/token
   ```
   Expected: HTTP 400 or 401 in under 6s — proves the TLS path reaches Google.

   Failure modes worth surfacing to the user:
   - **SSL "unexpected EOF" or HTTP `000`** — the TCP/TLS handshake reached *something*, but the connection was killed mid-stream. Common cause: an outbound proxy/VPN whose exit IP is being blocked or rate-limited by Google's anti-abuse for this domain. Suggest switching to a different exit / disabling the proxy for `*.googleapis.com`.
   - **Timeout / no response** — outbound networking is broken or DNS is hijacked to an unreachable address. Suggest checking `dig oauth2.googleapis.com` and the user's default route.
4. **Eligibility endpoint reachability.** Run:
   ```bash
   timeout 12 curl -sS -o /dev/null -w "%{http_code} %{time_total}s\n" \
     -X POST https://daily-cloudcode-pa.googleapis.com/v1internal:loadCodeAssist \
     -H 'content-type: application/json' --data '{}'
   ```
   Expected: HTTP 401 (no auth) in under 6s. Same failure modes as step 3 apply.

## Output rules

- One line per check: `✓ <name>` or `✗ <name> — <one-sentence reason>`.
- End with a single suggestion line if anything failed; otherwise say "All clear — try `/agy:task` to delegate work."
