# Wazuh + LLM Integration: AI-Powered Alert Enrichment & Triage

> **Status:** Done (working pipeline)
> **Tools:** Wazuh Manager, Groq API (Llama 3.3 70B), Python
> **Note on naming:** the integration files are named `custom-gemini` for historical reasons — the project began with Google Gemini, but the free tier proved unusable (see Troubleshooting). The working integration uses **Groq / Llama 3.3**. The name was kept to avoid disturbing a working pipeline; treat "gemini" throughout as the integration label, not the provider.

---

## Goal

When a high-severity alert fires, automatically send it to a Large Language Model and get back a plain-English analysis: what the alert means, a severity assessment, and a recommended first triage step. The result is written to a log, ingested back into Wazuh, and made visible on a dashboard.

## Why do this

A SOC analyst faces hundreds of alerts a day, many with cryptic rule names. For each, they must ask: what is this, is it serious, what do I do? Answering takes time and experience, and the volume causes **alert fatigue** — the single biggest problem in SOC work.

An LLM can translate a cryptic alert into plain language, suggest a severity, and recommend a first step — like giving every junior analyst an experienced mentor on every alert. It is a **triage accelerator, not an autopilot**: the model can be wrong, so a human still decides. This pattern is current in the industry precisely because it targets alert fatigue directly.

---

## Architecture

The integration follows the same pattern as the VirusTotal integration — a script in `/var/ossec/integrations/`, registered in `ossec.conf`, triggered by alerts. Wazuh has **no built-in LLM integration**, so the script is written from scratch.

```
High-severity alert (level >= 10) fires
   → Wazuh calls the integration script, passing the alert as JSON
   → Script builds a prompt and calls the LLM API
   → LLM returns analysis (verdict + explanation + action)
   → Script writes structured JSON to a log
   → Wazuh ingests that log as new alerts (via localfile + rule)
   → Analysis is searchable and visualised on a dashboard
```

**Scoping is deliberate:** the integration triggers only on level 10+ alerts. This is essential — an earlier VirusTotal integration triggered on every file event and generated 766 rate-limit errors. Free LLM tiers are rate-limited, so scoping tightly is both a technical necessity and good practice.

---

## Step 1 — Get an API key

### Google Gemini (attempted first)
1. Go to `aistudio.google.com`
2. Sign in with a Google account
3. **Get API key** → **Create API key** (no card required)

Gemini's free tier turned out to be effectively unusable for this — see Troubleshooting.

### Groq (used in the end)
1. Go to `console.groq.com`
2. Sign in (Google/GitHub)
3. **API Keys** → **Create API Key**
4. Copy the key and store it privately

Groq's free tier is far more generous and the models are actually available. Treat the key like a password — keep it out of screenshots, chat, and any file committed to version control.

---

## Step 2 — The integration script

Two files in `/var/ossec/integrations/`: a shell **wrapper** (no extension) that locates Wazuh's bundled Python, and the **Python script** (`.py`) that does the work.

### The wrapper (`custom-gemini`)

```sh
#!/bin/sh
WPYTHON_BIN="framework/python/bin/python3"
SCRIPT_PATH_NAME="$0"
DIR_NAME="$(cd $(dirname ${SCRIPT_PATH_NAME}); pwd -P)"
SCRIPT_NAME="$(basename ${SCRIPT_PATH_NAME})"
PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd -P)"

if [ -z "${WPYTHON_BIN}" ] || [ ! -f "${WAZUH_PATH}/${WPYTHON_BIN}" ]; then
    python3 "${PYTHON_SCRIPT}" "$@"
else
    "${WAZUH_PATH}/${WPYTHON_BIN}" "${PYTHON_SCRIPT}" "$@"
fi
```

### The Python script (`custom-gemini.py`)

```python
#!/usr/bin/env python3
import sys
import json
import time
import requests

# Wazuh passes: sys.argv[1] = alert file path, sys.argv[2] = api_key (from ossec.conf)
alert_file = sys.argv[1]
api_key = sys.argv[2]

with open(alert_file) as f:
    alert = json.load(f)

rule = alert.get("rule", {})
description = rule.get("description", "No description")
level = rule.get("level", "unknown")
agent = alert.get("agent", {}).get("name", "unknown")
full_log = alert.get("full_log", "")

prompt = f"""You are a SOC analyst assistant. Analyze this security alert.

Alert: {description}
Severity level: {level}
Agent: {agent}
Log: {full_log}

Start your reply with exactly one line: VERDICT: LOW  or  VERDICT: MEDIUM  or  VERDICT: HIGH
Then on the next lines give:
WHAT: what this alert means in plain English
ACTION: one recommended first step for triage
Keep it concise."""

url = "https://api.groq.com/openai/v1/chat/completions"
headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}
data = {
    "model": "llama-3.3-70b-versatile",
    "messages": [{"role": "user", "content": prompt}]
}

try:
    response = requests.post(url, headers=headers, json=data, timeout=30)
    result = response.json()
    if response.status_code == 200:
        ai_text = result["choices"][0]["message"]["content"]
    else:
        ai_text = f"API error {response.status_code}: {json.dumps(result)}"
except Exception as e:
    ai_text = f"AI analysis failed: {str(e)}"

# Parse the verdict from the first line so it can be charted
verdict = "UNKNOWN"
for line in ai_text.splitlines():
    if "VERDICT:" in line.upper():
        v = line.upper().split("VERDICT:")[1].strip().split()[0]
        if v in ("LOW", "MEDIUM", "HIGH"):
            verdict = v
        break

ai_oneline = ai_text.replace("\n", " ").replace("\r", " ").strip()

with open("/var/ossec/logs/gemini-analysis.log", "a") as out:
    out.write(json.dumps({
        "integration": "gemini",
        "gemini": {
            "verdict": verdict,
            "original_alert": description,
            "level": level,
            "agent": agent,
            "analysis": ai_oneline
        }
    }) + "\n")
```

