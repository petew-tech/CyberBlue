# Project 05 — Screenshot Evidence

This folder is reserved for screenshots captured from the actual CyberBlue lab during Suricata and Wazuh validation.

## Captured screenshots — 2026-10-16

| Screenshot | Content | Status |
|---|---|---|
| [01-wazuh-suricata-alert-details.png](01-wazuh-suricata-alert-details.png) | Wazuh Threat Hunting: agent `003`, Suricata signature `1000001`, and event details | Uploaded and verified |
| [02-wazuh-suricata-rule-and-timestamp.png](02-wazuh-suricata-rule-and-timestamp.png) | Same alert: Wazuh rule `86601`, log source, and timestamp | Uploaded and verified |
| [03-suricata-dual-interface-configuration.png](03-suricata-dual-interface-configuration.png) | AF_PACKET settings for `ens4` (cluster 99) and `ens5` (cluster 100) | Uploaded and verified |
| [04-suricata-dual-interface-initialization.png](04-suricata-dual-interface-initialization.png) | Suricata process `2922` initialized capture threads on both interfaces | Uploaded and verified |
| [05-br30-traffic-visibility-gap.PNG](05-br30-traffic-visibility-gap.PNG) | On Linux Mint `ens5`, tcpdump observed zero matching ICMP packets from Kali during the test; kernel drops were zero | Uploaded and verified |
| [06-br30-truenas-packet-capture.PNG](06-br30-truenas-packet-capture.PNG) | TrueNAS `vnet5` captured 4 ICMP echo requests/replies between Kali `10.10.30.20` and bridge `10.10.30.1`, with 0 kernel drops | Uploaded and verified |
| [07-suricata-br30-mirrored-flow.png](07-suricata-br30-mirrored-flow.png) | Linux Mint Suricata EVE JSON recorded 4 mirrored ICMP echo requests from Kali `10.10.30.20` to `10.10.30.1` on `ens5`; no replies and no alert | Uploaded and verified |

Screenshots 01 and 02 are two views of the same detection, not two separate detections. Screenshots 03 and 04 show the later dual-interface configuration and service initialization; they do not demonstrate bridge-wide traffic visibility. The source timestamp and indexed alert timestamp differ slightly, reflecting event collection and processing.

Both original dashboard screenshots have been uploaded to this GitHub folder and their filenames verified.

See [Project validation record](../validation.md).
