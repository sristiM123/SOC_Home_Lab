# 05 — Attack & Detect: SSH Brute-Force Simulation

> **Status:** Done
> **Tools:** Kali (attacker), Ubuntu (target + Wazuh agent), Wazuh Manager

---

## Goal

Complete the full SOC loop — launch a real attack, have Wazuh detect it, and read the resulting signature like an analyst.

## What I did

After a failed first approach (see below), I ran an **SSH brute-force** from Kali against the Ubuntu machine — a classic attack that Wazuh detects natively through system auth logs.

Confirmed SSH was running on the target, then launched repeated failed logins from Kali:

```bash
for i in $(seq 1 10); do ssh wronguser@<ubuntu-ip> -o StrictHostKeyChecking=no -o ConnectTimeout=2; done
```

Ten failed attempts in quick succession is a textbook brute-force pattern.

## What broke (and how I fixed it)

**The first attempt — Kali scanning Windows — produced no detection, and the reason was the most valuable lesson of the whole lab.**

I ran an nmap port scan from Kali against the Windows host. The scan showed ports as `filtered` (something was blocking them) and Kali could ping Windows — yet no firewall drops appeared in the Windows log or in Wazuh.

After methodically ruling out logging settings (logging *was* enabled for the correct profile), the conclusion was: **VMware's NAT layer was filtering the VM-to-host traffic before Windows Defender Firewall ever saw it.** Kali is a VM behind NAT; the Windows host sits outside that virtual network. The hypervisor dropped the scan in a place neither the host firewall nor Wazuh could observe.

**The fix:** attack a target on the *same* virtual network. Kali → Ubuntu (both VMs on the same NAT network) has no hypervisor boundary in the path, so the attack lands and is detected cleanly.

## Evidence

The dashboard showed a sharp spike of `authentication_failed` events at the moment of the attack — "PAM: User login failed" and "unix_chkpwd: Password check failed" — clustered in time and sourced from Kali's IP, auto-mapped to PCI DSS 10.2.4 / 10.2.5 (invalid access attempts).

## What I learned

- Virtual network topology can hide attacks: a NAT'd VM scanning the host is filtered by the hypervisor — a real detection blind spot. VM-to-VM on the same network works cleanly.
- `filtered` in a scan means "blocked somewhere," not necessarily "blocked by the host firewall."
- A brute-force signature is a *burst* of auth failures from one source in a short window — the pattern, not any single failure, is the detection.
- Wazuh detects SSH brute-force through native auth-log monitoring, no custom plumbing needed.

## SOC relevance

SSH brute-force is one of the most common alerts a real SOC sees. Recognising the signature — a sudden cluster of authentication failures from one source — and triaging "who is this and is it an attack?" is exactly the L1 job. The networking insight (knowing *where* in the stack traffic is filtered) is the kind of infrastructure understanding that strengthens L2 work.
