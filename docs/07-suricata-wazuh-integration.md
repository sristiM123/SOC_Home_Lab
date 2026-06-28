# 07 — Suricata + Wazuh Integration: Full Pipeline Working

> **Status:** Done
> **Tools:** Suricata 8.x, Wazuh Manager, ET Open ruleset

---

## Goal

Integrate Suricata's network IDS alerts into Wazuh so that network detections appear in the SIEM dashboard alongside host-log and firewall alerts — giving a single, unified view of host and network telemetry.

## Background: why integrate

Suricata on its own writes alerts to local files (`fast.log` and `eve.json`). That is useful, but an analyst should not have to read raw IDS log files separately from everything else. The value of a SIEM is one place to see all telemetry. By feeding Suricata's structured output into Wazuh, network IDS alerts join host logs and firewall events in the same dashboard — which is how a real SOC operates.

Suricata writes alerts in two formats:
- `fast.log` — human-readable, one line per alert.
- `eve.json` — structured JSON, one event per line, machine-readable.

The integration uses `eve.json` because Wazuh can parse JSON natively and extract the individual fields (signature, source/destination IP, ports, severity).

## What I did

### 1. Confirmed Suricata was producing structured output

Before wiring anything up, I verified that `eve.json` existed and contained data:

```bash
sudo tail -3 /var/log/suricata/eve.json
```

This returned JSON event lines, confirming Suricata was logging in the format Wazuh needs.

### 2. Told the Wazuh manager to read eve.json

Added a `localfile` block to the Wazuh manager configuration, pointing at Suricata's JSON output and declaring the format as JSON:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Added just before the closing `</ossec_config>` tag:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

- `log_format json` tells Wazuh the file is JSON (Suricata's `eve.json` format).
- `location` is the path to the file Wazuh should read.

### 3. Validated the config before restarting

Same test-before-restart discipline used throughout the lab — a broken config stops the manager from starting:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
```

A clean result (no errors) means it is safe to restart.

### 4. Restarted the manager

```bash
sudo systemctl restart wazuh-manager
```

### 5. Generated a test alert

Triggered a known Suricata test signature with a controlled request — the standard "is the IDS working?" check:

```bash
curl http://testmynids.org/uid/index.html
```

Confirmed Suricata caught it locally first:

```bash
sudo tail -3 /var/log/suricata/fast.log
```

The test signature appeared, which means it was also written to `eve.json` — the file Wazuh now reads.

### 6. Confirmed the alerts reached Wazuh

In the Wazuh dashboard → **Threat Hunting** → Ubuntu agent → **Last 15 minutes**, searching for `suricata` returned the alerts. Spikes in the timeline lined up with the times the test was run. Wazuh's built-in rules decoded the Suricata JSON and tagged the events as IDS alerts.

As a manager-side check, the alerts were also confirmed in the alert log:

```bash
sudo grep -i suricata /var/ossec/logs/alerts/alerts.log | tail -5
```

## The full pipeline

With this integration complete, the end-to-end flow is:

```
Network traffic
   → Suricata inspects the packets on the monitored interface
   → matches a rule (signature)
   → writes the alert to eve.json
   → Wazuh manager reads eve.json
   → Wazuh decodes the JSON and generates an alert
   → the alert appears in the dashboard alongside host and firewall events
```

This is unified host + network visibility in a single SIEM — the model a real SOC runs on.

## What I learned

- A SIEM's value is consolidation: network IDS alerts should live in the same place as host and firewall telemetry, not in separate log files.
- Wazuh ingests Suricata via a simple `localfile` block with `log_format json` — no custom decoder needed, because Wazuh ships with built-in Suricata rules.
- `eve.json` (structured) is the integration point, not `fast.log` (human-readable) — structured data is what a SIEM can parse into fields.
- The same operational discipline applies everywhere: validate config before restarting, and verify end-to-end with a controlled test before trusting the pipeline.
- Confirming an alert at both the dashboard *and* the manager log is good practice — it distinguishes an ingestion problem from a display/timing issue.

## SOC relevance

Combining network IDS with host telemetry in one SIEM is exactly how modern SOCs achieve full visibility. A network alert (e.g. an exploit attempt or C2 traffic) and a host alert (e.g. a suspicious process) often relate to the same incident — seeing them together is what enables correlation and faster triage. Being able to stand up an IDS, integrate it into a SIEM, and verify the pipeline end to end is directly relevant L1/L2 work.

## Next step

Generate real attack traffic (e.g. a network scan from the attacker VM) and observe Suricata detecting it as reconnaissance in Wazuh — moving from a controlled test signature to a realistic detection scenario. *(To be documented.)*
