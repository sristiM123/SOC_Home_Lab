# Configs

Sanitised configuration snippets from the lab, kept here for reference and reuse.

## Files to add as you go

- `local_rules.xml` — your custom Wazuh rules (sensitive-port escalation, SSDP suppression, port-scan correlation)
- `local_decoder.xml` — any custom decoders
- `ossec-agent-localfile.xml` — the Windows agent `localfile` block for firewall log forwarding
- `active-response.xml` — the command + active-response blocks from `ossec.conf`

## Before committing

Remove anything environment-specific or sensitive: real public IPs, hostnames you don't want shared, and never commit the Wazuh admin password or certificates.
