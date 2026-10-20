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

## Module 06 — Dedicated CYBERBLUE_ZONES Chain Test (2026-10-20)

**Scope:** A separately prepared, temporary `iptables-nft` chain was activated and tested for three directed routed IPv4 ICMP flows. The original two-rule Victim ↔ Attack baseline was retained throughout the trial. **This was not a permanent three-zone deployment.**

| ID | Validation | Operator-supplied evidence | Status |
|---|---|---|---|
| V-63 | Backend, script and existing baseline review | `iptables v1.8.9 (nf_tables)`; saved six-rule chain candidate and companion rollback script both passed `sh -n` and content review. Before activation, `iptables -S FORWARD` showed only the original `br40 → br30` and `br30 → br40` DROP rules ahead of Docker forwarding entries. An initial `iptables -S CYBERBLUE_ZONES` reported an ambiguous compatibility error; `nft -a list chain ip filter CYBERBLUE_ZONES` returned no such chain and `iptables-save -t filter` found no chain or jump. | PASS — preflight reviewed |
| V-64 | Timed rollback safeguard | `systemd-run --unit=cyberblue-chain-rollback-20261020 --on-active=15m /bin/sh /mnt/Pool1/CyberBlue/cyberblue-firewall-chain-rollback.sh` created a transient timer. `systemctl status` showed `active (waiting)` and a trigger at **02:51:23 PDT** on October 20. | PASS — safeguard armed |
| V-65 | Dedicated-chain installation and order | Executing `cyberblue-firewall-chain-candidate.sh` returned `CANDIDATE APPLIED`. `iptables -L FORWARD -n -v --line-numbers` showed jump to `CYBERBLUE_ZONES` at rule 1, original Victim ↔ Attack DROP rules 2–3, and Docker rules 4–9. Dedicated chain showed exactly six directional DROP rules: `br20→br30`, `br30→br20`, `br20→br40`, `br40→br20`, `br30→br40`, `br40→br30`, each initially 0 packets. | PASS — temporary rules observed |
| V-66 | Attack → SOC routed IPv4 ICMP | Kali `10.10.40.20` used a temporary `10.10.20.10/32 via 10.10.40.1 dev eth0` route with `EXIT` cleanup trap; `ping -4 -n -I 10.10.40.20 -c 4 -W 1 10.10.20.10` returned 4 sent, 0 received. Dedicated chain **rule 4**, `br40→br20`, increased 0 → **4 packets / 336 B**. | PASS — tested packets dropped |
| V-67 | SOC → Attack routed IPv4 ICMP | Ubuntu `10.10.20.10` used temporary `10.10.40.20/32 via 10.10.20.1 dev ens4` route with cleanup trap; four ICMP probes had 0 replies. Chain **rule 3**, `br20→br40`, increased 0 → **4 packets / 336 B**. | PASS — tested packets dropped |
| V-68 | SOC → Victim routed IPv4 ICMP | Ubuntu `10.10.20.10` used temporary `10.10.30.10/32 via 10.10.20.1 dev ens4` route with cleanup trap; four ICMP probes had 0 replies. Chain **rule 1**, `br20→br30`, increased 0 → **4 packets / 336 B**. | PASS — tested packets dropped |
| V-69 | Manual rollback, live firewall and timer state | `cyberblue-firewall-chain-rollback.sh` returned `CYBERBLUE CHAIN ROLLBACK COMPLETE`. `iptables -S FORWARD` then showed **no** `CYBERBLUE_ZONES` jump, original bidirectional Victim ↔ Attack DROP rules, Docker rules intact, and `-P FORWARD ACCEPT`. `systemctl show cyberblue-chain-rollback-20261020.timer` returned `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`. The inactive chain itself is not claimed deleted. | PASS — baseline and timer verified |
| V-70 | Independent Linux guest route-cleanup readback | On Ubuntu `cbubuntu01`, `ip -4 route show exact 10.10.40.20/32` and `ip -4 route show exact 10.10.30.10/32` both returned no entries. On Kali (`pete@BlueSOC`), `ip -4 route show exact 10.10.20.10/32` returned no entries. Kali `ip -4 route get 10.10.20.10` selected `via 192.168.1.1 dev eth1 src 192.168.1.91`. Ubuntu `ip -4 route get 10.10.40.20` and `ip -4 route get 10.10.30.10` subsequently both selected `via 192.168.1.1 dev mgmt0 src 192.168.1.243`. | PASS — named temporary routes absent; Kali and Ubuntu normal routing confirmed |
| V-71 | Dedicated chain retained after rollback | After the `FORWARD` jump had been removed and baseline verified in V-69, a separate read-only `sudo iptables -L CYBERBLUE_ZONES -n -v --line-numbers` still listed six DROP rules. Counters: rule 1 `br20→br30` 4 packets / 336 B; rule 2 `br30→br20` 0; rule 3 `br20→br40` 4 / 336 B; rule 4 `br40→br20` 4 / 336 B; rules 5–6 Victim ↔ Attack each 0. These are retained historical test counters, **not** evidence that the detached chain is now enforcing policy. | PASS — saved in-kernel chain observed, detached status based on V-69 |

