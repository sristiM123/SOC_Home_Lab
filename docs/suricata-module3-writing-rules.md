# Suricata Module 3 — Writing Custom Rules (Lab)

Hands-on: writing custom Suricata rules from scratch, testing them, and triggering them from other machines in the lab. All IPs shown are placeholders (`<ubuntu-ip>`).

---

## Goal

Move from reading rules to writing them — create custom detection rules, load them into Suricata, and confirm they fire on real traffic.

## Setup: a separate file for custom rules

Custom rules are kept in their own file, separate from the downloaded ET Open ruleset (good practice — your rules stay distinct and survive ruleset updates).

The ET Open rules live at `/var/lib/suricata/rules/suricata.rules`. Create a local file in the same directory:

```bash
sudo touch /var/lib/suricata/rules/local.rules
```

Tell Suricata to load it. Edit the config and add `local.rules` under `rule-files`:

```bash
sudo nano /etc/suricata/suricata.yaml
```

```yaml
rule-files:
  - suricata.rules
  - local.rules
```

Only the filename is listed, not the full path — Suricata already knows the rules directory from `default-rule-path`.

## The discipline (same as Wazuh)

For every rule change: **write → test → restart → verify.** Never restart on an untested config.

Test command:
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```
A clean result ends with "Configuration provided was successfully loaded."

Apply:
```bash
sudo systemctl restart suricata
```

Verify (filtering to only custom rules, which are all prefixed "LAB -"):
```bash
sudo grep "LAB -" /var/log/suricata/fast.log | tail -20
```

---

## Rule 1 — Detect a ping (ICMP)

```
alert icmp any any -> any any (msg:"LAB - ICMP ping detected"; sid:1000001; rev:1;)
```

Reads: alert on any ICMP (ping) traffic, labelled clearly, custom sid 1000001.

**Trigger:** `ping -c 3 <some-ip>` then check `fast.log`.

## Rule 2 — Detect RDP connection attempts

RDP (port 3389) is a top attack target, so watching for connection attempts to it is practical.

```
alert tcp any any -> $HOME_NET 3389 (msg:"LAB - RDP connection attempt to internal host"; flow:to_server; sid:1000002; rev:1;)
```

Reads: alert on TCP traffic to any internal host on port 3389, only traffic going to the server (an actual connection attempt).

**Trigger (from Windows PowerShell):**
```powershell
Test-NetConnection <ubuntu-ip> -Port 3389
```
The target does not need RDP running — the connection *attempt* is what fires the rule.

## Rule 3 — Detect a TCP SYN scan (advanced: flags + threshold)

A port scan often sends many SYN packets without completing handshakes. This rule uses TCP flags and a threshold to catch that pattern rather than alerting on single packets.

```
alert tcp any any -> $HOME_NET any (msg:"LAB - Possible TCP SYN scan"; flags:S; threshold:type both, track by_src, count 10, seconds 5; sid:1000003; rev:1;)
```

- `flags:S` — matches SYN-only packets (connection initiation).
- `threshold:type both, track by_src, count 10, seconds 5` — only alert if the same source sends 10+ SYN packets within 5 seconds. This is correlation inside a single rule — the signature of a scan, not normal traffic.

**Trigger (from Windows PowerShell) — probe many ports quickly:**
```powershell
1..50 | ForEach-Object { Test-NetConnection <ubuntu-ip> -Port $_ -WarningAction SilentlyContinue -InformationLevel Quiet }
```

## Rule 4 — Detect an HTTP request to /admin (content matching)

The everyday content-signature type.

```
alert http any any -> $HOME_NET any (msg:"LAB - HTTP request to admin panel"; flow:to_server,established; http.uri; content:"/admin"; nocase; sid:1000004; rev:1;)
```

Reads: alert on HTTP to an internal host, on an established connection to the server, where the URI contains "/admin" (case-insensitive). `http.uri` scopes the match to the URL portion of the request.

Note: this rule needs a web server on the target to fully trigger; without one it may stay quiet. It is included to demonstrate content matching.

---

## Troubleshooting encountered

**The rules directory was not where expected.** The default `/etc/suricata/rules/` did not exist on this install. Located the actual path with:
```bash
sudo find /etc/suricata /var/lib/suricata -name "*.rules" 2>/dev/null
```
Rules were under `/var/lib/suricata/rules/` (where `suricata-update` places them). Created `local.rules` there instead.

**A rule appeared not to fire, but actually did.** After triggering the ICMP rule, plain `tail -5 /var/log/suricata/fast.log` showed only benign apt package-management alerts (ET INFO), not the ping. The ping alert *was* logged — it was simply above the last 5 lines, because the log is busy with frequent apt traffic. Filtering with `grep "ICMP ping detected"` revealed it immediately.

**Lesson:** logs are noisy; filtering is how you find signal. `grep` on the CLI (or a search query in the SIEM) is the core triage move — isolate the one event you care about from the thousands around it.

## What I learned

- Custom rules live in a separate `local.rules` file, referenced in the config, using sids ≥ 1000000.
- The write → test → restart → verify discipline prevents broken-config surprises.
- `flags` and `threshold` let a single rule detect behavioural patterns like scans.
- `http.uri` and `content` scope a match to specific parts of a request.
- A quiet-looking log usually means "filter better," not "the rule failed."
