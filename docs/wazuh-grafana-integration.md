# Integrating Grafana with Wazuh: SOC Dashboards Beyond the Built-In UI

> **Status:** Done
> **Tools:** Grafana, Wazuh Indexer (OpenSearch), grafana-opensearch-datasource plugin

---

## Why Grafana when Wazuh already has dashboards?

Wazuh's built-in dashboard is **OpenSearch Dashboards**, a fork of **Kibana**. It works well and is tightly integrated with Wazuh's data. So why add Grafana at all?

This is worth answering honestly, because "because it looks nice" is not a reason.

### What Grafana genuinely adds

**1. Multiple data sources in one view.** This is the real differentiator. Kibana is built around Elasticsearch/OpenSearch — it visualises data from that one store. Grafana is data-source agnostic: it connects to OpenSearch, Prometheus, MySQL, PostgreSQL, InfluxDB, CloudWatch, Loki and dozens more, **simultaneously, in the same dashboard**. A single panel row can show Wazuh security alerts next to host CPU/memory metrics from Prometheus next to application logs from Loki. Kibana cannot do that natively.

For a SOC, that matters: correlating "alert volume spiked" with "the server was at 100% CPU at the same moment" is a genuinely different capability.

**2. It is the industry standard for operational dashboards.** Grafana appears in job descriptions on its own, independent of any SIEM. Kibana skills are usually tied to an Elastic/OpenSearch stack; Grafana skills are portable across monitoring, DevOps, SRE and security roles.

**3. Alerting across sources.** Grafana's alerting engine can trigger on conditions spanning different data sources, with a wide range of notification channels.

**4. Visualisation flexibility.** More panel types, more layout control, and a large library of community-built dashboards that can be imported.

### What Kibana / Wazuh's dashboard does better

Being fair about the trade-off:

- **Tighter integration with the data.** Wazuh's dashboard understands Wazuh's field mappings natively. Field labelling "just works" — something that requires manual attention in Grafana (see Troubleshooting below).
- **Purpose-built security views.** Wazuh ships with modules for SCA, vulnerability detection, MITRE ATT&CK and compliance that are wired to its data model. Recreating those in Grafana would be significant manual work.
- **No extra resource cost.** It is already running as part of Wazuh.
- **Full-text log exploration.** Kibana's Discover view is stronger for raw log searching than Grafana's equivalent.

### The honest conclusion

Grafana does not replace Wazuh's dashboard — it complements it. Wazuh's built-in views remain better for security-specific analysis; Grafana is the right tool when correlating security data with infrastructure telemetry, or when standardising dashboards across an organisation that already runs Grafana.

For a SOC lab, the value is twofold: a real, separate, widely-used skill, and understanding *when* each tool is the right choice.

---

## Resource consideration (important on constrained hardware)

Unlike API-based integrations (VirusTotal, LLM enrichment) which cost almost nothing, **Grafana is a running service**. Measured impact on this lab:

| State | Available RAM |
|-------|---------------|
| Before installing Grafana | ~1.8 Gi |
| With Grafana running | ~1.2 Gi |

Roughly **600 Mi** consumed continuously. On an 8 GB machine also running Wazuh (manager + indexer + dashboard) and Suricata, that is significant.

**Mitigation:** disable autostart and run Grafana only when needed. Dashboards persist on disk, so nothing is lost by stopping the service.

```bash
sudo systemctl stop grafana-server
sudo systemctl disable grafana-server
# start deliberately when needed:
sudo systemctl start grafana-server
```

Always check headroom before and after adding a service:
```bash
free -h
```

---

## Step 1 — Install Grafana

Add the official repository and install:

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana
```

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server     # expect: active (running)
```

Access at `http://localhost:3000` — default credentials `admin` / `admin`, which it prompts you to change on first login.

**Take a VM snapshot before installing.** This is the first continuously-running service added to the lab since earlier memory-related crashes; a snapshot makes rollback trivial.

---

## Step 2 — Install the OpenSearch data source plugin

Wazuh's indexer is OpenSearch, so Grafana needs the corresponding plugin:

