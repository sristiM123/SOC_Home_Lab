# 06 — Suricata Network IDS Integration

> **Status:** Planned
> **Tools:** Suricata, Wazuh Manager

---

## Goal

Add network-based intrusion detection to the lab with Suricata, and integrate its alerts into Wazuh so host telemetry (firewall, auth logs) and network telemetry (IDS) live in one pane of glass.

## Plan

- Install Suricata in the Ubuntu VM alongside Wazuh (lightweight enough to coexist once the indexer heap is capped).
- Configure Suricata to monitor the network interface and write alerts to `eve.json`.
- Point Wazuh at `eve.json` so IDS alerts flow into the dashboard.
- Generate test traffic from Kali (e.g. nmap, known malicious signatures) and confirm Suricata + Wazuh detect it.
- Write up the integration, any decoder/rule work, and the resulting alerts.

*This section will be filled in as I build it.*
