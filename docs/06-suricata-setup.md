# 06 — Suricata Network IDS: Installation & Configuration

> **Status:** In progress
> **Tools:** Suricata 8.x, Ubuntu, ET Open ruleset

---

## Goal

Add network-based intrusion detection to the lab with Suricata, so the SOC can see *network traffic* (packets crossing the wire) in addition to *host logs* (events on a machine). The end goal is to integrate Suricata's alerts into Wazuh for a single view of both host and network telemetry.

## Background: where Suricata fits

The lab already had host visibility through Wazuh (reading logs) and a firewall (blocking and logging connections). Suricata adds the missing layer — **network intrusion detection**. It inspects the actual packets flowing through a network interface and compares them against known attack patterns called signatures (rules). When traffic matches a rule, Suricata raises an alert.

The combined picture:

```
Firewall   → blocks/logs connections
Wazuh      → reads host logs
Suricata   → inspects network packets
```

Together these give host-level and network-level visibility, which is the model a real SOC operates on.

## What I did

### 1. Checked resources first

Before installing anything, I confirmed available memory. Suricata is relatively light (a few hundred MB), but loading a large ruleset increases its footprint, so checking headroom first is a habit worth keeping on constrained hardware:

```bash
free -h
```

### 2. Installed Suricata

```bash
sudo apt update
sudo apt install -y suricata
```

Verified the install by confirming the binary location and package status:

```bash
which suricata
dpkg -l | grep suricata
```

This confirmed Suricata was installed at `/usr/bin/suricata` (version 8.x).

### 3. Identified the monitoring interface

Suricata must be told which network interface to watch. I listed the interfaces and identified the active one (the interface in the `UP` state with the host's IP — not the `lo` loopback):

```bash
ip -br a
```

In a VMware Ubuntu VM this is typically named `ens33` (yours may differ — `eth0`, `enp0s3`, etc.).

### 4. Pointed Suricata at the correct interface

Edited the main config and set the capture interface under the `af-packet` section:

```bash
sudo nano /etc/suricata/suricata.yaml
```

```yaml
af-packet:
  - interface: ens33   # change to your active interface name
```

The default config often references `eth0`, which may not exist on a given machine — so this step matters.

### 5. Downloaded detection rules

Suricata ships with a rule-management tool that fetches the free community ruleset (ET Open):

```bash
sudo suricata-update
```

This loaded tens of thousands of signatures. Conceptually this is similar to updating antivirus definitions — it pulls current known-attack patterns. In production, rules are kept up to date so detection stays effective, and commercial environments often layer paid rule feeds on top of ET Open.

### 6. Validated the config before running

Same test-before-run discipline used throughout the lab:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

A successful test ends with **"Configuration provided was successfully loaded"** and reports the number of rules loaded with zero failures.

### 7. Started Suricata and confirmed it was running

```bash
sudo systemctl start suricata
sudo systemctl status suricata --no-pager | grep Active
```

Confirmed `active (running)`, then re-checked memory to ensure the loaded ruleset didn't exhaust available RAM.

### 8. Verified detection with a test alert

Suricata's ruleset includes a signature designed to fire on a known test URL — the standard "is my IDS working?" check:

```bash
curl http://testmynids.org/uid/index.html
```

Then checked Suricata's alert log:

```bash
sudo tail -5 /var/log/suricata/fast.log
```

The test signature fired, confirming Suricata is inspecting traffic and generating alerts correctly.

## What I learned

- Suricata is an **IDS** that inspects network packets, complementing host-log monitoring rather than replacing it.
- It runs in **IDS mode** by default — it detects and alerts but does not block (IPS mode would block). This is the right starting point for learning.
- The capture interface must match the machine's actual active interface; the default config value cannot be assumed correct.
- Rules (signatures) are the detection logic, managed and updated with `suricata-update`, analogous to keeping antivirus definitions current.
- Validating the config before starting the service prevents broken-config surprises — the same discipline that applies to Wazuh.
- Suricata writes alerts to `fast.log` (human-readable) and `eve.json` (structured JSON) — the latter is what will be fed into Wazuh.

## SOC relevance

Network IDS is a core SOC data source. Suricata alerts reveal scanning, exploit attempts, malware command-and-control traffic, and policy violations that host logs alone would miss. Understanding how to deploy it, point it at the right interface, manage its rules, and verify detection is directly relevant L1/L2 work — and combining network IDS alerts with host telemetry in a SIEM is exactly how modern SOCs achieve full visibility.

## Next step

Integrate Suricata's `eve.json` output into Wazuh so network alerts appear in the SIEM dashboard alongside host and firewall alerts. *(To be documented.)*
