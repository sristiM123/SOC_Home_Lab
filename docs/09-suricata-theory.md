# Suricata: Theory & Concepts

Foundational theory for Suricata: how the engine works, how to read a rule, and how Suricata is deployed in production (IDS vs IPS). The hands-on practice like writing rules, reading alerts, tuning, and attack detection, is documented in separate files.

---

## Module 1: How Suricata Works

### What Suricata is

Suricata is an **IDS / IPS / NSM** engine:

- **IDS (Intrusion Detection System)** — watches traffic and raises alerts on suspicious activity. Passive.
- **IPS (Intrusion Prevention System)** — same detection, but can block traffic inline. Active.
- **NSM (Network Security Monitoring)** — records metadata about all traffic (connections, DNS, TLS, HTTP), not just alerts.

In this lab Suricata runs in **IDS mode** : detecting and alerting, not blocking. This is the right default for learning and for most monitoring deployments.

### How a packet flows through Suricata

A packet travels through five stages, like an assembly line:

1. **Capture** — Suricata grabs packets off the monitored network interface (using `af-packet` on Linux). This is the tap on the wire.
2. **Decode** — each raw packet is unpacked layer by layer (Ethernet → IP → TCP/UDP → application data) into structured fields Suricata can reason about.
3. **Stream reassembly** — packets belonging to the same conversation are reassembled into a complete flow, so Suricata inspects the whole request rather than fragments. This defeats attackers who split malicious payloads across packets to evade detection.
4. **Detect** — the reassembled traffic is checked against the rules (signatures). If a pattern matches, the rule fires.
5. **Output** — a match is written out, to `fast.log` (human-readable) and `eve.json` (structured JSON). `eve.json` is what feeds the SIEM.

**Full flow:** capture → decode → reassemble → detect → output.

### Two detection approaches

- **Signature-based** — matches traffic against known-bad patterns (rules). Fast and precise, but only catches known threats. The everyday method.
- **Protocol anomaly / behavioral** — flags traffic that violates how a protocol should behave, even without a specific rule. Catches some unknown threats.

### Terminology

- A **rule** (= **signature**) is one line of detection logic.
- A **ruleset** is a collection of rules (e.g. ET Open, ~50k rules).
- An **alert** is what is produced when traffic matches a rule.

Rule is the cause; alert is the effect.

---

## Module 2 — Anatomy of a Suricata Rule

Every rule has three parts:

```
action   header   (options)
```

Read any rule as a sentence: **"DO THIS — to THIS TRAFFIC — when you see THIS."**

### Part 1 — Action

The first word: what to do on a match.
- `alert` — log an alert (used almost always).
- `drop` — block the packet and alert (IPS mode).
- `reject` — block and send a rejection back.
- `pass` — explicitly allow.

### Part 2 — Header

Describes which traffic to inspect. Example: `tcp any any -> any 22`

- **Protocol** — tcp, udp, icmp, http, dns, tls, etc. Suricata understands application protocols, not just ports.
- **Source IP** and **Source port** — where traffic comes from.
- **Direction** — `->` (source to destination) or `<>` (both directions).
- **Destination IP** and **Destination port** — where it is going.

The header is a filter that narrows down which packets get checked against the options.

**Variables** make rules portable:
- `$HOME_NET` — the protected/internal network.
- `$EXTERNAL_NET` — the untrusted/outside network.
- `$HTTP_PORTS` — web ports (80, 8080, etc.).

### Part 3 — Options

The detection logic and metadata, inside parentheses, each ending in `;`.

Core metadata (every rule needs these):
- `msg` — the human-readable alert text.
- `sid` — unique Signature ID. **Custom rules must use 1000000 and above**; lower ranges are reserved for official rulesets.
- `rev` — revision number; bump it when the rule is edited.

