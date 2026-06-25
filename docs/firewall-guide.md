# Firewalls for SOC Analysts: From Home Lab to Enterprise

A complete reference connecting what I built in my home lab (Windows Defender Firewall + Wazuh) to how firewalls work in enterprise SOC environments. Organized so it is clear: what transfers directly, what is similar with different terminology, and what is genuinely different.

---

## Part 1 — The Foundations (these are universal)

These concepts are identical everywhere, from a laptop firewall to a data-center Palo Alto. 

### The anatomy of a firewall rule

Every rule, on every firewall, answers the same questions:

- **Direction** — inbound (traffic coming in) or outbound (traffic going out).
- **Action** — allow, deny/drop, or reject.
- **Source** — where the traffic comes from (an IP, a range, a zone).
- **Destination** — where it is going.
- **Service / Port / Protocol** — what kind of traffic (TCP 443, UDP 53, etc.).

In my lab I set these in Windows Defender Firewall. On an enterprise firewall, the same five fields exist: they are just presented in a policy table with more columns.

### Drop vs Reject (a distinction)

- **Drop** — silently discard the packet; the sender gets no reply (appears as "filtered" in nmap). Preferred at the perimeter because it gives an attacker no information.
- **Reject** — actively refuse and send back a "denied" response (appears as "closed"). Faster feedback, but it reveals the firewall is there.

This is why my Kali nmap scan showed ports as **filtered**, the firewall was dropping, not rejecting.

### Stateful vs Stateless

- **Stateful** — remembers active connections. If you allow an outbound request, the reply is automatically permitted back. Almost all modern firewalls are stateful.
- **Stateless** — inspects each packet in isolation with no memory. Faster, less secure, mostly legacy or used for simple ACLs.

This is why you only write a rule for the request, not the response.

### Default-deny and least privilege

- **Default-deny** — block everything by default, then explicitly allow only what is needed. The single most important firewall design principle.
- **Least privilege** — open only the specific ports and services genuinely required, to the specific sources that need them.

### Ingress vs Egress filtering

- **Ingress (inbound)** — what most people think of: blocking traffic coming in. Stops external scanning and unauthorised access.
- **Egress (outbound)** — the underrated one: restricting what leaves the network. This catches malware that is *already inside* trying to reach its command-and-control server, and blocks data exfiltration. Mature SOCs care deeply about egress.

### Key ports to know cold

| Port | Service | Why it matters |
|------|---------|----------------|
| 22 | SSH | Remote admin; brute-force target |
| 80 | HTTP | Unencrypted web |
| 443 | HTTPS | Encrypted web; most traffic hides here |
| 53 | DNS | Used for exfiltration and C2 |
| 445 | SMB | Ransomware lateral movement (WannaCry) |
| 3389 | RDP | Top ransomware entry point |
| 23 | Telnet | Obsolete, unencrypted — flag any use |
| 25/587 | SMTP | Mail; abused for spam/exfiltration |

**Everything in Part 1 transfers directly from my lab to any enterprise firewall.**

---

## Part 2 — What I Did in the Lab, and Its Enterprise Equivalent

| What I did in the lab | The enterprise equivalent | Same / Similar / Different |
|----------------------|---------------------------|----------------------------|
| Created an inbound block rule for RDP (3389) in Windows Defender Firewall | Creating a security policy rule on a perimeter firewall to deny RDP from untrusted zones | **Same concept**, different UI. Enterprise uses zones and policy tables, not a host GUI. |
| Enabled "Log dropped packets" to pfirewall.log | Enabling logging on a firewall policy and forwarding to a SIEM via syslog | **Same idea.** Enterprise firewalls stream logs over syslog, not a local text file. |
| Forwarded firewall logs into Wazuh (SIEM) | Forwarding Palo Alto / Fortinet logs into Splunk, Sentinel, QRadar, or Wazuh | **Same architecture.** This is exactly what a SOC does, at scale. |
| Read firewall drop alerts in the dashboard | Triaging firewall events in the SIEM during monitoring/investigation | **Same skill.** The core L1 activity. |
| Wrote a correlation rule for repeated drops (port scan) | SIEM correlation rules, or the firewall's own IPS detecting a scan | **Similar.** Enterprise firewalls often detect scans natively via built-in IPS. |
| Hit the per-profile logging blind spot | Misconfigured logging on a policy/zone causing missing telemetry | **Same class of problem.** A real operational gotcha at any scale. |
| Discovered hypervisor NAT filtering traffic | Understanding NAT, network paths, and where inspection happens | **Same principle**, bigger scale (NAT, DMZs, segmentation). |

