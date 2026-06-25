# 01 — Wazuh All-in-One Setup on Constrained Hardware

> **Status:** Done
> **Tools:** Wazuh 4.x (Manager + Indexer + Dashboard), VMware Workstation Player, Ubuntu, Kali

---

## Goal

Stand up a working SIEM as the core of the lab — Wazuh's manager, indexer, and dashboard — on a single 8 GB laptop, with Windows and Kali enrolled as agents.

## What I did

Installed Wazuh using the official all-in-one assistant inside an Ubuntu VM:

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The `-a` flag installs all three components together and generates the certificates automatically — the correct choice for a single-node lab, as opposed to the step-by-step multi-node guide which is meant for distributed clusters.

After install I enrolled two agents: the Windows host and the Kali VM. Both reported as **Active**, confirmed with:

```bash
sudo /var/ossec/bin/agent_control -l
```

## What broke (and how I fixed it)

This was mostly a fight with resources, and it taught me more than the install itself.

**Disk exhaustion.** The VM repeatedly hit 0 bytes free. The virtual disk was only 36 GB and full. I expanded the disk in VMware (powered off → Expand to 60 GB), but the Linux partition didn't grow automatically — VMware only grows the container. I had to extend the partition and filesystem inside Ubuntu:

```bash
sudo growpart /dev/sda 2
sudo resize2fs /dev/sda2
```

**Memory over-allocation.** Earlier attempts crashed because the VMs were allocated more RAM than the host physically had (Ubuntu + Kali totals exceeded 8 GB). The fix was disciplined allocation: cap each VM so the totals respect the physical limit, and never run both heavy VMs at once.

**Leftover partial installs.** A previous failed attempt left half-removed Wazuh packages and orphaned config that caused confusing errors. I learned to purge cleanly (`apt purge`, remove leftover directories, confirm no processes held the ports) before reinstalling with the `-o` overwrite flag.

**Background containers eating RAM.** A previous Shuffle/Docker stack was silently running on boot, consuming gigabytes. Disabling Docker auto-start freed the memory the SIEM needed.

## What I learned

- The all-in-one installer is the right path for a single-node lab; the multi-node guide is a different, more complex procedure.
- Expanding a virtual disk is two separate steps — grow the container in VMware, then grow the partition/filesystem inside the OS.
- A SIEM's indexer (OpenSearch) is genuinely memory-hungry; on limited hardware you must plan what runs when.
- A clean slate matters — leftover partial installs cause failures that look mysterious but are just residue.

## SOC relevance

Operating a SIEM starts with deploying and maintaining it. Understanding its components (manager decodes and correlates, indexer stores and searches, dashboard presents) and its resource profile is foundational knowledge an analyst draws on constantly.