### Permissions (mandatory — Wazuh silently refuses wrong perms)

```bash
sudo chmod 750 /var/ossec/integrations/custom-gemini /var/ossec/integrations/custom-gemini.py
sudo chown root:wazuh /var/ossec/integrations/custom-gemini /var/ossec/integrations/custom-gemini.py
```

---

## Step 3 — Register the integration

In `/var/ossec/etc/ossec.conf`, before `</ossec_config>`:

```xml
<integration>
  <name>custom-gemini</name>
  <api_key>YOUR_API_KEY_HERE</api_key>
  <level>10</level>
  <alert_format>json</alert_format>
</integration>
```

- `name` must match the script filename exactly.
- `api_key` is passed to the script as `sys.argv[2]`. This is the only place the key lives — never hardcode it in the script, never commit it.
- `level 10` scopes the integration to high-severity alerts only (rate-limit protection).

---

## Step 4 — Ingest the AI output back into Wazuh

So the analysis appears on dashboards, the output log is ingested as JSON. In `ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/ossec/logs/gemini-analysis.log</location>
</localfile>
```

Because the log is JSON, Wazuh auto-parses every field (`gemini.verdict`, `gemini.analysis`, etc.) — **no custom decoder needed**. JSON ingestion avoids decoder-writing entirely.

Then a rule in `/var/ossec/etc/rules/local_rules.xml` turns each analysis into an alert:

```xml
<group name="gemini,ai_analysis,">
  <rule id="100200" level="5">
    <decoded_as>json</decoded_as>
    <field name="integration">gemini</field>
    <description>Gemini AI analysis: $(gemini.verdict) - $(gemini.original_alert)</description>
  </rule>
</group>
```

Validate and restart:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

---

## Step 5 — Test

Generate a level 10+ alert (SSH brute force from an attacker machine):

```bash
for i in $(seq 1 10); do ssh wronguser@<target-ip> "test" 2>/dev/null; done
```

Check the AI output:

```bash
sudo tail -5 /var/ossec/logs/gemini-analysis.log
```

Then in the dashboard → Threat Hunting → search `gemini`. The analyses appear as alerts, ready to visualise (a verdict pie chart + a table of full analyses).

---

## Understanding the prompt

The prompt is the most important — and most tunable — part. Ours does four things:

1. **Assigns a role** ("You are a SOC analyst assistant") — framing the model as a domain expert measurably improves relevance.
2. **Provides structured context** — the alert description, level, agent, and raw log.
3. **Forces a machine-readable field** — the mandatory `VERDICT: LOW/MEDIUM/HIGH` first line is what allows the output to be charted. Without a structured field, the response is just prose.
4. **Constrains the format** — WHAT and ACTION, "keep it concise" — to get consistent, short, useful output rather than an essay.

### Prompt-engineering principles applied here

- **Role assignment** sets expertise and tone.
- **Explicit output structure** makes the result parseable (critical for turning text into data).
- **Brevity constraints** keep responses usable in an alert context.
- **Determinism** matters for triage — a lower model temperature (more consistent, less creative) is preferable, so the same alert yields a stable verdict.

### Alternative / richer prompts (and why they were not run here)

These are stronger prompt designs that were **not** implemented, primarily due to the lab's 8GB RAM ceiling and free-tier rate limits — documented so they can be added when more resources are available:

