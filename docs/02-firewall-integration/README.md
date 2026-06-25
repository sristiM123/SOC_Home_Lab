# 02 — Windows Firewall → Wazuh Integration

> **Status:** Done
> **Tools:** Windows Defender Firewall, Wazuh agent, Wazuh Manager

---

## Goal

Learn how host firewalls work in practice and integrate them into the SOC pipeline, so that a blocked connection on Windows surfaces as an alert in Wazuh.

## What I did

**On Windows:** In Windows Defender Firewall with Advanced Security, I created an inbound rule to block RDP (TCP 3389), which taught me the five parts of any firewall rule — direction, action, protocol, port, and profile.

I then enabled firewall logging via the firewall's Properties, turning on **Log dropped packets**, which writes every blocked connection to:

```
C:\Windows\System32\LogFiles\Firewall\pfirewall.log
```

**Forwarding to Wazuh:** I pointed the Wazuh agent at that log file by adding a `localfile` block to the agent config (`ossec.conf`):

```xml
<localfile>
  <location>C:\Windows\System32\LogFiles\Firewall\pfirewall.log</location>
  <log_format>syslog</log_format>
</localfile>
```

After restarting the agent, the dropped-connection events flowed from Windows into the manager.

## What broke (and how I fixed it)

**Logs arrived but no alerts appeared.** This was the central lesson. The firewall logs were reaching the manager (visible in the raw data path), but the Threat Hunting view stayed empty. The reason: ingesting a log and generating an alert are two separate stages. A log only becomes an alert if a rule matches it, and the dashboard shows alerts, not raw logs.

**Searching by filename returned nothing.** I searched `pfirewall` and found nothing, because Wazuh categorises events by **rule group** (`firewall`, `firewall_drop`), not by source filename. Once I searched by rule group / rule ID, the alerts appeared — Wazuh's built-in rule 4101 ("Firewall drop event") was already decoding and alerting on them.

## What I learned

- A log reaching the SIEM is not the same as an alert — two distinct stages.
- Events are found by rule group and rule ID, not by the source filename.
- Wazuh ships built-in decoders and rules that often already handle common log sources — worth checking before writing custom logic.

## SOC relevance

Firewall drop logs are an early signal of scanning, malware beaconing, or unauthorised access attempts. Knowing how to get them into a SIEM and where to find the resulting alerts is core L1 work. Equally important is the judgement that most drops (e.g. routine multicast/discovery traffic) are benign — distinguishing noise from signal is the actual skill.
