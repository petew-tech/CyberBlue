# Project 05 — Screenshot Evidence

This folder is reserved for screenshots captured from the actual CyberBlue lab during Suricata and Wazuh validation.

## Captured screenshots — 2026-10-16

| File to upload | Content | Status |
|---|---|---|
| `01-wazuh-suricata-alert-details.png` | Wazuh Threat Hunting document details: agent `003` (`cyberblue-suricata`), Suricata SID `1000001`, target `10.10.40.1`, event type `alert` | Captured; awaiting GitHub upload |
| `02-wazuh-suricata-rule-and-timestamp.png` | Same alert: source `ens4`, source IP `10.10.40.10`, `/var/log/suricata/eve.json`, rule `86601`, manager `siem-01`, timestamp `2026-10-16 09:29:00.495` as displayed | Captured; awaiting GitHub upload |

These are two views of the same detection, not two separate detections. The source timestamp and indexed alert timestamp differ slightly, reflecting event collection and processing.

Do not mark an image as uploaded until the actual PNG is present in this folder.

See [Project validation record](../validation.md).
