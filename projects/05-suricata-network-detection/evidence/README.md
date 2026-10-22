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

## Captured Screenshot — 2026-10-20: Victim Suricata to Wazuh

| Screenshot | Evidence supported | Status |
|---|---|---|
| [05-victim-suricata-wazuh-detection.png](05-victim-suricata-wazuh-detection.png) | Operator saved screenshot of Wazuh manager CLI summary showing `MATCHING ALERTS: 4` for Suricata SID `1000002`, agent `003`, Wazuh rule `86601` and UTC timestamps around 18:16:07–18:16:11. Console output was supplied separately and validated as [V-115](../validation.md#project-05--victim-sid-1000002-end-to-end-suricata-and-wazuh-alert-2026-10-20). | File **uploaded and existence verified** in GitHub on 2026-10-20; screenshot visual content not independently inspected |

This is a **terminal evidence screenshot**, not a Wazuh dashboard screenshot. It supports the bounded Windows ICMP / temporary TrueNAS packet-mirroring / Suricata / Wazuh test, **not** continuous network-wide detection. Filename begins `05-` but refers to a different October 20 artifact than the earlier `05-br30-traffic-visibility-gap.PNG`.

## Wazuh Victim Alert Dashboard — 2026-10-20

| Screenshot | Operator's intended view | Verification |
|---|---|---|
| [10-wazuh-victim-alert-overview.png](10-wazuh-victim-alert-overview.png) | Wazuh Threat Hunting overview of the Victim alert, including timestamp, Wazuh rule `86601`, and Suricata agent information | **File uploaded and presence verified** in the GitHub evidence directory; image pixels not independently inspected |
| [11-wazuh-victim-alert-details.png](11-wazuh-victim-alert-details.png) | Expanded detection fields for Suricata signature `1000002` and Windows Victim source/destination | **File uploaded and presence verified** in GitHub; image pixels not independently inspected |

The operator separately confirmed in Wazuh Threat Hunting that `data.alert.signature_id` was `1000002` for an October 20 alert returned under Wazuh `rule.id:86601`. Earlier read-only manager `alerts.json` queries verified **four** matching SID 1000002 events from agent `003`; dashboard screenshot upload verification by itself does **not** establish that all four are visible in the same dashboard view. No persistent packet mirroring is claimed.

[Validation V-116](../validation.md#project-05--wazuh-victim-alert-dashboard-and-screenshot-evidence-2026-10-20).

## Post-Recovery Wazuh Validation — 2026-10-22

| Screenshot | Recorded scope | Verification |
|---|---|---|
| [12-project05-post-recovery-wazuh-validation.png](12-project05-post-recovery-wazuh-validation.png) | Operator-uploaded SIEM_01 terminal screenshot intended to document Wazuh ingestion of four Attack SID `1000001` alerts under rule `86601` from agent `003` after restarting the VMs. Four corresponding manager JSON records were independently supplied in the guided test. | **GitHub file presence verified**; screenshot pixels not independently inspected |

[Validation V-119–V-123](../validation.md#project-05--post-recovery-attack-detection-validation-2026-10-22). No permanent Victim-network mirror or bridge-wide IDS coverage is implied.