Detection options (where matching happens):
- `content` — match specific bytes/text in the payload (the workhorse).
- `pcre` — match with a regular expression for complex patterns.
- `flow` — match connection state/direction, e.g. `flow:established,to_server`.
- `flags` — match TCP flags, e.g. `flags:S` (SYN only).
- `http.uri`, `http_method` — inspect specific parts of an HTTP request.
- `threshold` — control how often a rule fires (rate limiting / correlation).
- `classtype` — a category label (e.g. `attempted-recon`).
- `reference` — link to external info (CVE, advisory URL).
- `metadata` — context such as affected product, severity, confidence, dates.

### Rule types to recognise (by detection mechanism)

- **decode-event** (header `pkthdr`) — anomalies in raw packet structure, the lowest layer (e.g. `ipv4.wrong_ip_version`). Built-in engine rules.
- **app-layer-event** — anomalies in application-protocol behaviour (e.g. protocol mismatch). Built-in engine rules.
- **content signature** — matches specific bytes/strings in the payload (e.g. ET Open rules). The everyday type.

Recognising which type a rule is — the moment you read it — is a genuine expert skill. Most people only read the `msg`; naming the detection mechanism shows real depth.

### The professional way to explain a rule (six steps)

1. **Action & scope** — what it does and which traffic (action + header).
2. **Rule type** — content signature, app-layer anomaly, or decode event.
3. **Core detection** — the option doing the real work, in plain terms.
4. **Supporting options** — flow, threshold, classtype, etc.
5. **Why it matters** — the security relevance.
6. **Metadata** — sid range (tells you the source), rev.

Reading `flow:to_client` and immediately recognising "this is a client-side attack against a browsing user" (versus `to_server`, an attack against your servers) is the kind of read that separates an expert from someone reading words.

---

## Module 7 — IDS vs IPS & Production Deployment

### The core distinction

**IDS (passive)** — Suricata sits beside the traffic, watching a copy. Detects and alerts but cannot stop anything; the packet has already passed. Safe, no risk of blocking legitimate traffic.

**IPS (inline)** — Suricata sits in the traffic path; everything flows through it. Can drop malicious packets before they reach the target. Powerful, but a bad rule can break legitimate traffic (a false positive becomes an outage).

The rule difference is literally the action word: `alert` (IDS) vs `drop` (IPS). Same engine, same rules — the deployment position and action change.

### Deployment

- **IDS** — connected to a SPAN port or network TAP (a mirror of the traffic). Suricata sees everything but is not in the path; if it fails, traffic still flows.
- **IPS** — physically in the traffic path (e.g. between firewall and internal network). If it fails, you need a bypass or traffic stops.

### Why most SOCs run IDS mode

- No risk of blocking legitimate business traffic.
- Analysts investigate and decide, rather than auto-blocking.
- IPS is applied selectively, to high-confidence rules where blocking is worth the risk.

The mature pattern: **IDS for broad visibility, IPS selectively for known-bad, high-confidence threats.**

### Where Suricata sits in a real SOC stack

```
Internet → Firewall → IDS/IPS (Suricata) → Internal network
                            ↓
                      alerts (eve.json)
                            ↓
                     SIEM (Wazuh) ← also receives host logs, firewall logs
                            ↓
                      Analyst investigates
                            ↓
                   SOAR automates response
```

This mirrors the lab architecture: Suricata feeding Wazuh, alongside firewall and host logs, into one analyst view.

### Production realities

- **Performance** — high-traffic networks require tuning for throughput (multi-threading, hardware acceleration). Running a large ruleset on constrained hardware demonstrates the memory cost firsthand.
- **Rule management** — keep rules current with `suricata-update`, disable irrelevant categories, tune out noise.
- **Placement** — the sensor only sees what its network position exposes. (A VM behind NAT scanning the host, for example, may be filtered by the hypervisor before the sensor ever sees it — a real blind spot.)
- **IDS + IPS together** — run IDS broadly, enable IPS-mode drops only for a curated set of high-confidence signatures.

