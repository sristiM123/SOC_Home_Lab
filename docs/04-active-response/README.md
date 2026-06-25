# 04 — Active Response: Automated Action on Alerts

> **Status:** Done
> **Tools:** Wazuh Manager, active-response scripts, `ossec.conf`

---

## Goal

Move Wazuh from passive monitoring to automated response — making it *act* when a high-severity alert fires, which is the conceptual foundation of SOAR.

## What I did

**A safe, observable response first.** Rather than start with an action that blocks IPs (which can lock you out of your own lab), I built a logging response that proves the mechanism works safely. I created a script the agent runs when triggered:

```bash
#!/bin/bash
LOG_FILE="/var/ossec/logs/active-responses.log"
echo "$(date '+%Y-%m-%d %H:%M:%S') custom-log.sh triggered by Wazuh active response" >> "$LOG_FILE"
exit 0
```

Wazuh requires correct ownership and permissions or it silently refuses to run the script:

```bash
sudo chmod 750 /var/ossec/active-response/bin/custom-log.sh
sudo chown root:wazuh /var/ossec/active-response/bin/custom-log.sh
```

**Registering the command and response** in `ossec.conf`:

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

This triggers the script on any alert of level 10 or above — which includes my custom sensitive-port rules.

## What broke (and how I fixed it)

**Nothing triggered at first.** The active response only fires on level 10+, but my initial test events were below that threshold, so nothing happened. I temporarily lowered the threshold to confirm the wiring, watched the trigger line appear in `active-responses.log`, then restored it to 10 for real use. This proved the end-to-end chain: alert fires → manager decides → agent runs script → action logged.

**Tuning the threshold matters.** Setting it too low makes the response fire constantly on routine events — in a real deployment with an IP-blocking action, that would be dangerous. The right threshold acts only on genuinely significant alerts.

## What I learned

- Active response wires detection to action: detect → decide → act → log.
- Script ownership/permissions are mandatory; Wazuh won't run a script without them.
- The trigger threshold is a safety and tuning decision — too low causes the automation to misfire on noise.
- Proving a powerful mechanism safely (logging) before enabling a destructive one (IP blocking) is the responsible way to build automation.

## SOC relevance

Automated response is what lets a SOC react at machine speed — blocking a scanning IP, isolating a host, or enriching an alert without waiting for a human. Understanding how to wire it *and* how to constrain it so it doesn't act on noise is a strong L2 capability and the bridge to full SOAR platforms.
