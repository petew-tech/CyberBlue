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
| [08-suricata-br30-bidirectional-flow.png](08-suricata-br30-bidirectional-flow.png) | Suricata EVE `ens5` flow confirmed 4 Kali ICMP requests and 4 TrueNAS replies (`pkts_toserver:4`, `pkts_toclient:4`, `state:established`, `alerted:false`) after adding scoped egress mirror | Uploaded and verified |

## Captured screenshot — 2026-10-17

| Screenshot | Content | Status |
|---|---|---|
| [09-wazuh-post-soc-migration-alerts.png](09-wazuh-post-soc-migration-alerts.png) | Wazuh Threat Hunting after Linux Mint SOC NIC addition: manager `siem-01`, agent `003` (`cyberblue-suricata`), DQL `rule.id: 86601`; four level-3 ICMP test alerts around 12:24 PM CDT on October 17 | Uploaded and GitHub file verified |

Screenshots 01 and 02 are two views of the same detection, not two separate detections. Screenshots 03 and 04 show the later dual-interface configuration and service initialization; they do not demonstrate bridge-wide traffic visibility. The source timestamp and indexed alert timestamp differ slightly, reflecting event collection and processing.

Both original dashboard screenshots have been uploaded to this GitHub folder and their filenames verified.

See [Project validation record](../validation.md).

---

## Module 06 — Network Migration Evidence (2026-10-18)

| Screenshot | What it proves | Status |
|---|---|---|
| [project06-netplan-routing-validated.png](project06-netplan-routing-validated.png) | Ubuntu IPv4 management/SOC retained; old `10.10.30.0/24` route absent after Netplan application | Uploaded and verified |
| [project06-netplan-post-reboot-validation.png](project06-netplan-post-reboot-validation.png) | Ubuntu guest reboot kept management/SOC IPv4 and routes while old Victim IPv4 remained absent | Uploaded and verified |
| [module06-windows-victim-ip-pre-reboot.png](module06-windows-victim-ip-pre-reboot.png) | Windows `10.10.30.10` Victim and `192.168.1.151` management addresses are Preferred before reboot | Uploaded and verified |
| [module06-windows-victim-post-reboot-validation.png](module06-windows-victim-post-reboot-validation.png) | Windows retained intended two addresses and routes after guest reboot | Uploaded and verified |
| [module06-kali-attack-network-pre-reboot.png](module06-kali-attack-network-pre-reboot.png) | Kali Attack `10.10.40.20/24`, management `192.168.1.91/24`, default and connected routes before final reboot | Uploaded and verified |
| [module06-kali-attack-network-post-reboot.png](module06-kali-attack-network-post-reboot.png) | Kali retained the intended two IPv4 addresses and routes after final reboot | Uploaded and verified |

The two files named `project06-` are **Module 06 evidence within Project 05**; filenames were retained because they were already uploaded. Screenshots document specific guest addressing/routing states, not full zone isolation, TrueNAS host bridge persistence or comprehensive IDS visibility.
