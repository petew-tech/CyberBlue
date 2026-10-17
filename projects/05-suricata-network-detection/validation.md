# Project 05 — Validation Record

**Validation date:** 2026-10-16  
**Sensor:** Linux Mint 22.3, `cyberblue-suricata`  
**SIEM:** Wazuh manager `siem-01` (`192.168.1.213`)

## Test Results

| ID | Test | Evidence / observed result | Status |
|---|---|---|---|
| V-01 | Suricata enabled at boot | `systemctl is-enabled suricata` returned `enabled` | PASS |
| V-02 | Suricata running | `systemctl is-active suricata` returned `active` | PASS |
| V-03 | Automatic restart policy | `Restart=on-failure` | PASS |
| V-04 | Rule load after recovery | PID 13422 logged 53,144 rules loaded and 0 failed | PASS |
| V-05 | Isolated-network connectivity | Ping to `10.10.40.1`: 4 transmitted, 4 received, 0% loss | PASS |
| V-06 | Custom Suricata rule | SID `1000001` logged `CYBERBLUE TEST - ICMP Echo Request` | PASS |
| V-07 | Wazuh agent enrollment/connectivity | Agent ID `003`; log confirmed connection to `192.168.1.213:1514/tcp` | PASS |
| V-08 | EVE JSON collection | Wazuh logcollector reported analyzing `/var/log/suricata/eve.json` | PASS |
| V-09 | Centralized Wazuh detection | Wazuh rule `86601` recorded the custom Suricata signature from `/var/log/suricata/eve.json` | PASS |
| V-10 | Dashboard visibility | Operator confirmed rule `86601` visible in Threat Hunting | PASS |
| V-11 | Controlled process failure | Suricata main process PID 3502 terminated using SIGKILL | PASS — failure injected |
| V-12 | Automatic recovery | Main PID changed to 13422; `NRestarts=1`; service active/running | PASS |
| V-13 | Detection after recovery | New SID `1000001` alert observed at 09:29:00 CDT and confirmed in Wazuh | PASS |
| V-14 | Packet capture statistics | 278 kernel packets, 0 kernel drops, 0 capture errors in sampled stats event | PASS |
| V-15 | Bridge-wide visibility | Temporary TrueNAS `vnet5` ingress+egress filtered ICMP mirrors to `vnet7` verified for Kali ↔ TrueNAS gateway; broader traffic, other hosts, `br40`, and persistence unverified | PARTIAL — OPEN |
| V-16 | Dual-interface Suricata initialization | `ens4` and `ens5` each created 2 capture threads under PID `2922`; screenshots `03` and `04` | PASS — initialization only |
| V-17 | `ens5` EVE flow ingestion | DHCP and IPv6-ICMP flow records tagged `in_iface: ens5` | PASS — observed traffic only |
| V-18 | Kali-to-gateway unicast observation | Kali `10.10.30.20` pinged `10.10.30.1` successfully (4/4); matching `tcpdump` on Mint `ens5` saw 0 ICMP packets and 0 kernel drops | VISIBILITY GAP — further investigation required |
| V-19 | Controlled ICMP packet mirror | TrueNAS `tc clsact` on `vnet5` with ingress flower source `10.10.30.20`, `ip_proto icmp`, action `mirred egress mirror dev vnet7`; 4 packets / 336 bytes mirrored, no drops | PASS — scoped test |
| V-20 | Mirrored packet ingestion | Linux Mint `tcpdump` on `ens5` captured all four mirrored echo requests with zero kernel drops; Suricata EVE JSON `in_iface: ens5` flow recorded `pkts_toserver:4`, `pkts_toclient:0`, `alerted:false` | PASS — one-way flow |
| V-21 | Bidirectional ICMP mirror on `br30` | Added TrueNAS `vnet5` egress flower filter matching `10.10.30.1` → `10.10.30.20` ICMP with `mirred egress mirror dev vnet7`; Linux Mint `tcpdump` on `ens5` captured 4 requests + 4 replies, 8 captured, 0 kernel drops | PASS — scoped test |
| V-22 | Bidirectional Suricata flow decoding and logging | Suricata ICMPv4 cumulative decoder counter rose 12 → 20 following a further 4-ping test; separately, EVE flow for the 13:06 exchange logged at 13:11 shows `in_iface:ens5`, `pkts_toserver:4`, `pkts_toclient:4`, `state:established`, `alerted:false` | PASS — ICMP bidirectional |

## Screenshot Evidence

Actual Wazuh Threat Hunting screenshots from October 16, 2026:

1. [Suricata alert details](evidence/01-wazuh-suricata-alert-details.png) — Wazuh agent `003`, source address `192.168.1.92`, Suricata signature `CYBERBLUE TEST - ICMP Echo Request` (SID `1000001`), destination `10.10.40.1`.
2. [Wazuh rule and timestamp](evidence/02-wazuh-suricata-rule-and-timestamp.png) — `ens4`, source `10.10.40.10`, event source `/var/log/suricata/eve.json`, manager `siem-01`, Wazuh rule `86601`, and recorded timestamp.

Both screenshots are separate views of the same alert, not two independent detections.

Dual-interface rollout evidence (same date):

