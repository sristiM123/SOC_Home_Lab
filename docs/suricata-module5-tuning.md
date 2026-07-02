# Suricata Module 5 — Tuning & Performance (Lab)

Hands-on: cutting noise so real threats stand out, and managing the ruleset for performance on constrained hardware.

---

## Goal

Reduce false-positive and low-value noise so genuine signals are visible — the same principle applied to the SIEM, now applied at the IDS layer.

## The problem

Out of the box, Suricata alerts on everything its ruleset matches — including benign traffic. A common example in the lab: the `ET INFO GNU/Linux APT User-Agent Outbound` rule (sid 2013504), which fires constantly on normal package-management traffic. This floods the log and buries real alerts.

## Technique 1 — Suppress a noisy rule

Stop a specific rule from alerting entirely. Edit the threshold/suppression config:

```
# in /etc/suricata/threshold.config
suppress gen_id 1, sig_id 2013504
```

This silences that one rule. The traffic still flows; Suricata just stops alerting on it. Use when a rule is confirmed benign in your environment.

## Technique 2 — Threshold (rate-limit) a rule

Instead of full suppression, limit how often a rule fires:

```
threshold gen_id 1, sig_id 2013504, type limit, track by_src, count 1, seconds 60
```

This alerts at most once per source per 60 seconds — keeping visibility while cutting volume. Useful for rules that are worth seeing occasionally but not hundreds of times.

## Technique 3 — Manage the ruleset

A large ruleset (tens of thousands of rules) costs memory and CPU — noticeable on constrained hardware. You do not need every rule. `suricata-update` can enable/disable rule categories so only relevant rules load:

```bash
sudo suricata-update list-sources        # see available rule sources
sudo suricata-update enable-source <name>
# disable categories that don't apply to your environment
```

Running a leaner, relevant ruleset improves performance and reduces noise at the same time.

## The tuning principle (critical)

**Never suppress without understanding what you are silencing.** The danger of tuning is muting a real threat because it looked like noise. The discipline:

- Confirm an alert is genuinely benign (you know its source and cause) before suppressing it.
- Document *why* each suppression exists.

"Suppressed sid 2013504 because it is routine apt package-management traffic from internal hosts" is a defensible analyst decision. Silencing something merely because it is frequent is how real detections get missed.

## What I learned

- Suppression removes a rule's alerts entirely; thresholding rate-limits them. Choose based on whether the rule ever has value.
- `threshold.config` is where suppression and thresholding live.
- A smaller, relevant ruleset improves both performance and signal-to-noise.
- Tuning is a judgement skill: understand and document what you silence, or risk hiding a real threat.
- This mirrors SIEM tuning — the same discipline applied at a different layer of the stack.
