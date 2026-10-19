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

---

## Module 06 — Staged Network Migration Validation (2026-10-18)

This section validates **guest address cleanup and migration only**, not full zone isolation or TrueNAS host reboot persistence.

| ID | Test | Observed evidence | Status |
|---|---|---|---|
| V-31 | Ubuntu Netplan syntax/effective config | `netplan generate` returned without errors; `netplan get` contained `mgmt0` and `ens4`, not old `lab0` configuration | PASS |
| V-32 | Ubuntu live address/route cleanup | After `netplan try` was accepted, `mgmt0 192.168.1.243/24`, `ens4 10.10.20.10/24`; `lab0` had no IPv4; `10.10.30.0/24` route absent | PASS |
| V-33 | Ubuntu guest reboot persistence | Reconnected over SSH; `ens3` (former `lab0`) had IPv6 link-local only; required `mgmt0`/`ens4` addresses and default/SOC routes remained | PASS |
| V-34 | Windows new Victim IP | `10.10.30.10/24` Preferred; `10.10.30.11` removed; `192.168.1.151/24` Preferred on management NIC | PASS |
| V-35 | Windows Victim routing/connectivity | Direct `10.10.30.0/24` route on `Ethernet`; ping from `10.10.30.10` to `10.10.30.1` 4/4, 0% loss after removing old address | PASS |
| V-36 | Windows management preserved | Default route via `192.168.1.1` on `Ethernet 3`; ping from `192.168.1.151` 4/4, 0% loss | PASS |
| V-37 | Windows guest reboot persistence | Both desired IPs and routes persisted; post-reboot Victim and management gateway pings each 4/4, 0% loss | PASS |
| V-38 | Three-zone isolation and host persistence | Kali migration V-39–V-45, Victim ↔ Attack IPv4 ICMP V-46–V-51, and four SOC-direction IPv4 ICMP checks V-52–V-60 validated in separate controlled windows; SOC policy was rolled back after testing, while the inter-zone protocol matrix, management and IPv6 paths, full capture visibility, and TrueNAS host reboot are still not demonstrated | OPEN — partial isolation proven |

### Module 06 evidence (files verified in GitHub)

- [Ubuntu routes after applied Netplan change](evidence/project06-netplan-routing-validated.png)
- [Ubuntu interfaces/routes after guest reboot](evidence/project06-netplan-post-reboot-validation.png)
- [Windows Victim and management IPs before reboot](evidence/module06-windows-victim-ip-pre-reboot.png)
- [Windows Victim/management IPs and routes after reboot](evidence/module06-windows-victim-post-reboot-validation.png)

**Interpretation:** Ubuntu's old `br30` VirtIO link still exists without a guest IPv4 address. IP persistence is not proof of physical attachment removal, TrueNAS host persistence, passive packet-mirror coverage, or firewall-enforced separation.

### Kali Attack-Zone Migration and Recovery — 2026-10-18

| ID | Test | Observed evidence | Status |
|---|---|---|---|
| V-39 | Kali target bridge and NIC inventory | TrueNAS `br40 10.10.40.1/24` UP; after migration VM `6` device `42` attached to `br40`, management device `16` remained `enp4s0` | PASS |
| V-40 | Fault diagnosis: NIC order/profile mismatch | Post-move `eth0` mapped to Attack MAC but held management IP; `eth1` mapped to management MAC but held Attack IP. Management gateway initially failed 0/4 and SSH was unavailable | OBSERVED FAULT |
| V-41 | SSH and management recovery | Rebound NetworkManager profiles to correct interfaces, activated management via SPICE console; management gateway 4/4 and PuTTY SSH restored | PASS |
| V-42 | Persistent NIC identification | TrueNAS stored management MAC `00:a0:98:5b:5f:c8` on device `16`; Attack MAC `00:a0:98:76:2d:fd` explicitly assigned to device `42`; each NetworkManager profile has matching MAC with no interface-name restriction | PASS |
| V-43 | Attack and management IPv4 routes | `eth0 10.10.40.20/24` on `br40`; `eth1 192.168.1.91/24` management, default via `192.168.1.1 dev eth1`; old `10.10.30.0/24` route absent | PASS |
| V-44 | Attack network reboot persistence | Kali rebooted; expected address and connected route persisted; `ping -I eth0 -c 4 10.10.40.1` returned 4/4, 0% loss, mean RTT 0.280 ms | PASS |
| V-45 | Management network reboot persistence | PuTTY reconnected after reboot; `ping -I eth1 -c 4 192.168.1.1` returned 4/4, 0% loss, mean RTT 0.480 ms | PASS |

**Evidence:** [Kali before final reboot](evidence/module06-kali-attack-network-pre-reboot.png) · [Kali after final reboot](evidence/module06-kali-attack-network-post-reboot.png).