**Takeaway:** the *architecture* you built — firewall blocks, firewall logs, logs flow to SIEM, analyst investigates — is precisely the enterprise model. What changes is scale and the specific products.

---

## Part 3 — Host Firewalls vs Enterprise Firewalls

### What you used: a host firewall

Windows Defender Firewall protects **one machine**. It uses **profiles** (Domain / Private / Public) to apply different rules depending on the network the machine is connected to. Host firewalls are real and important — they are the "last line" on each endpoint — but they are not what sits at the network perimeter.

### What enterprises use: network / perimeter firewalls

These are dedicated appliances (physical or virtual) that sit between networks and inspect all traffic crossing the boundary. Key differences:

**Zones instead of profiles.** Instead of Domain/Private/Public, enterprise firewalls define **security zones** — e.g. `trust` (internal), `untrust` (internet), `dmz` (public-facing servers). Rules are written between zones: "allow trust → untrust on 443."

**Next-Generation Firewall (NGFW) features.** Modern enterprise firewalls do far more than port/IP filtering:
- **Application awareness** — identify the actual app (e.g. "this is BitTorrent on port 443") regardless of port. Palo Alto calls this App-ID.
- **User awareness** — tie rules to user identity, not just IP (Palo Alto User-ID, integrated with Active Directory).
- **Intrusion Prevention (IPS)** — detect and block known attack patterns inline.
- **SSL/TLS inspection** — decrypt and inspect encrypted traffic (with certificates), since most traffic is HTTPS now.
- **Threat intelligence feeds** — automatically block known-malicious IPs/domains.
- **Sandboxing** — detonate suspicious files in isolation (e.g. Palo Alto WildFire).

**Centralised management.** Hundreds of firewalls managed from one console (Panorama for Palo Alto, FortiManager for Fortinet), with version-controlled policy.

**High availability and throughput.** Clustered, redundant, handling gigabits of traffic.

---

## Part 4 — The Major Enterprise Firewalls (and what they look like)

You do not need hands-on with all of these, but you should recognise the names and know roughly what each is.

### Palo Alto Networks (PAN-OS)
The market leader for NGFW; very common in SOC job descriptions.
- **Zones**: trust, untrust, dmz.
- **Signature features**: App-ID (application identification), User-ID (identity-based rules), WildFire (sandboxing).
- **Logs look like**: structured fields — type=TRAFFIC/THREAT, source/dest zone, app, action, rule name.
- **Management**: Panorama (centralised).
- **Interview phrase**: "App-ID lets you write policy by application rather than port."

### Fortinet (FortiGate / FortiOS)
Extremely common, especially in mid-market and across many SOCs.
- **Concept**: "security fabric" — firewalls integrated with switches, endpoints, etc.
- **Logs look like**: key-value pairs — `srcip=`, `dstip=`, `action=`, `service=`, `policyid=`.
- **Management**: FortiManager + FortiAnalyzer (logging/reporting).
- **Note**: FortiGate logs are one of the most common things you will parse in a SOC.

### Cisco (ASA, and newer Firepower / FTD)
Long-established; ASA is older/stateful, Firepower adds NGFW features.
- **ASA logs look like**: numbered message IDs, e.g. `%ASA-6-302013` (connection built), `%ASA-4-106023` (denied by ACL).
- **Concept**: ACLs (access control lists) — the older but still widely used rule format.
- **Interview phrase**: knowing that `%ASA-4-106023` means a denied connection is a useful party trick.

