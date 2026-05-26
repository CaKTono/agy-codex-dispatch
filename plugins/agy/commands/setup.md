---
description: Verify Antigravity is installed and that the network path to Google's API is healthy. Probes mihomo-specific checks only if mihomo is detected on this host.
allowed-tools: Bash(command:*), Bash(which:*), Bash(agy:*), Bash(ip:*), Bash(dig:*), Bash(curl:*), Bash(timeout:*), Bash(systemctl:*), Bash(test:*)
---

Run these checks and report a punch list (✓ / ✗ per item) at the end. Do not stop on a failure — collect every result first, then summarize.

## Always run

1. **Binary present.** Run `command -v agy`. If empty, report ✗ "agy not installed — install from https://antigravity.google/product/antigravity-cli" and skip the rest.
2. **agy version.** Run `agy changelog 2>&1 | head -3 || agy --help 2>&1 | head -1`. Note the version line.
3. **Google OAuth reachability.** Run:
   ```bash
   timeout 12 curl -sS -o /dev/null -w "%{http_code} %{time_total}s remote=%{remote_ip}\n" \
     -X POST https://oauth2.googleapis.com/token
   ```
   Expected: HTTP 400 or 401 in under 6s — proves the TLS path reaches Google. SSL "unexpected EOF" or HTTP `000` means whatever proxy/VPN you're using is being killed by Google's anti-abuse on this exit IP; switch nodes.
4. **Eligibility endpoint reachability.** Run:
   ```bash
   timeout 12 curl -sS -o /dev/null -w "%{http_code} %{time_total}s\n" \
     -X POST https://daily-cloudcode-pa.googleapis.com/v1internal:loadCodeAssist \
     -H 'content-type: application/json' --data '{}'
   ```
   Expected: HTTP 401 (no auth) in under 6s.

## Conditional — only if mihomo is detected

First detect:
```bash
mihomo_present=false
command -v mihomo >/dev/null 2>&1 && mihomo_present=true
systemctl list-unit-files 2>/dev/null | grep -q '^mihomo' && mihomo_present=true
test -d /etc/mihomo && mihomo_present=true
echo "mihomo_present=$mihomo_present"
```

If `mihomo_present=false`, skip the rest of this section and just say "(mihomo not detected — skipped TUN/fake-IP checks)".

If `mihomo_present=true`, run:

5. **Mihomo TUN routing.** Run:
   ```bash
   ip rule show | grep -q 'lookup 2022' && echo OK || echo BROKEN
   ```
   If `BROKEN`, tell the user to run `sudo systemctl restart mihomo` (mihomo's auto-route can partially install after a network event).
6. **DNS fake-IP active.** Run `dig +short oauth2.googleapis.com`. Pass if the answer is in `198.18.0.0/16` (mihomo's fake-IP range), warn otherwise.
7. **If OAuth probe in step 3 returned SSL EOF / `000`,** point the user at mihomo's dashboard (default `http://127.0.0.1:9090/ui`) to switch the active proxy node — Google's anti-abuse frequently kills shared exit IPs against `oauth2.googleapis.com`.

## Output rules

- One line per check: `✓ <name>` or `✗ <name> — <one-sentence reason>`.
- End with a single suggestion line if anything failed; otherwise say "All clear — try `/agy:task` to delegate work."