**Interpretation and limitations:** All three explicitly tested flows were dropped under the temporary dedicated chain; each corresponding rule counter increased by exactly four. The other three directional chain rules were **installed but not traffic-tested in this session** (earlier independent baseline/candidate trials cover those directions, not this particular chain trial). Guest route commands contained an `EXIT` cleanup trap. An independent subsequent readback confirmed the two tested Ubuntu /32 host routes and the tested Kali /32 host route were absent; Kali's route to the SOC target again selected the management interface. See V-70. Ubuntu's post-test route lookup to Kali's Attack address also selected the normal management route: `via 192.168.1.1 dev mgmt0 src 192.168.1.243`. No new evidence screenshots were uploaded for this test.

**End state:** The temporary jump was removed. The original permanent startup script `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh` remains unchanged, and permanent three-zone enforcement is **OPEN**. No claim is made for TCP/UDP, IPv6, same-LAN management bypasses, Layer-2/host paths, TrueNAS reboot persistence or comprehensive IDS mirroring. The approved management SSH path (V-61–V-62) remains a separate management-LAN exception.

## Module 06 — Hardened Persistent-Startup V2 Controlled Trial (2026-10-20)

**Scope:** Reversible live testing of the *candidate* `/mnt/Pool1/CyberBlue/cyberblue-firewall-persistent-v2.sh` against the already-populated detached `CYBERBLUE_ZONES` chain. Despite the candidate name, this was **not** a change to the registered TrueNAS POSTINIT job or a TrueNAS reboot/persistence test. No new guest ICMP traffic was sent during the V2 trial; network packet-drop tests V-66–V-68 concern the earlier dedicated-chain trial.

