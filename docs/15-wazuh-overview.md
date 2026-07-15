# Wazuh — Complete Module Overview

A reference guide to Wazuh's capabilities: what each module is, why it exists, where it fits in SOC work, and how it is used. Written while building and operating a home SOC lab.

---

## What Wazuh is

Wazuh is an open-source **SIEM** (Security Information and Event Management) and **XDR** (Extended Detection and Response) platform. It collects security data from endpoints, analyses it against detection rules, stores it for searching, and presents it for investigation.

### The four components

| Component | Role |
|-----------|------|
| **Agent** | Lightweight program on each monitored endpoint. Collects logs, monitors files, checks configuration, inventories software. The sensor. |
| **Manager** | Receives data from agents. Runs decoders (parse raw logs into fields) and rules (decide what is alert-worthy and how severe). The brain — all detection logic lives here. |
| **Indexer** | Stores processed alerts and makes them searchable. Built on OpenSearch (an Elasticsearch fork). |
| **Dashboard** | Web interface for searching, visualising, and investigating. Built on OpenSearch Dashboards (a Kibana fork). |

### The data flow

```
Endpoint event → Agent collects → Manager decodes & matches rules
→ alert generated → stored in Indexer → investigated in Dashboard
→ (optional) Active Response acts, or an Integration enriches
```

Understanding this flow is what enables troubleshooting. "Logs arrive but no alert appears" is a *rule* problem, not an agent problem — knowing which stage broke is the diagnostic skill.

---

## Endpoint Security modules

### Configuration Assessment (SCA)

**What it is:** scans systems against hardening benchmarks — primarily CIS (Center for Internet Security) benchmarks.

**Why it exists:** most breaches exploit misconfiguration, not zero-days. SCA answers "is this system *configured* securely?" rather than "is something attacking it?"

**How it works:** each check runs either a command (e.g. a PowerShell query of the security policy) or a registry/file lookup, then compares the result against the benchmark's expected value. Each check returns Passed, Failed, or Not Applicable, and produces an overall score.

**Where you use it:** proactive hardening and audit readiness. Findings come with a **rationale** (why it matters) and **remediation** (how to fix it).

**The analyst skill:** prioritisation, not perfection. A personal machine failing half the CIS checks is normal — enterprise benchmarks assume locked-down builds. The question is always *which failures matter for this system's role and exposure*. A failed domain-policy check on a standalone laptop is irrelevant; a failed firewall-state check on an internet-facing server is not.

### Malware Detection

**What it is:** checks for indicators of compromise from malware infections or attacks.

**Why it exists:** detects known-malicious artefacts on endpoints (rootkits, suspicious files, anomalies) rather than network activity.

**Where you use it:** confirming whether a suspicious host shows signs of infection during triage.

### File Integrity Monitoring (FIM)

**What it is:** watches directories and files for changes — content, permissions, ownership, attributes.

**Why it exists:** unauthorised file changes are a core intrusion signal. Attackers modify system binaries, drop payloads, alter configs. FIM catches this.

**How it works:** configured via `<directories>` entries inside the `<syscheck>` block in `ossec.conf`. Supports `realtime="yes"` for immediate detection, or scheduled scans. Wazuh computes file hashes, which enables downstream enrichment (see VirusTotal integration).

**Where you use it:** monitoring critical system paths, web roots, and configuration directories. Also required for compliance frameworks (PCI DSS explicitly requires file integrity monitoring).

**Practical note:** watching busy system directories generates high event volume. Scope FIM deliberately.

---

## Threat Intelligence modules

### Threat Hunting

**What it is:** the main alert-browsing and investigation view.

**Why it exists:** this is where an analyst spends most of their time — searching alerts, filtering by agent and time, and investigating what fired.

**Where you use it:** every triage. Select an agent, set a time range, search, and read the resulting alerts. Note that this view shows **alerts** (rule matches), not raw logs — a log that matched no rule will not appear here.

### Vulnerability Detection

**What it is:** identifies known CVEs in software installed on monitored systems.

**Why it exists:** answers "does this system run software with known flaws?" — distinct from SCA, which asks about configuration. Different problem, different fix: SCA → change a setting; vulnerabilities → patch or upgrade.

