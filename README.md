# Home SOC Lab

A hands-on Security Operations Center (SOC) lab, built from the ground up on constrained hardware, documenting the journey toward SOC Analyst (L1/L2) capability. This repository records not just what was built, but *how*, including the failures, the troubleshooting, and the reasoning behind each fix, because that process is where the real learning happens.

---

## What this lab is

A functioning, end-to-end detection and response environment built and operated by hand: endpoints monitored by agents, a firewall feeding logs into a SIEM, a network intrusion detection system inspecting traffic, custom detection rules, automated response, attack simulation, and threat-intelligence enrichment, all integrated into a single analyst view.

The lab runs on a single laptop with limited RAM. That constraint is deliberate and valuable: it forces real understanding of resource trade-offs, component design, and what can run where, the kind of practical judgment that unlimited cloud resources would hide.

---

## Tools & technologies

**Currently used in this lab:**

| Tool | Role |
|------|------|
| **Wazuh** | SIEM / XDR — log collection, decoding, correlation, alerting, dashboard |
| **Suricata** | Network Intrusion Detection System (IDS) — packet inspection and signature detection |
| **Windows Defender Firewall** | Host firewall — connection blocking and drop logging |
| **VirusTotal** | Threat-intelligence enrichment — automated file-reputation lookups |
| **Wazuh FIM** | File Integrity Monitoring — detects file changes on endpoints |
| **VMware Workstation** | Virtualization host for the lab environment |
| **Ubuntu / Windows / Kali Linux** | Monitored endpoints and attacker machine |
| **MITRE ATT&CK** | Technique mapping applied to alerts |

**Skills applied across the stack:** detection engineering (custom Wazuh rules & decoders, Suricata signatures), alert tuning, active response / automation, attack simulation, log analysis, and methodical pipeline troubleshooting.

**Planned additions:** Sysmon (endpoint telemetry), CDB-list threat intelligence, Shuffle (SOAR), OpenCTI (threat-intel platform), n8n (automation), and Proxmox (dedicated virtualization for the heavier tools).

---

## The roadmap

The lab is built in stages, from network and host fundamentals through to advanced detection, automation, and threat intelligence. By the end, it demonstrates the full range of skills expected of an L1/L2 SOC analyst.

**Stage 1 — Foundations (network + host visibility)**
*Tools: Windows Defender Firewall, Suricata, Wazuh.* Host firewall configuration and log forwarding, network intrusion detection with Suricata, and a Wazuh SIEM collecting and correlating it all. Learn how traffic is blocked and logged, how an IDS inspects packets, and how host and network telemetry come together in one place.

**Stage 2 — Detection engineering**
*Tools: Wazuh (custom rules & decoders), Suricata (custom signatures).* Writing and tuning custom detection rules, building correlation logic, and cutting noise so genuine threats stand out. Learn how detections are created, how attacks appear in the data, and the discipline of tuning without hiding real threats.

**Stage 3 — Automated response**
*Tools: Wazuh active response.* Configuring the SIEM to act on high-severity alerts, not just report them. Learn the foundation of automated defense and how it is constrained so it acts on signal, not noise.

**Stage 4 — Wazuh in depth**
*Tools: Wazuh (FIM, SCA, vulnerability detection, API, reporting).* Going through the SIEM thoroughly: agents, decoders and rules, File Integrity Monitoring, Security Configuration Assessment, vulnerability detection, active response, the API, and reporting. Learn the tool corner to corner, the way an operator uses it daily.

**Stage 5 — Threat intelligence**
*Tools: VirusTotal, CDB lists, MITRE ATT&CK.* File-reputation enrichment (VirusTotal) and indicator matching against known-bad lists (CDB lists), plus deeper MITRE ATT&CK technique mapping. Learn how observed activity is enriched and matched against global threat data.

**Stage 6 — Endpoint visibility & incident response**
*Tools: Sysmon, Wazuh.* Sysmon for deep endpoint telemetry, and the incident-response process itself — triage, documentation, and escalation. Learn to investigate an alert end to end, the way an analyst does on shift.

**Stage 7 — Automation & advanced tooling (planned)**
*Tools: Shuffle (SOAR), OpenCTI, n8n, Proxmox.* SOAR playbooks (Shuffle), a threat-intelligence platform (OpenCTI), and workflow automation (n8n) — run on dedicated infrastructure (planned via Proxmox), reflecting the reality that heavier tools need proper resources. Learn how a mature SOC orchestrates response and manages threat intelligence at scale.

---

## What you'll be able to do by the end

- Operate a SIEM end to end — deploy it, enroll agents, read and triage alerts.
- Integrate host, network, and firewall telemetry into one analyst view.
- Write, tune, and correlate custom detection rules.
- Configure automated response to high-severity events.
- Enrich alerts with threat intelligence and map them to MITRE ATT&CK.
- Simulate attacks and recognise their signatures in the data.
- Investigate an incident from alert to conclusion.
- Troubleshoot a real detection pipeline methodically when it breaks.

---

## Progress

Stages 1 through 3 are complete (firewall, Suricata, Wazuh, detection engineering, active response, attack-and-detect), Stage 5 is underway (VirusTotal enrichment done), and the remaining stages are in progress or planned. The lab is documented continuously as it grows.

*This is an active, evolving project. Feedback and suggestions are welcome.*