3. [Suricata AF_PACKET dual-interface configuration](evidence/03-suricata-dual-interface-configuration.png) — `ens4` configured with cluster ID `99` and `ens5` configured with cluster ID `100`; configuration test exited `0`.
4. [Suricata dual-interface initialization](evidence/04-suricata-dual-interface-initialization.png) — process `2922` logged two capture threads initialized on each interface. **EVE flow records confirm some traffic on `ens5`, but Kali's gateway ICMP traffic was not visible to the sensor. Comprehensive inter-VM visibility remains unverified.**
5. [br30 traffic visibility gap](evidence/05-br30-traffic-visibility-gap.PNG) — Linux Mint's `ens5` capture showed zero matching packets for the tested Kali ICMP exchange, although the Kali ping succeeded. This documents a tested observation gap, not full packet-loss statistics.
6. [TrueNAS host-side packet capture](evidence/06-br30-truenas-packet-capture.PNG) — `tcpdump` on Kali's host-side `vnet5` captured 4 ICMP echo requests/replies involving `10.10.30.20` and `10.10.30.1`, with 0 kernel drops. This confirms host-side observation for the tested flow.

7. [Suricata br30 mirrored flow](evidence/07-suricata-br30-mirrored-flow.png) — after temporary TrueNAS mirroring, Suricata `eve.json` on Linux Mint recorded all 4 Kali ICMP requests on `ens5`, no replies (`pkts_toclient:0`) and no alert (`alerted:false`).

8. [Suricata br30 bidirectional flow](evidence/08-suricata-br30-bidirectional-flow.png) — ICMP exchange beginning 13:06 logged as EVE flow at 13:11, `ens5`, `pkts_toserver:4`, `pkts_toclient:4`, `state:established`; this is flow telemetry, not an intrusion alert.

## Representative Centralized Alert

Wazuh recorded the custom detection with:

- Agent: `cyberblue-suricata` (ID `003`, address `192.168.1.92`)
- Manager: `siem-01`
- Wazuh rule: `86601`, level 3
- Suricata signature ID: `1000001`
- Source: `10.10.40.10`
- Destination: `10.10.40.1`
- Protocol: ICMP, echo request
- Location: `/var/log/suricata/eve.json`

## Conclusion

**PASS:** The tested detection path from ICMP traffic delivered to the Linux Mint sensor through Suricata EVE JSON to Wazuh alerts and the dashboard is validated. Automatic Suricata recovery was demonstrated, and detection resumed afterward.

**PARTIAL / OPEN:** Temporary ingress and egress `tc` ICMP mirrors on TrueNAS `vnet5` successfully delivered **4 requests and 4 replies** from a controlled Kali ↔ TrueNAS gateway test to Linux Mint `ens5`, and Suricata logged a matching established bidirectional flow. The earlier [visibility-gap screenshot](evidence/05-br30-traffic-visibility-gap.PNG), [one-way proof](evidence/07-suricata-br30-mirrored-flow.png), and [bidirectional proof](evidence/08-suricata-br30-bidirectional-flow.png) establish this progression. All-protocol/other-VM coverage, `br40` mirror configuration, persistence and a new Wazuh alert for mirrored `br30` traffic remain unverified.

## SOC Migration Regression Checks — 2026-10-17

| ID | Validation | Evidence | Result |
|---|---|---|---|
| V-23 | SOC guest connectivity | Mint ens6, 10.10.20.11/24, reached TrueNAS br20 10.10.20.1 and Ubuntu 10.10.20.10: 4/4 replies each, zero loss | PASS |
| V-24 | Monitoring engine after Mint reboot | Suricata 7.0.3 started two capture threads each on ens4 and ens5; 53,144 rules loaded with zero failures | PASS |
| V-25 | Live capture post-reboot | Fresh ens5 IPv6-ICMP flows and stats in EVE JSON; sampled 696 kernel packets, zero drops and errors | PASS |
| V-26 | Custom alert after network change | Ping via ens4 to 10.10.40.1 succeeded 4/4; EVE logged four SID 1000001 ICMP alerts from 10.10.40.10 at about 12:24 CDT | PASS |
| V-27 | Wazuh manager delivery | SIEM_01 alerts.json recorded corresponding Wazuh rule 86601 level-3 alerts, agent 003 cyberblue-suricata (192.168.1.92), with firedtimes reaching 4 | PASS |
| V-28 | Management network preserved | Mint ens3, 192.168.1.92, reached 192.168.1.1: 4/4 replies, zero loss, average RTT 0.711 ms | PASS |
| V-29 | Scoped mirrored-traffic POC cleanup | Temporary TrueNAS clsact mirroring qdisc removed from vnet5 and absence confirmed before Mint restart | PASS |

**Remaining gaps:** The post-migration alerts have not yet been checked in the Wazuh dashboard. The temporary br30 ICMP mirroring proof of concept has been removed and is not an active monitoring capability. Full inter-zone separation, TrueNAS br20 reboot persistence, and full network visibility remain unverified.

## Threat Hunting Dashboard Confirmation — 2026-10-17

| ID | Validation | Evidence | Result |
|---|---|---|---|
| V-30 | Post-SOC-change Wazuh dashboard visibility | User screenshot of Wazuh Threat Hunting on Oct 17 shows query `rule.id: 86601`, filters `manager.name: siem-01` and `agent.id: 003`, **4 hits** at approximately 12:24 CDT, agent `cyberblue-suricata`, rule level 3, and description `Suricata: Alert - CYBERBLUE TEST - ICMP Echo Request`. | **PASS — dashboard confirmed** |

This completes the validated test path: Linux Mint `ens4` ping -> Suricata SID `1000001` EVE alert -> Wazuh agent `003` -> manager `siem-01` rule `86601` -> dashboard Threat Hunting. Screenshot [09-wazuh-post-soc-migration-alerts.png](evidence/09-wazuh-post-soc-migration-alerts.png) was uploaded to the Project 05 `evidence/` directory and its GitHub file verified. This does not prove full bridge traffic mirroring or inter-zone isolation.