### pfSense / OPNsense
Open-source firewalls (FreeBSD-based). Common in home labs, small businesses, and as a free way to learn perimeter concepts.
- **Why relevant to you**: this is the realistic "next step up" from host firewalls in a lab. Running pfSense in a VM (when you have the RAM/cloud resources) would let you practise true perimeter-firewall concepts — zones, NAT rules, WAN/LAN interfaces — for free.

### Cloud firewalls (increasingly important)
- **AWS Security Groups / NACLs**, **Azure NSGs**, **GCP firewall rules** — software-defined firewalls in cloud environments. Same five-field logic, configured as code/policy.
- Worth knowing these exist, as cloud SOC work is growing fast.

---

## Part 5 — What Transfers, What is Similar, What is Different

### Transfers directly (you already have these)
- The five-field rule model.
- Default-deny, least privilege, ingress/egress.
- Stateful vs stateless, drop vs reject.
- The concept of firewall logging → SIEM → analyst triage.
- Reading a drop event and judging benign vs suspicious.
- Port knowledge.

### Similar but with different terminology / format
- **Profiles (host) → Zones (enterprise).** Same idea of context-based rules.
- **pfirewall.log → syslog streams.** Same purpose, different delivery and format.
- **Your custom Wazuh correlation rule → firewall IPS / SIEM correlation.** Same detection logic, different engine.
- **Reading drop alerts → triaging firewall events at scale.** Same skill, far higher volume and more correlation.

### Genuinely different (gaps to be aware of)
- **NGFW features**: App-ID, User-ID, SSL inspection, sandboxing — no host-firewall equivalent.
- **Zone-based, identity-based policy** at scale.
- **Vendor-specific log formats** — you will need to learn to read Palo Alto / Fortinet / Cisco ASA logs; each looks different.
- **Centralised management** of fleets of firewalls.
- **Rule-base auditing** — finding shadowed, redundant, or overly permissive rules in a policy of thousands.
- **NAT at scale, VPN inspection, DMZ and segmentation design.**

---

## Part 6 — How to Talk About This in an Interview

The honest, strong framing:

> "I've built hands-on experience with host firewalls — Windows Defender Firewall — including writing rules, enabling drop logging, and forwarding those logs into a SIEM (Wazuh) where I wrote custom detection and correlation rules. I understand the underlying principles — default-deny, least privilege, stateful inspection, ingress and egress filtering — apply directly to enterprise firewalls like Palo Alto and Fortinet, though the terminology differs (zones instead of profiles) and NGFWs add capabilities like application and user awareness. I haven't operated those appliances directly yet, but I can read their logs and I understand the architecture."

This is honest, demonstrates real understanding, and shows you know the gap and how to close it. Far stronger than pretending to enterprise experience you don't have — interviewers test claims, and getting caught overstating is fatal.

---

## Part 7 — Highest-Value Next Steps (firewall-specific)

You've hit diminishing returns on host firewalls. If you want to strengthen the firewall dimension specifically:

1. **Learn to read Fortinet and Palo Alto log formats.** You can find sample logs online; practise identifying source, destination, action, and rule. This is directly useful day one in a SOC.
2. **Stand up pfSense/OPNsense** (when resources allow — likely cloud or more RAM) to practise true perimeter concepts: zones, WAN/LAN, NAT rules.
3. **Understand NAT, DMZ, and segmentation** conceptually — how traffic flows and where inspection happens.
4. **Don't over-invest further in host firewalls** — you've covered them well. Breadth across the rest of the SOC skillset (SIEM depth, IDS/Suricata, incident response, MITRE ATT&CK) is now higher value.

---

*This guide maps lab experience to enterprise reality. The principles you've learned are the foundation; the enterprise layer is terminology, scale, and vendor specifics on top of that foundation.*