**Interpretation:** The guest-network migration and recovery tests passed. The earlier temporary TrueNAS `tc` bridge mirroring rules are not present as a validated persistent monitoring path. VM network addressing, gateway pings and saved MAC settings alone do not prove firewall isolation between `br20`, `br30` and `br40`, or eliminate alternate management-LAN paths. TrueNAS host bridge reboot persistence and comprehensive IDS capture coverage remain open.

### Module 06 — Victim ↔ Attack IPv4 Isolation Tests (2026-10-19)

| ID | Test | Observed evidence | Status |
|---|---|---|---|
| V-46 | TrueNAS firewall and forwarding baseline | `sudo iptables -S FORWARD`: default policy ACCEPT; `br40 → br30 DROP`, `br30 → br40 DROP` present behind Docker chains. `/proc/sys/net/ipv4/ip_forward` = `1`. `iptables-legacy` showed separate Tailscale `ts-forward` chain, without explicit CyberBlue bridge rules | PASS — rules observed |
| V-47 | Kali Attack-path selection | Default route lookup to Windows `10.10.30.10` selected management `eth1`. Temporary `10.10.30.10/32` route via `10.10.40.1 dev eth0`; lookup verified source `10.10.40.20`, `eth0` | PASS |
| V-48 | Attack → Victim scoped IPv4 ICMP blocking | Kali `ping -I eth0 -c 4 10.10.30.10`: 4 sent, 0 received. TrueNAS `br40 → br30 DROP`: 4 packets / 336 B → 8 packets / 672 B, delta +4 / +336 B. Opposite DROP counter unchanged | PASS — tested traffic dropped |
| V-49 | Kali route cleanup | Removed only temporary `10.10.30.10/32` route; `ip -4 route get 10.10.30.10` returned original management path `192.168.1.1 dev eth1 src 192.168.1.91` | PASS |
| V-50 | Victim → Attack scoped IPv4 ICMP blocking | Windows `Find-NetRoute` first selected management. Temporary `10.10.40.20/32` via Victim `Ethernet` / `10.10.30.1` applied in `ActiveStore`; route lookup confirmed source `10.10.30.10`. Four-probe Windows ping timed out. TrueNAS `br30 → br40 DROP`: 4 / 336 B → 8 / 576 B, delta +4 / +240 B. Opposite DROP counter unchanged | PASS — tested traffic dropped |
| V-51 | Windows route cleanup | Removed temporary `10.10.40.20/32` route. `Find-NetRoute` again selected management `Ethernet 3`, source `192.168.1.151`, default `0.0.0.0/0` next hop `192.168.1.1` | PASS |

**Evidence source:** Operator-supplied TrueNAS, Kali and Windows command outputs during guided validation on October 19, 2026. No screenshots for these tests have yet been uploaded or verified in GitHub. The original counters of 4 / 336 B in both directions may include historical tests; the new **four-packet increments observed during the controlled tests** are the current evidence.

**Interpretation:** Bidirectional **routed IPv4 ICMP** deny behavior for Kali Attack `10.10.40.20` ↔ Windows Victim `10.10.30.10` through TrueNAS `br40` and `br30` is confirmed. Results do not establish IPv6 policies, other protocols, SOC bridge isolation, a complete management-LAN bypass assessment, TrueNAS host reboot persistence, or comprehensive Suricata mirroring. **V-38 remains OPEN for full isolation.**

## Module 06 — Three-Zone Scoped IPv4 ICMP Firewall Trial (2026-10-19)

**Status: PASS — six of six tested directional routed IPv4 ICMP paths were blocked. NOT a production/persistent three-zone isolation PASS.** Victim ↔ Attack checks V-48/V-50 used the existing baseline rules; the four SOC directions below used a temporary six-rule candidate. The candidate was rolled back after each test window. The persistent POSTINIT startup script remains the original two-rule version.