- **Context-enriched prompting** — before calling the LLM, the script gathers related data (recent alerts from the same source IP, any VirusTotal result, asset criticality) and includes it. This yields far better, context-aware triage. Not run here because assembling context means extra queries/lookups per alert, increasing load and API calls beyond the free tier's comfort on constrained hardware.
- **MITRE-aware prompting** — supplying the alert's mapped ATT&CK technique and asking the model to reason about attacker intent and likely next steps. Excellent for higher-quality actions; adds prompt length and tokens (free-tier token limits were already a factor).
- **Few-shot prompting** — including 2–3 worked examples of correctly triaged alerts in the prompt to steer output quality. Very effective, but each example inflates token usage per call — impractical under a rate-limited free tier at any volume.
- **Chain-of-thought / multi-step** — asking the model to reason step by step, or making two calls (one to analyse, one to recommend). Better reasoning, but multiplies latency and API calls — untenable on free-tier limits.
- **Confidence scoring** — asking the model to attach a confidence level to its verdict, so low-confidence outputs can be routed to a human. Cheap to add to the prompt; deferred only to keep the first working version simple.
- **Local model (no prompt limit, full privacy)** — running a model locally (e.g. via Ollama) removes rate limits and keeps data in-house, enabling any of the above freely. **Not possible on 8GB alongside Wazuh + Suricata** — even a small local model needs several GB. This is a Proxmox/upgraded-hardware or cloud item.

---

## Troubleshooting (documented honestly)

The build failed several times before working. The debugging is the real learning.

### Symptom 1 — every analysis logged `AI analysis failed: 'candidates'`

The pipeline ran end to end (alert → integration → script → log), but every entry showed this error.

**Cause:** the first version targeted Google Gemini and parsed the response as `result["candidates"][0]...`. The response did not contain `candidates`, so Python raised a KeyError, which the `except` block caught and logged with the unhelpful message `'candidates'`. The naive code assumed success and never inspected the actual response.

**Lesson:** never assume an API returned the success schema. Inspect the real response — status code and body — before parsing.

### Symptom 2 — Gemini returned HTTP 429 with `limit: 0`

Printing the raw response revealed the true problem: `RESOURCE_EXHAUSTED`, quota `limit: 0` for `gemini-2.0-flash`. This is not "quota used up" — it is **zero free-tier quota** for that model. The API key was valid; the model was simply unavailable on the free tier.

Trying other model names produced HTTP 404 (wrong name) or the same zero-quota 429. Listing the key's available models (`GET /v1beta/models`) showed models that *appeared* available but still returned zero free quota when called.

**Resolution:** switched provider to **Groq (Llama 3.3 70B)**, whose free tier is genuinely usable. Groq uses the OpenAI-compatible schema, so three changes were needed versus Gemini:
- URL → Groq's `chat/completions` endpoint
- Auth → `Bearer` header instead of a URL parameter
- Response parsing → `choices[0].message.content` instead of `candidates`

The rewrite also added proper status-code handling, so any future API error is logged with its real message instead of failing silently.

### Symptom 3 — files appeared missing (`No such file or directory`)

`sudo ls -la /var/ossec/integrations/custom-gemini*` reported no such file, even though the files existed.

**Cause:** the `*` wildcard is expanded by the shell *before* sudo runs, as the normal user — who cannot read inside `/var/ossec/integrations/`. The wildcard failed to expand and the literal `custom-gemini*` was passed to `ls`. The files were fine; the command was the problem. Listing files by explicit name (no wildcard) confirmed they existed.

**Lesson:** wildcards expand with the caller's permissions, not sudo's.

---

## Research findings (practical constraints of no-budget SOC-AI)

- **Free-tier LLM access is model-restricted.** Several Gemini models returned `limit: 0` — zero free quota — forcing a provider switch. A no-budget integration is constrained not just by rate but by which models are usable at all.
- **Silent failure is the default.** Naive integration code assumes a success schema and hides real errors. Robust error handling (inspecting status + body) is essential and should be built in from the start.
- **Scoping is mandatory.** Triggering only on level 10+ prevents the rate-limit flooding seen earlier with VirusTotal. Free tiers cannot absorb per-event LLM calls at real alert volumes.
- **Hardware caps ambition.** Local models (which would remove rate limits and privacy concerns) do not fit alongside a SIEM on 8GB. Richer prompting strategies (few-shot, chain-of-thought, context enrichment) are limited by free-tier token and rate limits.

### Metrics worth measuring (next steps)

- **Latency** — alert to AI response time.
- **Verdict agreement** — how often the AI verdict matches the actual rule level.
- **Accuracy** — whether the plain-English explanation is correct (human-judged).
- **Consistency** — whether the same alert yields the same verdict across runs.

---

## Honest caveats

- The model can be **confidently wrong** — this is an assistant, not a decision-maker.
- Alert data is sent to a **third-party API** — acceptable in a lab, a real privacy concern in production (a point in favour of local models when hardware allows).
- Free tiers are **rate-limited** — this design works at lab volume, not production volume.

## SOC relevance

LLM-assisted triage is an emerging, in-demand capability. Building the integration from scratch — rather than using a black box — demonstrates understanding of how SIEM integrations work, how to design and constrain prompts for machine-readable output, and how to reason honestly about the operational limits (cost, rate limits, privacy, accuracy) of applying AI to security operations.
