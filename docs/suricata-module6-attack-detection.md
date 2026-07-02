# Suricata Module 6 — Real Attack Detection (Lab)

Hands-on: generating attack traffic and watching Suricata detect it, both in Suricata's own logs and in the SIEM. Traffic generated from Windows to avoid running a second Linux VM (memory-constrained host).

---

## Goal

Move from a controlled test signature to realistic attack traffic, and observe detection end to end — Suricata → SIEM.

## Attack 1 — Port scan (reconnaissance)

A port scan is classic recon — probing many ports to find open services. Generated from Windows PowerShell against the monitored host:

```powershell
1..50 | ForEach-Object { Test-NetConnection <ubuntu-ip> -Port $_ -WarningAction SilentlyContinue -InformationLevel Quiet }
```

This rapidly probes 50 ports from one source — the signature of a scan. Check what fired, filtering to relevant events:

```bash
sudo grep -iE "scan|SYN" /var/log/suricata/fast.log | tail -10
sudo grep "LAB -" /var/log/suricata/fast.log | tail -10
```

The custom SYN-scan rule from Module 3 (sid 1000003), which thresholds on 10+ SYN packets from one source in 5 seconds, is designed to catch exactly this.

## Attack 2 — Known test signature (guaranteed detection)

The standard "is the IDS working?" check, using a URL that trips a known Suricata signature:

```bash
curl http://testmynids.org/uid/index.html
sudo tail -3 /var/log/suricata/fast.log
```

Confirms the full detection chain functions.

## Attack 3 — Confirm in the SIEM

Because Suricata feeds Wazuh, the same alerts appear in the SIEM. In the Wazuh dashboard → Threat Hunting → filter to the Ubuntu agent → Last 15 minutes → search `suricata`.

### Reading the SIEM view

- **Total alerts card** — count in the window.
- **Level 12+ card** — high-severity count (zero here; test traffic is low-severity).
- **Alert level evolution** — a timeline; a spike marks when the attack traffic was generated.
- **Top agents** — all Suricata alerts attributed to the Ubuntu host, where the sensor runs (correct).
- **MITRE ATT&CK panel** — techniques mapped from events; benign system activity (e.g. sudo caching) can appear here and is not evidence of an attack.

The spike in the timeline lining up with the moment the scan/test was run confirms the pipeline captured the activity.

## The analyst skill

For every alert: **source, target, what fired, severity, benign or real?** A scan from your own Windows machine is benign — you generated it. The identical alert from an unknown external IP would be an incident. Recognising that difference is the core of triage.

## Troubleshooting / realistic notes

- Whether the built-in ET SCAN rules fire depends on scan speed and which rules are enabled. If they do not trip, the custom threshold rule (1000003) from Module 3 is the reliable catch, since it was written specifically for this pattern.
- Filtering the log (`grep`) is essential — benign apt traffic and other noise otherwise bury the alerts of interest.

## What I learned

- Real attack detection ties every prior module together: rules (Module 3), reading alerts (Module 4), and tuning out noise (Module 5).
- Attack traffic can be generated safely from an existing host (Windows) without a dedicated attacker VM.
- The same detection appears in both Suricata's logs and the SIEM — unified visibility in action.
- A spike in the SIEM timeline is the visual signature analysts look for.
- Context, not the alert alone, determines whether activity is benign or an incident.
