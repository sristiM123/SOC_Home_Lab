# 03 — Detection Engineering: Custom Rules, Correlation, and Noise Tuning

> **Status:** Done
> **Tools:** Wazuh Manager, local rules/decoders, `wazuh-logtest`, `wazuh-analysisd -t`

---

## Goal

Move from *using* Wazuh to *engineering detections* in it — writing custom rules that build on built-in ones, suppressing benign noise, and correlating repeated events into a single meaningful alert.

## What I did

**Extending a built-in rule.** Rather than replacing Wazuh's built-in firewall rule (4101), I wrote child rules that fire only when a drop targets a sensitive port (RDP 3389, SMB 445) and raise the severity, using `<if_sid>` to chain onto the built-in rule:

```xml
<rule id="100110" level="10">
  <if_sid>4101</if_sid>
  <match>3389</match>
  <description>Firewall drop targeting RDP (3389) - sensitive port</description>
  <group>attack,firewall_drop,</group>
</rule>
```

**Suppressing noise.** The dashboard was flooded with benign SSDP/UDP 1900 multicast drops. I wrote a rule setting their level to 0 so they stop cluttering the alert view while remaining searchable:

```xml
<rule id="100120" level="0">
  <if_sid>4101</if_sid>
  <match>1900</match>
  <description>Benign SSDP/UDP 1900 firewall drop - suppressed</description>
</rule>
```

**Correlation.** I wrote a frequency-based rule to detect a burst — the signature of a port scan — rather than alerting on single drops:

```xml
<rule id="100130" level="12" frequency="5" timeframe="60">
  <if_matched_sid>4101</if_matched_sid>
  <same_source_ip />
  <description>Possible port scan: 5+ firewall drops from same source in 60s</description>
  <group>attack,recon,</group>
</rule>
```

## What broke (and how I fixed it)

**"Field 'action' is static" / "Field 'dstport' is static".** My first rules referenced decoded fields that weren't valid in the context of the chained built-in rule, and the manager refused to start (it validates rules and won't load broken ones — a safety feature). I switched from matching decoded fields to matching the value directly in the log text with `<match>`, which sidestepped the issue.

**The testing discipline that saved me.** After every change I validated *before* restarting:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t      # test config loads
sudo /var/ossec/bin/wazuh-logtest           # test a real log against rules
```

`wazuh-logtest` shows exactly which decoder parsed a log and which rule fired — the core tool for iterating on detections. Confirming the suppression rule fired at level 0 on a sample SSDP line proved the tuning worked.

## What I learned

- `<if_sid>` lets you build on top of built-in rules instead of replacing them.
- Matching decoded fields vs. matching raw log text behave differently depending on which decoder is in scope.
- Correlation (`frequency` + `timeframe` + `same_source_ip`) turns many low-value events into one high-value alert.
- Noise tuning is a top SOC skill — but you must understand what you're silencing before you silence it, or you risk muting a real threat.
- Always test rule changes before restarting; the manager won't start on broken XML.

## SOC relevance

Detection engineering is the difference between an L1 who consumes alerts and an L2 who creates and tunes them. Suppressing benign noise, escalating genuinely sensitive events, and correlating patterns into actionable alerts is exactly what keeps a SOC from drowning in false positives.