**How it works:** Wazuh inventories installed packages per agent and matches them against public vulnerability feeds, reporting CVE ID, severity (Critical/High/Medium/Low), affected package, and version.

**The analyst skill:** triage by *real exposure*, not raw count. A machine may show hundreds of CVEs; the meaningful questions are:
- Start with Critical and High (CVSS ≥ 7.0; 9.8 typically means remotely exploitable, no authentication, full compromise).
- **Is it actually exploitable here?** A critical CVE in a development library that is installed but never run is lower practical risk than a medium in an exposed, network-facing service.
- Is a patch available?

A real example from this lab: ~300 vulnerabilities dropped to 11 Critical when filtered — and of those, the genuinely urgent one was remote-access software, because it is network-exposed by design. Everything else was development tooling with no exposed service. The deliverable is not "we have 300 vulnerabilities"; it is "we have 11 critical, of which one needs action today."

### MITRE ATT&CK

**What it is:** maps alerts to adversary tactics and techniques from the MITRE ATT&CK framework.

**Why it exists:** translates "an alert fired" into "here is what an attacker would be *doing*." Executives and security leads ask what adversaries are attempting, not how many alerts fired — ATT&CK answers that in a framework everyone recognises.

**Important caution:** an ATT&CK-tagged alert is **not** evidence of an attack. Benign system activity frequently matches technique patterns (for example, running inside a VM can map to Virtualization/Sandbox Evasion). Distinguishing a technique-tagged alert from an actual intrusion is analyst judgement.

---

## Security Operations modules

### IT Hygiene

**What it is:** assesses system, software, process, and network layers to detect misconfigurations, unauthorised changes, and anomalies.

**Why it exists:** a consolidated view of the environment's baseline health.

### Compliance modules — PCI DSS, GDPR, HIPAA, NIST 800-53, TSC

**What they are:** views that map alerts to specific regulatory framework requirements.

**Why they exist:** compliance drives security budgets and audits. Regulated organisations (finance, healthcare, retail) must *prove* they monitor specific requirements. These modules turn raw alerts into auditor-facing evidence.

- **PCI DSS** — payment cardholder data security. Requirements like 10.2.4 (invalid access attempts) and 10.6.1 (log review) appear automatically on relevant alerts.
- **GDPR** — EU personal data protection.
- **HIPAA** — US healthcare data privacy and security.
- **NIST 800-53** — US federal security controls catalogue.
- **TSC** — Trust Services Criteria (SOC 2 auditing).

**Where you use them:** audit preparation and demonstrating control coverage. Wazuh tags alerts with the framework requirements they satisfy, so a compliance view is generated from the same data the SOC already collects.

**Practical value:** being able to say "this alert maps to PCI DSS 10.2.4" is the difference between a security tool and an audit-ready security programme.

---

## Cloud Security modules

These collect security events from cloud and SaaS platforms via their APIs, extending monitoring beyond on-premise endpoints.

| Module | What it monitors |
|--------|------------------|
| **Docker** | Container activity — creation, running, starting, stopping, pausing events |
| **Amazon Web Services** | AWS security events, collected via the AWS API |
| **Google Cloud** | GCP service security events, via the GCP API |
| **GitHub** | Audit-log events from GitHub organisations |
| **Office 365** | Security events from Office 365 services |
| **Microsoft Graph API** | Microsoft Graph service events, via the Graph API |

**Why they exist:** modern environments are not only endpoints. Identity, code repositories, cloud infrastructure, and SaaS are all attack surfaces. Cloud modules bring that telemetry into the same SIEM as host and network data.

**Where you use them:** any organisation with cloud infrastructure. AWS/GCP modules catch things like unusual API calls or IAM changes; GitHub catches repository and access changes; Office 365 catches identity and email events.

*(Noted here as platform capabilities — not configured in this lab, which is on-premise only.)*

---

## Detection Engineering: decoders and rules

The core of Wazuh, and the difference between operating the tool and engineering with it.

**Decoders** parse raw log text into structured fields (source IP, port, username, action). Without a decoder, a log is just a string.

**Rules** evaluate decoded fields and decide: does this warrant an alert, at what severity (level 0–15), and in which groups?

