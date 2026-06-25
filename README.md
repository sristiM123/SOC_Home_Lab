# Home SOC Lab

A hands-on Security Operations Center (SOC) lab built from scratch on a single resource-constrained laptop. This repository documents — day by day — how I built a working detection and response pipeline using Wazuh, integrated host telemetry, wrote custom detection logic, automated responses, and simulated and detected real attacks.

The goal is to learn SOC Analyst (L1/L2) skills by doing, and to document the *real* process — including the failures and the debugging — rather than a polished tutorial. Most of the learning here came from things that broke and had to be fixed.

---

## Why this lab exists

I am working toward a SOC Analyst role. Reading about SIEMs, detection rules, and incident response is not the same as operating them. This lab is where I build the muscle memory: enrolling agents, reading alerts, writing decoders and rules, tuning out noise, automating responses, and recognising attack patterns in live data.

A deliberate constraint shaped this lab: it runs on a single laptop with **8 GB of RAM**. That forced me to understand resource trade-offs that a cloud tutorial would hide — why the Wazuh indexer is memory-hungry, why two heavy components cannot run at once, and how to make disciplined choices about what to run when. Working within real limits taught me more than unlimited resources would have.

---

## Architecture

| Component | Role | Where it runs |
|-----------|------|---------------|
| Wazuh Manager + Indexer + Dashboard | SIEM / XDR core — collects, decodes, correlates, stores, and displays security events | Ubuntu VM |
| Wazuh Agent | Endpoint sensor — ships logs to the manager | Windows host + Kali VM |
| Windows Defender Firewall | Host firewall; dropped-packet logs forwarded to Wazuh | Windows host |
| Kali Linux | Attacker machine for attack simulation | Kali VM |
| Suricata *(planned)* | Network IDS feeding alerts into Wazuh | Ubuntu VM |
| SOAR — Shuffle *(planned)* | Automated response orchestration | Cloud / isolated |

**Host:** Windows laptop, 8 GB RAM, VMware Workstation Player.
**Data flow:** endpoint event → agent collects → manager decodes & matches rules → alert stored in indexer → analyst investigates in dashboard → (optional) active response acts automatically.

---

## What I have built so far

| # | Topic | Status | Writeup |
|---|-------|--------|---------|
| 01 | Wazuh all-in-one setup on constrained hardware | Done | [docs/01-wazuh-setup](docs/01-wazuh-setup/README.md) |
| 02 | Windows Firewall → Wazuh integration | Done | [docs/02-firewall-integration](docs/02-firewall-integration/README.md) |
| 03 | Detection engineering: custom rules, correlation, noise tuning | Done | [docs/03-detection-engineering](docs/03-detection-engineering/README.md) |
| 04 | Active response (automated action on alerts) | Done | [docs/04-active-response](docs/04-active-response/README.md) |
| 05 | Attack & detect: SSH brute-force simulation | Done | [docs/05-attack-and-detect](docs/05-attack-and-detect/README.md) |
| 06 | Suricata network IDS integration | Planned | [docs/06-suricata](docs/06-suricata/README.md) |
| 07 | SOAR / automated playbooks | Planned | [docs/07-soar](docs/07-soar/README.md) |

---

## Key skills demonstrated

- **SIEM operation** — deploying and operating Wazuh end to end; reading and triaging alerts.
- **Log integration** — forwarding host firewall logs into a SIEM and understanding the ingestion-vs-alerting distinction.
- **Detection engineering** — writing custom decoders and rules, building correlation rules, and tuning out benign noise.
- **Automated response** — configuring active response to act on high-severity alerts.
- **Attack simulation & detection** — generating real attack traffic and recognising the resulting signatures.
- **Troubleshooting** — diagnosing failures methodically using `wazuh-logtest`, `wazuh-analysisd -t`, and log inspection rather than guesswork.
- **MITRE ATT&CK & compliance awareness** — interpreting technique mappings and PCI DSS tagging on alerts.

---



## Repository structure

```
home-soc-lab/
├── README.md                  # this file
├── docs/                      # day-by-day writeups, one folder per topic
│   ├── 01-wazuh-setup/
│   ├── 02-firewall-integration/
│   ├── 03-detection-engineering/
│   ├── 04-active-response/
│   ├── 05-attack-and-detect/
│   ├── 06-suricata/           # planned
│   └── 07-soar/               # planned
├── configs/                   # sanitised config snippets (rules, decoders)
├── screenshots/               # dashboard and terminal evidence
└── TEMPLATE.md                # copy this to start a new writeup
```

---

*This lab is a work in progress and updated regularly as I learn. Feedback and suggestions are welcome.*
