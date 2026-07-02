# Reading & Understanding Alerts

Hands-on: investigating Suricata alerts using `eve.json`, the structured output, the way an analyst does.

---

## Goal

Move from *making* alerts fire to *investigating* them — the core L1/L2 triage skill.

## fast.log vs eve.json

Suricata writes two alert outputs:

- **fast.log** — one human-readable line per alert. Quick to scan, minimal detail.
- **eve.json** — one structured JSON event per line. Rich context: every field an analyst needs. This is also the file the SIEM ingests.

For investigation, `eve.json` is the one that matters.

## Reading a raw alert

Pull the most recent alert event and pretty-print it:

```bash
sudo grep '"event_type":"alert"' /var/log/suricata/eve.json | tail -1 | python3 -m json.tool
```

## The fields that matter (and what each tells you)

| Field | What it tells you |
|-------|-------------------|
| `src_ip` / `dest_ip` | Who talked to whom |
| `src_port` / `dest_port` | Which service/ports were involved |
| `proto` | Transport protocol (TCP/UDP/ICMP) |
| `app_proto` | Application protocol (http, dns, tls) |
| `alert.signature` | The rule that fired (the `msg`) |
| `alert.signature_id` | The sid — tells you the rule's source (custom, ET, built-in) |
| `alert.severity` | How serious Suricata rates it (lower number = higher severity) |
| `alert.category` | The classtype grouping |
| `flow_id` | Links all events belonging to the same conversation |
| `timestamp` | When it happened |

## The analyst triage method

For any alert, answer five questions fast:

1. **Source** — where did it come from? (internal, known, or unknown external?)
2. **Destination** — what was targeted?
3. **What fired** — read `alert.signature`.
4. **How severe** — check `alert.severity` and `alert.category`.
5. **Benign or real?** — does the context make this expected, or is it a genuine threat?

The same alert can be benign or serious depending on context. A port scan from your own test machine is benign; the identical alert from an unknown external IP is an incident. Judging that difference is the job.

## eve.json is more than alerts

`eve.json` also records non-alert events via `event_type`, e.g. `flow`, `dns`, `http`, `tls`, `stats`. This is the NSM (network security monitoring) side — a record of what happened on the network even when nothing malicious fired. During an investigation, correlating an alert with the surrounding `flow`, `dns`, and `http` records for the same `flow_id` reconstructs the full picture.

## What I learned

- `eve.json` (structured) is for investigation; `fast.log` (one-liner) is for a quick glance.
- An alert is triaged by five questions: source, destination, signature, severity, benign-or-real.
- Severity in Suricata is inverted — a lower number means higher severity.
- `flow_id` ties related events together, enabling reconstruction of a full conversation.
- Context determines whether an alert matters — the same signature can be noise or an incident.