```bash
sudo grafana-cli plugins install grafana-opensearch-datasource
sudo systemctl restart grafana-server
```

---

## Step 3 — Connect Grafana to the Wazuh indexer

First, confirm the indexer is reachable and the credentials work — better to verify on the command line than to debug inside a UI:

```bash
curl -k -u admin:YOUR_INDEXER_PASSWORD https://localhost:9200
```

Returning cluster JSON confirms three things at once: the indexer is up, **HTTPS** is correct (Wazuh enables TLS by default), and the credentials are valid. The `-k` flag skips certificate verification, which is necessary because Wazuh uses a self-signed certificate.

Then in Grafana: **Connections → Data sources → Add new data source → OpenSearch**

| Setting | Value |
|---------|-------|
| URL | `https://localhost:9200` (**https**, not http) |
| Basic auth | Enabled |
| Skip TLS Verify | Enabled (self-signed certificate) |
| User | `admin` |
| Password | the Wazuh indexer admin password |
| Index name | `wazuh-alerts-*` |
| Time field name | `timestamp` |

Click **Save & test**. Success is confirmed by both the index and the time field validating.

*(The indexer password is generated at install time and can be retrieved from the install files bundle produced by the Wazuh installer.)*

---

## Step 4 — Building panels (for beginners)

Grafana uses the same underlying logic as Wazuh's dashboard builder, with different terminology. The mapping:

| Wazuh dashboard (Kibana) | Grafana |
|---|---|
| Metric (Count) | Metric (Count) |
| Bucket → Terms | Group By → Terms |
| Bucket → Date Histogram | Group By → Date Histogram |
| Visualization type | Panel type (top-right selector) |
| DQL query syntax | **Lucene** query syntax |

That last row matters: a filter written as `rule.level >= 10` in Wazuh's DQL becomes **`rule.level:>=10`** in Grafana's Lucene box. Small difference, easy to trip on.

### The six panels built

**1. Alerts Over Time** (Time series)
- Metric: Count
- Group By: Date Histogram → `timestamp`
- Shows spikes in activity — the first thing an analyst scans for.

**2. Alerts by Severity** (Pie / Bar)
- Metric: Count
- Group By: Terms → `rule.level`, Size 10
- Distribution of alert severity levels.

**3. Alerts by Agent** (Bar chart)
- Metric: Count
- Group By: Terms → `agent.name`, Size 10
- Which host generates the most alerts.

**4. Top Rules** (Table)
- Metric: Count
- Group By: Terms → `rule.description`, Size 10
- What is actually firing — the most information-dense panel.

**5. High Severity Count** (Stat / Gauge)
- Metric: Count
- Group By: Date Histogram → `timestamp`
- Lucene query: `rule.level:>=10`
- Value options → Show: **Calculate**, Calculation: **Total**
- A single number: how many high-severity alerts in the window.

**6. MITRE ATT&CK Techniques** (Bar chart)
- Metric: Count
- Group By: Terms → `rule.mitre_techniques`, Size 10
- Time range: **Last 30 days** — MITRE-tagged alerts are sparse, and a short window produces a misleadingly empty chart.

### Assembling and arranging

Panels are added to a single dashboard and arranged by dragging the panel title and resizing from the corner. The layout follows the same priority logic used for the Wazuh dashboard:

- **Top row:** High Severity Count + Alerts Over Time — the "am I okay right now?" glance.
- **Middle row:** Alerts by Severity + Alerts by Agent — investigation context.
- **Bottom row:** Top Rules + MITRE Techniques — detail and reporting.

Rename via the **gear icon** (Dashboard settings) → General → Title. Set a default time range by selecting it in the time picker and saving with "save current time range as default".

---

## Troubleshooting (encountered and resolved)

### Every panel became its own dashboard

Creating each visualisation separately produced six single-panel dashboards rather than one dashboard with six panels.

**Cause:** in Grafana, panels belong to a dashboard. Saving after building each panel creates a new dashboard each time, unless you return to the existing dashboard and add the next panel to it.

