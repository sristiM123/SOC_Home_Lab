# 07 — SOAR / Automated Playbooks

> **Status:** Planned
> **Tools:** Shuffle (SOAR), Wazuh

---

## Goal

Add a SOAR layer to orchestrate automated playbooks — enriching alerts and taking coordinated action — building on the active-response foundation from section 04.

## Plan

- Run Shuffle in an isolated environment (it is container-heavy and cannot coexist with Wazuh on 8 GB RAM, so it runs separately — likely on a free-tier cloud VM with Wazuh sending alerts to it over the network).
- Build a simple playbook: on a high-severity Wazuh alert, enrich the source IP against a threat-intel source and take an action.
- Document the integration, the playbook logic, and the end-to-end flow from Wazuh alert to automated response.

## Note on hardware

This section reflects a real constraint I worked through: Wazuh + Suricata + Shuffle cannot all run simultaneously on 8 GB RAM. The honest engineering answer is to run Shuffle in isolation or on free cloud resources rather than forcing everything onto one machine — a trade-off worth understanding rather than hiding.

*This section will be filled in as I build it.*
