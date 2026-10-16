# Project 05 — Screenshot Evidence

This folder is reserved for screenshots captured from the actual CyberBlue lab during Suricata and Wazuh validation.

## Captured screenshots — 2026-10-16

| Screenshot | Content | Status |
|---|---|---|
| [01-wazuh-suricata-alert-details.png](01-wazuh-suricata-alert-details.png) | Wazuh Threat Hunting document details: agent `003` (`cyberblue-suricata`), Suricata SID `1000001`, target `10.10.40.1`, event type `alert` | Uploaded and verified |
| [02-wazuh-suricata-rule-and-timestamp.png](02-wazuh-suricata-rule-and-timestamp.png) | Same alert: source `ens4`, source IP `10.10.40.10`, `/var/log/suricata/eve.json`, rule `86601`, manager `siem-01`, timestamp `2026-10-16 09:29:00.495` as displayed | Uploaded and verified |

| [03-suricata-dual-interface-configuration.png](03-suricata-dual-interface-configuration.png) | Linux Mint Suricata YAML entries: `ens4`/cluster ID `99` and `ens5`/cluster ID `100` | Uploaded and verified |
| [04-suricata-dual-interface-initialization.png](04-suricata-dual-interface-initialization.png) | Suricata process `2922` initialized two AF_PACKET capture threads on each of `ens4` and `ens5` | Uploaded and verified |

Screenshots 01 and 02 are two views of the same detection, not two separate detections. Screenshots 03 and 04 show the later dual-interface configuration and service initialization; they do not demonstrate bridge-wide traffic visibility. The source timestamp and indexed alert timestamp differ slightly, reflecting event collection and processing.

Both original dashboard screenshots have been uploaded to this GitHub folder and their filenames verified.

See [Project validation record](../validation.md).