| ID | Validation | Operator-supplied evidence | Status |
|---|---|---|---|
| V-72 | V2 script preparation and hardening | First persistent candidate passed `sh -n` and was content-reviewed. A V2 script was created and its 1–83 lines inspected; a temporary heredoc paste error caused an earlier syntax failure, after which the saved V2 contents and syntax were verified. Backup `cyberblue-firewall-persistent-v2.pre-hardening.bak` was created before editing. Hardened V2 avoids flushing populated detached chains, reuses a matching six-rule chain, rejects unexpected detached contents or references, checks exact active-chain contents and jump ordering. `V2 HARDENED SYNTAX PASS` observed. | PASS — saved candidate reviewed, not yet a production safety guarantee |
| V-73 | Active-chain preflight and timed V2 activation | `iptables -S CYBERBLUE_ZONES` matched the exact six-rule V2 expectation. A transient `cyberblue-v2-rollback-20261020.timer` with 15-minute delay was confirmed `active (waiting)`, due **03:36:47 PDT**. Running V2 with timer-active guard printed `CYBERBLUE CHAIN ACTIVATED`. Readback `iptables -S FORWARD` showed **one** `CYBERBLUE_ZONES` jump at position 1, existing `br40→br30` and `br30→br40` DROPs at positions 2–3, Docker entries afterwards, and default `FORWARD ACCEPT`. | PASS — temporary activation and order |
| V-74 | V2 repeat-execution / idempotency check | Running V2 a second time with the timer guard printed `CYBERBLUE CHAIN ALREADY ACTIVE AND VERIFIED` (with expected iptables-legacy warnings). A second `iptables -S FORWARD` showed **exactly one** `CYBERBLUE_ZONES` jump, original two Victim ↔ Attack DROPs and Docker forwarding rules still intact. This verifies the tested same-state repeat path; it does not establish robustness to concurrent firewall modifications or future service reordering. | PASS — no duplicate jump after second run |
| V-75 | V2 manual rollback and forwarding baseline | `cyberblue-firewall-chain-rollback.sh` returned `CYBERBLUE CHAIN ROLLBACK COMPLETE`. Readback `iptables -S FORWARD` showed **no** `CYBERBLUE_ZONES` jump; original `br40→br30` and `br30→br40` DROPs and Docker entries preserved; `-P FORWARD ACCEPT`. | PASS — original live baseline restored |
| V-76 | V2 rollback timer cleanup | After the manual rollback and stop command, `systemctl show cyberblue-v2-rollback-20261020.timer -p LoadState -p ActiveState -p SubState --no-pager` returned `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`. | PASS — timer inactive |

**Outcome:** V2 correctly activated the prepared six-rule chain, recognized its already-active configuration on a second run, and was safely rolled back **in the tested conditions**. The two original Victim ↔ Attack direct DROP rules remain the deployed baseline; six-rule inter-zone default deny has **not** been installed as the TrueNAS startup configuration. V2 has not been tested during host reboot, Docker/Tailscale restarts, unexpected-rule failure conditions, or concurrently changing rule order. Full three-zone policy and non-IPv4/IP-forwarded paths remain OPEN.