Key rule capabilities:
- `<if_sid>` — fire only if another rule already matched. Enables **building on top of** built-in rules rather than replacing them.
- `<frequency>` + `<timeframe>` + `<same_source_ip>` — correlation. Turns many low-value events into one high-value alert (e.g. "5+ drops from one source in 60s" = possible scan).
- `<level>` — severity. Levels 0–3 informational, 4–7 low, 8–11 medium, 12–15 high. Level 0 means "decode but do not alert" — the mechanism for suppressing known-benign noise.

**Custom rules** live in `/var/ossec/etc/rules/local_rules.xml` and must use SIDs ≥ 100000 (lower ranges are reserved).

**Essential discipline:** always validate before restarting. The manager refuses to start on invalid rules — a safety feature, but it means an untested edit takes the SIEM down.

```bash
sudo /var/ossec/bin/wazuh-analysisd -t     # validate config loads
sudo /var/ossec/bin/wazuh-logtest          # test a real log against decoders/rules
sudo systemctl restart wazuh-manager       # only after a clean test
```

`wazuh-logtest` is the key iteration tool: paste a real log line and it shows exactly which decoder parsed it, which fields were extracted, and which rule fired.

---

## Active Response

**What it is:** makes Wazuh *act* when an alert fires, rather than only reporting.

**Why it exists:** machine-speed response. Blocking a scanning IP, killing a process, or running a remediation script without waiting for a human.

**How it works:** a `<command>` block defines a script; an `<active-response>` block defines when to run it (which command, where, and at what alert level threshold).

```xml
<command>
  <name>custom-log</name>
  <executable>custom-log.sh</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<active-response>
  <command>custom-log</command>
  <location>local</location>
  <level>10</level>
</active-response>
```

Scripts live in `/var/ossec/active-response/bin/` and **require correct ownership and permissions** (`chmod 750`, `chown root:wazuh`) — Wazuh silently refuses to run a script without them.

**Critical design consideration:** the level threshold is a safety decision. Set too low, the response fires constantly on routine events — with a benign logging script that is merely noisy, but with an IP-blocking action it becomes an outage or a self-lockout. Build and verify with a harmless action first, then swap in a powerful one.

---

## Integrations

**What they are:** configuration that makes Wazuh call an external service to enrich alerts. Not running services — just outbound API calls, so they are very light on resources.

**Example — VirusTotal:** when FIM detects a file change, Wazuh sends the file's hash to VirusTotal and enriches the alert with the reputation result (how many antivirus engines flag it).

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

Other available integrations include Slack, PagerDuty, Shuffle (SOAR), and Maltiverse — scripts live in `/var/ossec/integrations/`.

**Practical note:** integrations that call rate-limited APIs need scoping. Triggering a VirusTotal lookup on *every* file event in busy system directories exhausts the free tier's request limit quickly.

---

## The dashboard: search, visualise, investigate

Since the dashboard is built on OpenSearch Dashboards (a Kibana fork), the skills transfer directly to Kibana.

- **Index pattern:** `wazuh-alerts-*` — where alert data lives.
- **Query language:** DQL, in the top search bar (e.g. `rule.level >= 10`).
- **Visualisations:** each chart answers one question, built from a **metric** (what you measure — usually Count) and a **bucket** (how you split it — by agent, severity, time).
- **Dashboards:** arranged sets of visualisations.

**A distinction worth remembering:** the top search bar filters data; a visualisation's "Custom label" field only renames a display element. Confusing the two produces a chart that looks filtered but is not.

---

## Summary: which module answers which question

| Question | Module |
|----------|--------|
| Is this system configured securely? | Configuration Assessment (SCA) |
| Does it run software with known flaws? | Vulnerability Detection |
| Did a file change unexpectedly? | File Integrity Monitoring |
| Is there malware on this host? | Malware Detection |
| What is happening right now? | Threat Hunting |
| What would an attacker be doing? | MITRE ATT&CK |
| Can we prove compliance? | PCI DSS / GDPR / HIPAA / NIST / TSC |
| What is happening in our cloud? | Cloud Security modules |
| Is this file known-malicious? | VirusTotal integration |
| Can we respond automatically? | Active Response |