**Fix:** build all panels within one dashboard (returning to the dashboard between panels), or copy panels across via the panel menu (**More → Copy**, then **Add → Paste panel** on the target dashboard). Delete the leftover empty dashboards afterwards.

### Pie chart rendered as a single solid slice

The severity pie initially showed one solid circle with the legend reading "Count" and "rule.level" instead of separate slices per level.

**Cause:** the panel's **Value options → Show** was set to `Calculate` with calculation `Last *`, which collapses every bucket into a single value. The query was correct; the display setting was reducing the data.

**Fix:** set **Show → All values**, which renders every returned bucket as its own slice.

**The wider lesson:** `Calculate` vs `All values` is the single most consequential display setting in Grafana panels — and its correct value is the *opposite* for different panel types. A pie chart needs **All values** (show every bucket); a Stat panel needs **Calculate → Total** (collapse all buckets into one number). Choosing the wrong one produces a chart that looks plausible but is wrong.

### Legend labels showed the field name or "Count" instead of the actual values

Even after fixing the slice count, the legend repeated "Count" rather than showing the severity levels.

**Cause:** Grafana's OpenSearch data source does not automatically label Terms buckets in some panel types. The **Value options → Fields** setting controls which field supplies displayed values, and getting it to render bucket names alongside counts in a pie chart is awkward.

**Partial fix / pragmatic resolution:** switching the panel to a **Bar chart** or **Table** labels OpenSearch Terms buckets correctly without further configuration. This is a known rough edge in the Grafana + OpenSearch combination rather than a configuration error — and one of the concrete ways Wazuh's own dashboard is smoother for this data.

**Diagnostic tip:** the panel's **Table view** toggle shows the raw returned data. If the values are present there, the query is fine and the problem is purely presentational.

### Stat panel showed multiple gauges instead of one number

The "High Severity Count" panel rendered several gauges (one per severity level) rather than a single total.

**Cause:** a Terms group-by on `rule.level` was splitting the result, and/or the value calculation was not set to total.

**Fix:** use only a Date Histogram group-by (no Terms), and set **Value options → Show: Calculate**, **Calculation: Total**. Grafana requires a group-by, so a Date Histogram on `timestamp` is the standard approach — the Stat panel then reduces the resulting series to one figure.

### Menu and UI paths differ between Grafana versions

Buttons such as "Add visualization", the settings gear, and the save controls have moved between recent Grafana releases.

**Approach:** rather than following any single documented path, check the version (`grafana-server -v`) and look for the equivalent control — typically the **+** button, the **gear** icon on the right-hand toolbar, or a panel's own dropdown menu.

---

## What I learned

- **Grafana complements rather than replaces Kibana/Wazuh dashboards.** Its real advantage is multi-source correlation and portability across roles; Wazuh's own dashboard remains better for security-specific views and field handling.
- **A running service has a real, measurable cost.** Grafana consumed ~600 Mi continuously. On constrained hardware, deliberately starting and stopping a service is a legitimate operational decision.
- **The same conceptual model transfers.** Metric + bucket is metric + group-by. Learning one visualisation tool substantially transfers to another; only the terminology and query syntax change (DQL → Lucene).
- **`Calculate` vs `All values` is the setting to understand.** It determines whether a panel shows every bucket or reduces them to a single number, and the correct choice is opposite for pies and stats.
- **Verify connectivity outside the UI first.** A single `curl` against the indexer confirmed protocol, credentials and availability before any Grafana configuration — much faster than debugging through a web form.
- **Tool integration is rarely seamless.** Field labelling with the OpenSearch data source required workarounds. Recognising when to solve a problem versus when to switch panel types and move on is a practical engineering judgement.

## SOC relevance

Grafana is widely deployed for operational and security dashboards, and appears in job requirements independently of any specific SIEM. Building dashboards in both Grafana and a Kibana-based interface demonstrates transferable visualisation skill, an understanding of the trade-offs between tools, and the ability to integrate separate systems — connecting a visualisation platform to a SIEM's data store, with TLS and authentication, is a small but genuine integration task.
