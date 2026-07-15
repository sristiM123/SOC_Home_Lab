# Building a Custom SOC Dashboard in Wazuh

> **Status:** Done
> **Tools:** Wazuh Dashboard (OpenSearch Dashboards / Kibana-based), `wazuh-alerts-*` index

---

## Goal

Build a custom SOC overview dashboard from scratch — designing visualisations that answer real operational and organisational questions, then arranging them so the layout itself communicates priority.

## Why this matters

Wazuh's dashboard is built on **OpenSearch Dashboards**, a fork of **Kibana**. The visualisation builder, query language, and index concepts are effectively the same. Building dashboards here is transferable Kibana skill, which appears in a large share of SOC job descriptions.

More importantly: default dashboards show generic counts. A *custom* dashboard answers the questions a specific organisation actually cares about — and building one requires understanding both the data and the audience.

---

## The core concept: metrics and buckets

Every visualisation answers two questions:

- **Metric** — *what am I measuring?* (usually `Count`)
- **Bucket** — *how am I splitting it?* (by severity, by agent, over time)

Once that clicks, every chart type is the same idea in a different shape. Data source throughout: the `wazuh-alerts-*` index pattern.

---

## What I built

Seven visualisations, each answering one question:

| Visualisation | Type | Bucket / Field | Question it answers |
|---------------|------|----------------|---------------------|
| Alerts by Severity | Pie | Terms → `rule.level` | How serious is what's firing? |
| Alerts Over Time | Area | Date Histogram → `timestamp` | When did activity spike? |
| Top Rules | Data table | Terms → `rule.description` | What is firing most? |
| Alerts by Agent | Horizontal bar | Terms → `agent.name` | Which host is noisiest? |
| MITRE ATT&CK Techniques | Horizontal bar | Terms → `rule.mitre_techniques` | What would an attacker be doing? |
| PCI DSS Compliance Coverage | Pie | Terms → `rule.pci_dss` | Which compliance requirements are we evidencing? |
| Critical Alerts Gauge | Gauge | Count + filter `rule.level >= 10` | Are we healthy right now? |

The last three are the organisation-level panels. Counts and agent breakdowns are operational; **MITRE, compliance, and a health gauge are what security leads and auditors actually ask for.**

---

## Layout: the design decision

Panels were arranged by *audience and time budget*, not aesthetics:

**Top row — "Am I okay right now?"**
Critical Alerts Gauge + Alerts Over Time. A three-second glance should surface health and any spike.

**Middle row — "What's happening?"**
Alerts by Severity + Alerts by Agent. For an analyst investigating over a few minutes.

**Bottom row — "Details and reporting"**
Top Rules + MITRE ATT&CK + PCI DSS. Information-dense panels for investigation and management reporting.

Each row serves a different reader. That is what makes a dashboard *designed* rather than assembled.

---

## What the dashboard revealed (the real value)

A dashboard's job is to surface things you did not know. Mine surfaced two genuine problems immediately:

### 1. VirusTotal integration was being rate-limited

The **Top Rules** panel showed:

> `VirusTotal: Error: Public API request rate limit reached` — **766 events**

**Root cause:** VirusTotal's free API tier allows roughly 4 requests per minute. File Integrity Monitoring was detecting far more file changes than that (including in busy default-monitored system directories), so most lookups were rejected.

**Fix / mitigation:** scope what triggers the integration — either narrow which directories FIM monitors, or restrict the integration to specific rule IDs (e.g. file-added / file-modified) rather than the entire `syscheck` group:

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_API_KEY</api_key>
  <rule_id>554,550</rule_id>
  <alert_format>json</alert_format>
</integration>
```

In production this would mean a paid API key, scoping lookups to high-risk paths, or caching results.

### 2. A single rule dominating alert volume

> `Summary event of the report's signatures` — **6,970 events**

One rule accounted for the overwhelming majority of alerts — Suricata summary events flooding the SIEM. A clear tuning candidate: high volume, low analytical value.

**This is the point of the exercise.** I did not know about either issue before building the dashboard. Both were found because a visualisation made them visible.

---

## What I fixed

### The gauge was broken

Initially the gauge read **223** with ranges of 0–50 / 50–75 / 75–100. Since 223 exceeded the top range, the gauge was pegged permanently red — it could never move, and therefore communicated nothing.

**Fix:** set ranges that bracket the real data — 0–100 (green), 100–250 (yellow), 250–500 (red). The gauge now sits in yellow ("elevated, not critical") and can actually change state.

**The principle:** gauge ranges must be derived from the data's real distribution. *Normal = green, elevated = yellow, abnormal = red.* A gauge stuck at maximum is decoration, not information.

### The filter was in the wrong place

I first typed `rule.level >= 10` into the visualisation's **Custom label** field, and the gauge still showed the full alert count (15,734).

**Root cause:** Custom label is *cosmetic text* — it renames a display element. It does not filter anything. Filtering happens in the **top search bar** (DQL).

Moving the query to the search bar dropped the count to 223 — the genuine high-severity total.

**The lesson:** search bar = filtering; Custom label = naming. Confusing the two produces a chart that *looks* filtered but is not — which is worse than no chart, because it is confidently wrong.

### A duplicate metric

The gauge initially had two metrics configured (`Metric Count` and a second `Metric`), producing two identical gauges. Removed the duplicate — a gauge needs exactly one metric.

---

## Reading the finished dashboard

- **Alerts by Severity:** dominated by level 4 (low), with a thin tail of higher levels. This shape is what a healthy environment looks like — mostly low-level noise. If the dominant slice were level 12+, that would be a problem. The *shape* tells the story before reading a single alert.
- **Alerts by Agent:** the Windows host dominates, driven by firewall and registry events.
- **MITRE ATT&CK:** widening the time range to 30 days turned a single-technique chart into a meaningful distribution (T1076, T1051, T1078, T1133 and others). Sparse data produces misleading charts — time range matters.
- **PCI DSS:** requirements like 10.6.1, 10.2.x and 11.x appearing automatically on alerts — the raw material of an audit-facing report.

---

## What I learned

- Every visualisation is a **metric** plus a **bucket**. That single idea covers every chart type.
- Gauge ranges must be derived from the actual data distribution, or the gauge is decorative.
- The search bar filters; Custom label only renames. A chart can look filtered and not be.
- Time range dramatically changes what a chart says — a sparse chart usually means too narrow a window, not missing data.
- **Layout is information design.** Ordering panels by audience and urgency is a deliberate decision, not decoration.
- A dashboard's real value is surfacing what you did not know. Mine found a rate-limited integration and a dominant noise rule — neither of which I was looking for.
- Wazuh dashboard skills are Kibana skills. Building here transfers directly.

## SOC relevance

Dashboard building is a core SOC capability. Analysts consume dashboards; engineers *build* them — designing views that answer specific questions for specific audiences, from real-time analyst triage to auditor-facing compliance evidence. The ability to build one, read it correctly, and act on what it reveals is directly relevant L1/L2 work.