| ID | Validation | Operator-supplied evidence | Status |
|---|---|---|---|
| V-52 | SOC firewall change safeguards | Original `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh` copied to `cyberblue-firewall.sh.pre-br20-20261019.bak`; byte comparison verified. Candidate and rollback shell syntax checks passed. TrueNAS POSTINIT startup script was not changed. | PASS — safeguards verified |
| V-53 | Temporary candidate rule order | `cyberblue-firewall-candidate.sh` applied during three rollback-protected windows. `iptables -L FORWARD -n -v --line-numbers` showed six directional inter-zone DROP entries in rules 1–6 ahead of Docker-related forwarding chains (with a separate `iptables-legacy` warning). | PASS — observed live in test windows |
| V-54 | SOC → Victim ICMP | Ubuntu `10.10.20.10` temporary `10.10.30.10/32 via 10.10.20.1 dev ens4` route verified; 4 pings, 0 replies; TrueNAS `br20 → br30` DROP rule 6 recorded **4 packets / 336 B**. | PASS — tested flow dropped |
| V-55 | Ubuntu SOC → Victim cleanup | Temporary destination-specific route removed; `ip -4 route get 10.10.30.10` again selected `192.168.1.1 dev mgmt0 src 192.168.1.243`. First manual SOC rollback restored the two-rule baseline; first rollback timer was confirmed inactive. | PASS |
| V-56 | SOC → Attack ICMP | Ubuntu `10.10.40.20/32 via 10.10.20.1 dev ens4` selected source `10.10.20.10`; 4 pings, 0 replies; TrueNAS `br20 → br40` DROP rule 4 recorded **4 packets / 336 B**. Route removed; normal `mgmt0` path verified. | PASS — tested flow dropped |
| V-57 | Attack → SOC ICMP | Kali `10.10.20.10/32 via 10.10.40.1 dev eth0` selected source `10.10.40.20`; 4 pings, 0 replies; TrueNAS `br40 → br20` DROP rule 3 recorded **4 packets / 336 B**. Route removed; normal management `eth1` next hop `192.168.1.1` verified. | PASS — tested flow dropped |
| V-58 | Second rollback and timer cleanup | Manual rollback removed four SOC DROP entries while preserving Victim ↔ Attack rules and Docker chains. Second transient timer `cyberblue-br20-rollback2.timer` was stopped and verified `inactive`. | PASS |
| V-59 | Victim → SOC ICMP | Windows `10.10.20.10/32` ActiveStore route via `10.10.30.1` on `Ethernet` verified with `Find-NetRoute` (source `10.10.30.10`); four `ping -4 -S 10.10.30.10` probes, zero replies. TrueNAS `br30 → br20` DROP rule 5 recorded **4 packets / 240 B**. | PASS — tested flow dropped |
| V-60 | Final route and firewall cleanup | Windows test route removed; `Find-NetRoute` again chose management `Ethernet 3`, `192.168.1.151`, via `192.168.1.1`. Final manual rollback verified `iptables -S FORWARD` retained only the two baseline `br40 ↔ br30` DROP rules plus Docker chains; third timer `cyberblue-br20-rollback3.timer` stopped and verified `inactive`. | PASS — baseline restored |

**Directional test coverage (all PASS for the specific ICMP probes):** Attack → Victim, Victim → Attack (V-48 and V-50), SOC → Victim (V-54), SOC → Attack (V-56), Attack → SOC (V-57), Victim → SOC (V-59). Counter measurements were taken in their respective test windows; rule counters restarted when the candidate script removed and reinserted rules. Do not add these readings together as one uninterrupted counter series.

**Live end state at cleanup:** TrueNAS default `FORWARD ACCEPT`; only `br40 → br30 DROP` and `br30 → br40 DROP` retained ahead of existing Docker chains. All temporary SOC inter-zone rules removed, and all three transient rollback timers verified inactive. The **approved full three-zone default-deny policy is not yet persistently implemented**.

**Open:** Full TCP/UDP and other protocol paths; IPv6; management-LAN alternative routes and dual-homed guest bypasses; bridge Layer-2 and TrueNAS host traffic; deliberate authorized exceptions; TrueNAS host reboot persistence of `br20` and firewall; and Suricata mirror coverage. V-38 stays **OPEN**. Evidence is the operator's shell/PowerShell output in the guided validation; no new screenshots are claimed.

## Module 06 — Approved Management-LAN SSH Access (2026-10-19)

**Policy decision:** User explicitly approved keeping Windows 11 Victim → Kali TCP/22 connectivity over the shared management LAN. This is an *administrative management path*, not a blanket Victim `br30` → Attack `br40` exception.

| ID | Check | Observed evidence | Status |
|---|---|---|---|
| V-61 | Windows management reachability to Kali SSH | Windows `Test-NetConnection -ComputerName 192.168.1.91 -Port 22` returned `SourceAddress 192.168.1.151`, `InterfaceAlias Ethernet 3`, `RemotePort 22`, `TcpTestSucceeded True`. | PASS — TCP connection only |
| V-62 | Management exception authorization | Operator chose to **keep** Windows `192.168.1.151` → Kali `192.168.1.91:22/tcp` permitted for lab administration; no firewall change requested or made. | APPROVED — documented scope |

**Limits:** No SSH authentication/login was attempted. No claim is made that all management traffic is safe or restricted to this flow. The existing same-LAN TCP path is not filtered by the TrueNAS `br30` ↔ `br40` `FORWARD` rules; a future management ACL should explicitly account for this approved flow without broadly granting host-to-host access. Other guest-management paths and full zone isolation remain OPEN.

[Architecture decision](../../docs/architecture.md#16-approved-management-lan-ssh-exception-2026-10-19)
