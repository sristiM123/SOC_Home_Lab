# Wazuh + VirusTotal Integration: File Reputation Enrichment

> **Status:** Done
> **Tools:** Wazuh Manager, VirusTotal API, Wazuh FIM (File Integrity Monitoring)

---

## Goal

Automatically check files against VirusTotal when they are added or changed on a monitored system. When Wazuh's File Integrity Monitoring detects a file event, it sends the file's hash to VirusTotal and enriches the alert with the reputation result — automated, file-level threat intelligence.

## Background: how the integration works

The integration is not a running service — it is a configuration that makes Wazuh call the VirusTotal API when a relevant alert fires. This makes it very light on resources (only outbound API calls), which suits constrained hardware.

The full flow:

```
A file is added/changed in a monitored folder
   → Wazuh File Integrity Monitoring (FIM) detects the change
   → Wazuh calculates the file's hash (its unique fingerprint)
   → Wazuh sends that hash to the VirusTotal API
   → VirusTotal returns whether the file is known and how many engines flag it
   → Wazuh enriches the alert with the result and shows it in the dashboard
```

A **hash** is a file's fingerprint — a fixed string that uniquely identifies it. Change the file even slightly and the hash changes completely. VirusTotal looks up that hash against its database of known files scanned by ~70 antivirus engines.

## What I did

### 1. Got a free VirusTotal API key

Signed up at virustotal.com (free), then copied the API key from the account's API Key section. The key is treated like a password — kept out of screenshots and chat, placed directly into the config on the machine.

### 2. Added the integration to the Wazuh manager config

Edited `ossec.conf` and added an `<integration>` block before the closing `</ossec_config>`:

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```

- `name virustotal` — use Wazuh's built-in VirusTotal integration.
- `api_key` — the VirusTotal key.
- `group syscheck` — trigger on FIM (file integrity) events, so a watched-file change sends its hash to VirusTotal.
- `alert_format json` — structured output.

### 3. Made FIM watch a folder

VirusTotal only fires on file events, so FIM must be monitoring a directory. Added a watched folder inside the `<syscheck>` block:

```xml
<directories realtime="yes" check_all="yes">/path/to/watched-folder</directories>
```

Then created that folder on disk.

### 4. Validated config and restarted

Same test-before-restart discipline used throughout the lab:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

### 5. Tested with an EICAR file

EICAR is a harmless, industry-standard test file that every antivirus (and VirusTotal) recognises — not real malware, just a known test string. Dropping it into the watched folder triggers FIM, which triggers the VirusTotal lookup:

```bash
# create/modify a file in the watched folder to trigger FIM
echo "test $(date)" >> /path/to/watched-folder/eicar.txt
```

### 6. Confirmed the enrichment

Checked the alert log for the VirusTotal result:

```bash
sudo grep -i virustotal /var/ossec/logs/alerts/alerts.log | tail -20
```

A successful enrichment shows the file's hashes and the integration name, e.g.:

```
virustotal.source.md5: <hash>
virustotal.source.sha1: <hash>
Integration: virustotal
```

---

## Troubleshooting (this took several layers — documented honestly)

The integration did not work on the first try, and diagnosing it was the real learning. The steps below are a reusable checklist for when a Wazuh integration silently fails.

### Symptom
FIM detected file changes, but no VirusTotal alert appeared. The manager log showed:
```
ERROR: Unable to run integration for virustotal -> integrations
ERROR: While running virustotal -> integrations. Output:
```
(blank output — the unhelpful part).

### The actual root cause
**The API key had been entered incorrectly in the config.** A malformed/mistyped key means the integration script runs but the VirusTotal call fails, producing the "unable to run" error with blank output. Correcting the key and restarting fixed it.

The blank error made this hard to spot, because the same symptom can come from several different causes. That is why the checklist below matters — it rules out each possibility methodically.

### Diagnostic checklist (rule these out in order)

1. **Did FIM actually detect the file?** VirusTotal only fires if FIM saw the change first.
   ```bash
   sudo grep -i "modified\|added" /var/ossec/logs/ossec.log | tail
   ```
   Confirm a "File ... modified/added" line for the test file. (In this lab, FIM was working — the file showed as modified.)

2. **Is `jq` installed?** The integration needs it to parse the JSON response.
   ```bash
   which jq        # if missing: sudo apt install -y jq
   ```

3. **Do the integration scripts exist with correct permissions?**
   ```bash
   ls -la /var/ossec/integrations/virustotal*
   ```
   Expect `virustotal` and `virustotal.py`, both `-rwxr-x---` owned `root:wazuh`.

4. **Is the required Python module present?** The script needs `requests`.
   ```bash
   python3 -c "import requests; print('requests OK')"
   # if missing: sudo apt install -y python3-requests
   ```

5. **Run the script's trace to see where it stops:**
   ```bash
   sudo bash -x /var/ossec/integrations/virustotal test test test 2>&1 | tail -30
   ```
   (With dummy arguments the script exits quietly with no output — that is normal, because there is no real alert file to read. It confirms the script itself runs.)

6. **Check the API key.** This was the actual culprit here. Confirm the `<api_key>` value in `ossec.conf` is the exact key with no quotes, no spaces, and no line breaks. A single wrong character causes the "unable to run" error with blank output.

7. **Check timestamps — old errors can mislead.** After a fix, the old ERROR lines remain in the log. Trigger a fresh file change and confirm whether a *new* error appears or a successful enrichment does:
   ```bash
   sudo bash -c 'echo "fresh test $(date)" >> /path/to/watched-folder/eicar.txt'
   sleep 60
   sudo grep -i virustotal /var/ossec/logs/ossec.log | tail -5
   sudo grep -i virustotal /var/ossec/logs/alerts/alerts.log | tail -5
   ```

### Resolution
After correcting the API key and re-triggering, the alert log showed a successful enrichment with the file's md5 and sha1 hashes and `Integration: virustotal`. The stale ERROR lines were from before the fix.

## What I learned

- The VirusTotal integration is lightweight (API calls, not a running service) and is driven by FIM events.
- A "file reaching FIM" and "the integration succeeding" are separate stages — confirm each independently.
- A blank "unable to run integration" error can come from several causes (missing jq, missing Python module, wrong permissions, or a bad API key). A methodical checklist is faster than guessing.
- The actual root cause here was a mistyped API key — a reminder that the simplest explanation (a typo in a credential) is worth checking early, not last.
- Old error lines persist in logs after a fix; always re-trigger and check fresh timestamps before concluding something still fails.

## SOC relevance

Automated file-reputation lookups are a real SOC capability: when a file appears on a monitored system, checking it against a global threat database (VirusTotal) without manual effort lets analysts catch known-malicious files instantly. Understanding how to wire the integration, and how to debug it when it fails silently, is directly relevant L1/L2 work — the debugging discipline especially.