[Build notes](build-notes.md#module-06--hardened-persistent-v2-reversible-test-2026-10-20).

## Module 06 — Docker-Compatible DOCKER-USER Trial (2026-10-20)

**Scope:** Additional operator-supplied readbacks and a *temporary*, rollback-protected IPv4 `DOCKER-USER` jump to the already-populated `CYBERBLUE_ZONES` chain. **No new guest-packet probes, Docker restart, TrueNAS reboot, or permanent deployment were performed.** Existing TrueNAS POSTINIT registration continued to point at `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh` (enabled, SCRIPT, POSTINIT, 30-second timeout).

| ID | Verification | Operator-supplied evidence | Status |
|---|---|---|---|
| V-77 | Persistent-script baseline and backup | `initshutdownscript.query` showed enabled POSTINIT job ID 1 targeting original two-rule `cyberblue-firewall.sh`. `cmp` returned `ORIGINAL BACKUP MATCHES` for `cyberblue-firewall.sh.pre-br20-20261019.bak`. | PASS — existing startup target and backup verified |
| V-78 | V2 script bounded lock waits | `iptables --help` supported `-w [seconds]`. Operator copied V2 to `cyberblue-firewall-persistent-v2.pre-wait.bak` and received `V2 PRE-WAIT BACKUP VERIFIED`. Targeted edit reported 12 replacements of `"$IPT" -w` with `"$IPT" -w 2`; `sh -n` returned `V2 TIMEOUT SYNTAX PASS`. Read-only `grep -nF` displayed exactly 12 updated calls on lines 17,19,25,31,42,48,49,53,56,71,75,81. | PASS — saved candidate syntax and occurrences; no whole-script 30s timing guarantee |
| V-79 | TrueNAS / Docker startup sequencing | Current-boot journal: `ix-postinit.service` started **2026-10-09 12:11:57 PDT**, finished **12:11:58**; Docker began **12:12:01**, was started **12:12:09**. `ix-postinit.service` invokes `midclt call -job initshutdownscript.execute_init_tasks POSTINIT`; no direct Docker ordering dependency shown. Docker loaded/active, service unit disabled but `TriggeredBy=docker.socket`. `docker.service` override: `ExecStartPost=/bin/sh -c "iptables -P FORWARD ACCEPT && ip6tables -P FORWARD ACCEPT"`. | OBSERVED — POSTINIT finished before Docker on this boot; does not alone prove script success or all future boots |
| V-80 | Networking boundaries and alternate rulesets | IPv4 nft-backed `FORWARD` baseline: policy ACCEPT, original `br40→br30` and `br30→br40` DROPs before Docker jumps. Legacy `FORWARD` has `-j ts-forward` with Tailscale MARK/ACCEPT/DROP/ACCEPT entries; not changed. IPv6 forwarding `net.ipv6.conf.all.forwarding=0` and `br20`, `br30`, `br40` each `0`; `ip6tables -S FORWARD` showed ACCEPT and Docker-USER only. | PASS — current state inspected; **not** proof of comprehensive IPv6 or management-LAN isolation |
| V-81 | Bridge membership | `/sys/class/net/*/brif` showed SOC `br20`: `vnet12,vnet9`; Victim `br30`: `vnet11,vnet13,vnet8`; Attack `br40`: `vnet10,vnet15`. No overlapping bridge-port names; per-guest mapping and other paths not independently verified here. | PASS — distinct bridge membership observed |
| V-82 | Docker compatibility preflight | Initial `DOCKER-USER` exactly `-N DOCKER-USER` / `-A DOCKER-USER -j RETURN`; detached `CYBERBLUE_ZONES` had exactly six directed `br20/br30/br40` DROP rules. Created `cyberblue-docker-user-rollback.sh` and `cyberblue-docker-user-candidate.sh`, each passed `sh -n` and complete operator `cat` review. Candidate matches six rules and exact Docker baseline, refuses existing CyberBlue references, inserts one `DOCKER-USER` jump at rule 1. | PASS — static checks, not failure/restart testing |
| V-83 | Rollback timer and live temporary activation | Created 15-minute transient `cyberblue-docker-user-rollback-20261020.timer`; `systemctl status` confirmed **active (waiting)**, due **2026-10-20 04:19:22 PDT**. Timer-active guarded script output `CYBERBLUE DOCKER-USER CHAIN ACTIVATED`. `iptables -S DOCKER-USER`: `CYBERBLUE_ZONES` jump before `RETURN`; `iptables -S FORWARD`: original two DROP rules then `DOCKER-USER` then existing Docker rules. | PASS — transient activation and rule placement |
| V-84 | DROP counters during Docker-compatible window | `iptables -nvx -L CYBERBLUE_ZONES` showed `br20→br30` 4/336B, `br20→br40` 4/336B, `br40→br20` 4/336B; other three rules 0. These were the **previously accumulated dedicated-chain counts**; none were reset or independently shown to increment in this window. | INSPECTED — **not** fresh packet-drop evidence |
| V-85 | Manual rollback and automatic-timer cleanup | `cyberblue-docker-user-rollback.sh` printed `CYBERBLUE DOCKER-USER ROLLBACK COMPLETE`. Independent `iptables -S DOCKER-USER` readback returned only chain declaration and `-j RETURN` (no CyberBlue jump). Timer stopped, then `systemctl show` reported `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`. | PASS — Docker user-chain baseline and timer state verified |

**End state and limitations:** The temporary `DOCKER-USER` attachment was removed. Existing direct Victim ↔ Attack IPv4 DROP rules remain the documented deployed baseline, and the original POSTINIT script was not replaced. Unlike the earlier three-direction dedicated-chain trial (V-66–V-68), this window contained **no new isolated guest test traffic**. Durable insertion after Docker starts/restarts, reboot recovery, fresh all-direction traffic validation, management-LAN bypass boundaries, IPv6, Layer 2, host traffic, and Suricata mirroring remain **OPEN**. Preserve the separately approved Windows-management-IP → Kali-management-IP TCP/22 path.

[Implementation notes](build-notes.md#module-06--docker-startup-dependency-review-and-docker-user-trial-2026-10-20).
