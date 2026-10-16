# Project 05 — Screenshot Evidence

This folder is reserved for screenshots captured from the actual CyberBlue lab during Suricata and Wazuh validation.

## Captured screenshots — 2026-10-16

| Screenshot | Content | Status |
|---|---|---|
| [01-wazuh-suricata-alert-details.png](01-wazuh-suricata-alert-details.png) | Wazuh Threat Hunting document details: agent `003` (`cyberblue-suricata`), Suricata SID `1000001`, target `10.10.40.1`, event type `alert` | Uploaded and verified |
| [02-wazuh-suricata-rule-and-timestamp.png](02-wazuh-suricata-rule-and-timestamp.png) | Same alert: source `ens4`, source IP `10.10.40.10`, `/var/log/suricata/eve.json`, rule `86601`, manager `siem-01`, timestamp `2026-10-16 09:29:00.495` as displayed | Uploaded and verified |

These are two views of the same detection, not two separate detections. The source timestamp and indexed alert timestamp differ slightly, reflecting event collection and processing.

Both original dashboard screenshots have been uploaded to this GitHub folder and their filenames verified.

See [Project validation record](../validation.md).
