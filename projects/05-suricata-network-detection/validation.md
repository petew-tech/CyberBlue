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

**PARTIAL / OPEN — historical October 16 checkpoint:** Temporary ingress and egress `tc` ICMP mirrors on TrueNAS `vnet5` successfully delivered **4 requests and 4 replies** from a controlled Kali ↔ TrueNAS gateway test to Linux Mint `ens5`, and Suricata logged a matching established bidirectional flow. The earlier [visibility-gap screenshot](evidence/05-br30-traffic-visibility-gap.PNG), [one-way proof](evidence/07-suricata-br30-mirrored-flow.png), and [bidirectional proof](evidence/08-suricata-br30-bidirectional-flow.png) establish this progression. **At that time**, all-protocol/other-VM coverage, `br40` mirror configuration, persistent monitoring, and any new Wazuh alert for mirrored `br30` traffic **had not been verified**. **Subsequent October 20 result:** V-112–V-116 established four Suricata SID `1000002` alerts from **Windows Victim** `10.10.30.10 → 10.10.30.1` over a separate **temporary one-way** `vnet13 → vnet11` mirror, all four ingested as Wazuh rule `86601` alerts, and at least one matching SID confirmed in the Threat Hunting dashboard. V-117–V-118 confirmed timer execution and final `noqueue` state with no `clsact`. **Still OPEN:** persistent monitoring, other-host/other-protocol or bridge-wide coverage, and continuous visibility.

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

**Remaining gaps at the V-23–V-29 checkpoint (2026-10-17):** Wazuh dashboard verification of the post-migration alerts was **pending at this point**; it was subsequently completed the same day under **V-30 — PASS** with [dashboard evidence](evidence/09-wazuh-post-soc-migration-alerts.png). The temporary `br30` ICMP mirroring proof of concept had been removed and was not an active monitoring capability. Broader inter-zone isolation, TrueNAS `br20` host-reboot persistence, and full network visibility were not established by these tests; later scoped Project 05 detection results do not establish continuous packet mirroring.

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

## Module 06 — Fresh Six-Direction IPv4 ICMP Test via DOCKER-USER (2026-10-20)

**Scope:** A *second*, separate 30-minute rollback-protected temporary activation of `DOCKER-USER → CYBERBLUE_ZONES`. Six directional routed IPv4 **ICMP** tests each transmitted four echo requests with zero received and a corresponding +4 DROP-rule counter change. **Only the four SOC-related tests matched `CYBERBLUE_ZONES`; Victim ↔ Attack matched the original direct `FORWARD` DROP rules which precede the jump.** No new TCP/UDP, IPv6, reboot or Docker-restart tests occurred.

| ID | Verification | Operator-supplied evidence | Status |
|---|---|---|---|
| V-86 | Safety timer, activation and counter baseline | 30-minute transient `cyberblue-docker-traffic-rollback-20261020.timer` confirmed `active (waiting)`, due **2026-10-20 04:49:27 PDT**. Timer-guarded candidate output: `CYBERBLUE DOCKER-USER CHAIN ACTIVATED`. `iptables -S DOCKER-USER` confirmed `-j CYBERBLUE_ZONES` before `RETURN`. Initial `CYBERBLUE_ZONES` counters: SOC→Victim 4/336B; Victim→SOC 0; SOC→Attack 4/336B; Attack→SOC 4/336B; Victim→Attack 0; Attack→Victim 0. | PASS — activation and starting counters |
| V-87 | SOC → Attack fresh routed ICMP | Ubuntu `cbubuntu01`: `ens4 10.10.20.10/24`, `mgmt0 192.168.1.243/24`; temporarily added `10.10.40.20/32 via 10.10.20.1 dev ens4`, ping source `10.10.20.10`, **4 transmitted / 0 received**. `CYBERBLUE_ZONES br20→br40` increased **4→8 packets, 336→672 B**. Route `ip -4 route show exact 10.10.40.20/32` independently empty after command. An earlier attempt inadvertently run on the TrueNAS host failed at nonexistent `ens4`; it did not send the intended probes. | PASS — four new packets dropped and route cleaned |
| V-88 | Attack → SOC fresh routed ICMP | Kali `eth0 10.10.40.20/24` and management `eth1 192.168.1.91/24`; temporarily added `10.10.20.10/32 via 10.10.40.1 dev eth0`. Ping source `10.10.40.20`, **4 transmitted / 0 received**. `CYBERBLUE_ZONES br40→br20` **4→8 packets, 336→672 B**. Kali route readback for `10.10.20.10/32` empty. | PASS |
| V-89 | SOC → Victim fresh routed ICMP | Ubuntu temporarily added `10.10.30.10/32 via 10.10.20.1 dev ens4`, ping from `10.10.20.10`: **4 transmitted / 0 received**. `CYBERBLUE_ZONES br20→br30` **4→8 packets, 336→672 B**. Exact Ubuntu /32 route readback empty. | PASS |
| V-90 | Victim → SOC fresh routed ICMP | Windows Victim `Ethernet`, interface index **3**, `10.10.30.10/24`. No preexisting `10.10.20.10/32` route. A `try/finally` PowerShell test temporarily added the ActiveStore route via `10.10.30.1`, ping `-S 10.10.30.10 -n 4`: **4 sent / 0 received**, `TEMP ROUTE REMOVED`. `CYBERBLUE_ZONES br30→br20` **0→4 packets, 0→240 B**. Later `Get-NetRoute` exact /32 returned no output. | PASS |
| V-91 | Original direct rules baseline for remaining directions | `iptables -nvx -L FORWARD` showed original direct `br40→br30` and `br30→br40` DROP rules **at positions 1–2**, each with **0 packets / 0 B**, ahead of `DOCKER-USER` and other Docker entries. This ordering means remaining probes would not exercise `CYBERBLUE_ZONES` for those two directions. | PASS — direct-rule baseline |
| V-92 | Attack → Victim original direct-rule ICMP | Kali had no preexisting `10.10.30.10/32` route; temporarily added route via `10.10.40.1 dev eth0`. Ping from `10.10.40.20`: **4 transmitted / 0 received**. Original `FORWARD br40→br30` DROP **0→4 packets, 0→336 B**. Kali exact /32 route readback empty afterward. | PASS — original direct rule, not Docker-chain drop |
| V-93 | Victim → Attack original direct-rule ICMP | Windows had no preexisting `10.10.40.20/32` route; temporarily added ActiveStore /32 via `10.10.30.1` on interface index 3. Ping `-S 10.10.30.10 -n 4`: **4 sent / 0 received**, `TEMP ROUTE REMOVED`. Original `FORWARD br30→br40` DROP **0→4 packets, 0→240 B**. Independent Windows exact /32 route query returned no output. | PASS — original direct rule, not Docker-chain drop |
| V-94 | Manual firewall rollback and timer stop | `cyberblue-docker-user-rollback.sh` reported `CYBERBLUE DOCKER-USER ROLLBACK COMPLETE`. Independent `iptables -S DOCKER-USER` returned **only** `-N DOCKER-USER` and `-A DOCKER-USER -j RETURN`. Timer stop command returned `DOCKER TRAFFIC ROLLBACK TIMER STOPPED`; `systemctl show` then reported `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`. | PASS — DOCKER-USER baseline restored; timer inactive |
| V-95 | Final Windows temporary route readback | After V-94, Windows `Get-NetRoute -DestinationPrefix "10.10.40.20/32" -AddressFamily IPv4 -ErrorAction SilentlyContinue` produced no output. | PASS — named Windows route absent |

| V-97 | Approved Windows → Kali management TCP/22 reachability after rollback | Following the six-direction temporary IPv4 ICMP trial, verified manual rollback, and independent original `FORWARD` baseline readback, Windows 11 `Test-NetConnection -ComputerName 192.168.1.91 -Port 22 -InformationLevel Detailed` reported `InterfaceAlias: Ethernet 3`, `SourceAddress: 192.168.1.151`, `RemoteAddress: 192.168.1.91`, `RemotePort: 22`, `NetRoute (NextHop): 0.0.0.0`, `TcpTestSucceeded: True`. The connection followed the shared management subnet, outside the scoped inter-zone `FORWARD` rules. **No SSH authentication test and no new firewall exception rule.** | PASS — post-test approved TCP reachability preserved |

**Interpretation:** Six of six **tested routed IPv4 ICMP directions** produced matching four-packet DROP counter increases in this *combined* temporary configuration: four matched `DOCKER-USER → CYBERBLUE_ZONES`, two matched the original direct `FORWARD` rules. This does not prove all six would match the dedicated chain if the direct rules were removed, and does not validate other protocols or bypasses.

| V-96 | Final independent `FORWARD` readback after rollback | On TrueNAS, `sudo /usr/sbin/iptables -S FORWARD` returned policy `ACCEPT`, original `br40→br30` and `br30→br40` direct DROP rules at the top, followed by `DOCKER-USER`, `DOCKER-ISOLATION-STAGE-1` and unchanged displayed Docker forwarding rules. **No direct `FORWARD→CYBERBLUE_ZONES` jump** appeared. Earlier independent `DOCKER-USER` readback confirmed the temporary jump absent and `RETURN` restored. | PASS — original IPv4 forwarding baseline read back after rollback |

**Current documented end state:** The temporary `DOCKER-USER` jump was removed and its chain restored to `RETURN`; the rollback timer is independently inactive. **A subsequent full `FORWARD` readback (V-96) confirmed** the original two Victim ↔ Attack DROP rules, Docker forwarding entries, and no direct CyberBlue jump. No permanent change to the original two-rule TrueNAS POSTINIT script was made. Docker lifecycle/reboot persistence, management-LAN bypass protection, IPv6, guest-to-guest L2/host paths, and sensor mirroring remain OPEN.

[Implementation notes](build-notes.md#module-06--fresh-six-direction-ipv4-icmp-validation-and-rollback-2026-10-20).

## Module 06 — Docker Lifecycle Read-Only Discovery and V3 Preflight (2026-10-20)

| ID | Verification | Evidence / qualification | Status |
|---|---|---|---|
| V-98 | Lifecycle candidate backups and inspections | Operator verified byte-for-byte backups of `cyberblue-docker-user-candidate.sh`, `cyberblue-firewall-lifecycle-candidate.sh`, and `cyberblue-firewall-lifecycle-v2.sh`. Lifecycle V1, V2 and V3 saved script contents were read back. V1/V2/V3 passed `/bin/sh -n` syntax checks. The first two versions contain identified partial-failure and/or Docker-race limitations; V3 deliberately requires an existing complete chain and defaults to `--check`. | PASS — file and syntax inspections only |
| V-99 | Existing dedicated chain and detached status | `iptables -S CYBERBLUE_ZONES` returned the six expected directional DROP rules, and `iptables -S | grep -F -- '-j CYBERBLUE_ZONES'` returned **no matching rule**. The only other output was the pre-existing `iptables-legacy` warning; that separate backend was not edited. | PASS — chain exists, unreferenced in current nft-backed IPv4 filter ruleset |
| V-100 | Tool support and V3 preflight | `iptables` and `iptables-restore` both **v1.8.9 (nf_tables)**. `iptables-restore --help` offered `--test`, `--noflush`, `--table`; no claim of transaction atomicity. `/usr/bin/flock` present, util-linux **2.38.1**. V3 `--check` printed `CYBERBLUE V3 PREFLIGHT PASS - CHAIN DETACHED` after reading live Docker/iptables configuration. Its check mode opens a lockfile but does not change firewall rules. | PASS — scoped read-only preflight; no live V3 activation |
| V-101 | Docker daemon and socket configuration | `docker.service` `active` but `UnitFileState=disabled`, `TriggeredBy=docker.socket`, `Restart=always`, `Type=notify`, `ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock`. `docker.socket` also `active` / disabled, listens at `/run/docker.sock (Stream)`, triggers `docker.service`. Systemd dependencies/order inspected. Existing `/etc/systemd/system/docker.service.d/override.conf` runs `iptables -P FORWARD ACCEPT && ip6tables -P FORWARD ACCEPT` in `ExecStartPost`, last displayed successful execution Oct 9 12:12:09 PDT. | OBSERVED — restart/lifecycle dependency; no Docker restart induced |
| V-102 | TrueNAS POSTINIT unit and ordering limitation | `systemctl cat ix-postinit.service` showed `Type=oneshot`, `RemainAfterExit=yes`, `After=multi-user.target`, `ExecStart=midclt call -job initshutdownscript.execute_init_tasks POSTINIT`, `WantedBy=truenas.target`, output directed to null. There is **no declared `After=docker.service`**. Previously observed Oct 9 boot: POSTINIT finished before Docker initialized. | OBSERVED — no guaranteed POSTINIT-after-Docker or on-Docker-restart execution |

**Design status:** No Docker service, socket, POSTINIT service or permanent TrueNAS startup integration was changed in this investigation. V3 remains a tested **preflight-only** candidate; `--activate` has not been run and requires a separately verified rollback timer. A separate CyberBlue lifecycle reconciliation mechanism is a **proposal**, not deployed policy. `After=docker.service` alone does not ensure re-application after every Docker restart; do not claim restart resilience until explicitly implemented and failure-tested. IPv4 ICMP results remain scoped to V-86–V-95 and the previously restored two-rule production baseline.

[Build notes](build-notes.md#module-06--docker-lifecycle-architecture-review-2026-10-20).

## Project 05 — Sensor Runtime and Windows Unicast Visibility (2026-10-20)

| ID | Verification | Evidence / qualification | Status |
|---|---|---|---|
| V-103 | Mint Suricata/Wazuh runtime and dual-interface capture | On Linux Mint, `systemctl is-active suricata wazuh-agent` returned `active` twice. `ip -br addr show` returned management `ens3 192.168.1.92/24`, Attack `ens4 10.10.40.10/24`, passive Victim `ens5` UP with IPv6 link-local only, SOC `ens6 10.10.20.11/24`. `suricata.yaml` AF_PACKET config listed `ens4` cluster ID 99 and `ens5` cluster ID 100. `systemctl show suricata` showed `active`, PID 1056, `--af-packet`; `ps -T` displayed two `W#..-ens4` and two `W#..-ens5` workers. In the most recent 1,000 lines of EVE JSON, 55 records had `in_iface=ens4`, 240 had `in_iface=ens5`. | PASS — running workers and logged activity, **not** full bridge-wide visibility |
| V-104 | Recent Victim broadcast/multicast vs unicast visibility | The latest `ens5` EVE example at 2026-10-20 11:33:11 -0500 was ICMPv6 MLDv2 multicast from the sensor; Windows `10.10.30.10` multicast UDP 5355/5353/1900 records included 2026-10-20 11:10:54 -0500. A search for source `10.10.30.10` to `10.10.30.1` returned no matching historic `ens5` EVE records. A live Mint `sudo timeout 180 tcpdump -ni ens5 -c 4 'icmp and src host 10.10.30.10 and dst host 10.10.30.1'` was reported listening before Windows `ping.exe -4 -S 10.10.30.10 -n 4 10.10.30.1` returned 4 replies / 0% loss. Mint tcpdump subsequently displayed **0 packets captured, 0 received by filter, 0 dropped by kernel**. A previous capture had been interrupted *before* the coordinated ping and is not used as evidence. | **VISIBILITY GAP** — no matching unicast observed on Mint `ens5` during the coordinated test; absence from this filter is not proof against all possible paths |

**Scope:** Successful Windows-to-gateway unicast and no matching passive Mint capture are consistent with the absence of a persistent bridge mirroring path. The October 16 narrowly scoped `tc` mirroring proof-of-concept was removed October 17. The result does not imply Suricata has stopped working, that packet mirroring is impossible, or that a general unicast capture policy has been implemented. Next step: inspect current TrueNAS Victim bridge member interfaces read-only before deciding how to safely mirror selected traffic to the Mint sensor.

[Build notes](build-notes.md#project-05--coordinated-victim-unicast-visibility-test-2026-10-20).

## Project 05 — Scoped `tc` Mirror Proof, Packet Capture and Recovery (2026-10-20)

| ID | Verification | Evidence / qualification | Status |
|---|---|---|---|
| V-105 | TrueNAS `br30` guest NIC and clean-filter inventory | `ip -o link show master br30` listed `vnet8` (MAC suffix `11:81:0a`), `vnet11` (`0c:44:23`), `vnet13` (`31:78:bd`). TrueNAS `vm.device.query` mapped NICs attached to `br30` to VM 8 Ubuntu, VM 7 Linux Mint, VM 11 Windows respectively. Before test `tc qdisc show dev vnet13` returned only `noqueue`; ingress and egress filter listings were empty. `vnet11` qdisc also `noqueue`. | PASS — verified current host-side NIC mapping; no pre-existing `vnet13` tc filters listed |
| V-106 | Temporary rollback safeguards | Operator saved `/mnt/Pool1/CyberBlue/cyberblue-br30-mirror-rollback.sh`, read back contents, and verified `/bin/sh -n` syntax PASS. Script conditionally removes `clsact` from `vnet13`; it is safe only while this test owns that entire attachment. `tc` and `systemd-run` dependencies checked. First 15-minute timer `cyberblue-br30-mirror-rollback.timer` ran at 10:22:58 PDT; journal included `CYBERBLUE BR30 MIRROR ROLLBACK COMPLETE` and successful service deactivation; subsequent `tc qdisc show dev vnet13` returned `noqueue` only. New R2 15-minute timer `cyberblue-br30-mirror-rollback-r2.timer` was verified active/waiting with trigger 10:44:09 PDT. | PASS — first timer independently demonstrated rollback; R2 created/verified, execution not claimed |
| V-107 | Scoped Windows unicast traffic-control action | With R2 protection, operator added `clsact` on Windows host `vnet13` and one ingress `flower` filter matching IPv4 ICMP source `10.10.30.10`, destination `10.10.30.1`, with `mirred egress mirror dev vnet11`. `tc -s filter show dev vnet13 ingress` readback verified fields and mirror action. After four Windows ping requests, stats showed `Sent 240 bytes 4 pkt (dropped 0)`. An earlier Mint capture had shown zero matching packets despite a valid mirror filter; do not convert this to an end-to-end PASS. | PASS — narrow `tc` match/action processed four packets, not independently end-to-end delivery |
| V-108 | TrueNAS host-side mirror destination capture | Windows `ping.exe -4 -S 10.10.30.10 -n 4 10.10.30.1` returned 4/4 replies. Concurrent host `sudo timeout 120 tcpdump -ni vnet11 -c 4 'icmp and src host 10.10.30.10 and dst host 10.10.30.1'` logged echo-request sequences **220–223**, and final statistics `4 packets captured, 4 received by filter, 0 dropped by kernel`. | PASS — scoped mirrored packet visibility on TrueNAS host-side Mint NIC |
| V-109 | Linux Mint Suricata sensor interface receives mirrored unicast | In a subsequent coordinated test while mirror R2 was installed, Windows Victim ping returned 4/4 replies (0% loss). Linux Mint `sudo timeout 90 tcpdump -ni ens5 -c 4 'icmp and src host 10.10.30.10 and dst host 10.10.30.1'` captured echo-request sequences **224–227**, reporting `4 packets captured, 4 received by filter, 0 dropped by kernel`. This proves this flow reached the Mint monitoring NIC, **not** that Suricata processed/logged it or Wazuh alerted. Host-side and guest-side captures were separate ping rounds. | **PASS — scoped host-to-Mint mirror visibility** |
| V-110 | Manual mirror rollback, independently read back | Before R2 timer expiry, operator ran `sudo /bin/sh /mnt/Pool1/CyberBlue/cyberblue-br30-mirror-rollback.sh`, received `CYBERBLUE BR30 MIRROR ROLLBACK COMPLETE`. Independent `sudo tc qdisc show dev vnet13` then returned only `qdisc noqueue 8023: root refcnt 2` with **no `clsact`**. Therefore the temporary mirror and its attached filters are gone. R2 timer remains scheduled as verified earlier; no later R2 journal evidence was supplied. | **PASS — manual rollback and independent clean qdisc readback** |

**Evidence and limitation:** Controlled, filtered Windows-to-gateway unicast traffic was observable on `vnet11` and on Linux Mint `ens5` **only with the temporary, reversible mirror installed**. A full-network, bi-directional or persistent sensor tap is **not** established. Suricata EVE event ingestion for these exact ICMP requests and any Wazuh alert remain **NOT TESTED** in this checkpoint. No permanent host/bridge/firewall configuration was changed.

[Build notes](build-notes.md#project-05--scoped-br30-unicast-mirror-and-restoration-2026-10-20).

## Project 05 — Suricata EVE Flow Ingestion of Mirrored Victim Traffic (2026-10-20)

| ID | Verification | Evidence / qualification | Status |
|---|---|---|---|
| V-111 | Suricata EVE records of scoped Victim unicast mirror | On Linux Mint, operator searched `/var/log/suricata/eve.json` for `"in_iface":"ens5"`, `"src_ip":"10.10.30.10"`, `"dest_ip":"10.10.30.1"`. Three EVE JSON **`event_type:"flow"`** records matched at `2026-10-20T12:33:57.419972-0500`, `12:37:56.742489-0500`, and `12:40:46.215950-0500`. All reported `proto:"ICMP"`, `icmp_type:8`, `icmp_code:0`, `flow.pkts_toserver:4`, `flow.pkts_toclient:0`, `flow.bytes_toserver:296`. Last flow started `12:40:08.451863-0500` and ended `12:40:11.487714-0500`, correlating to Mint `ens5` tcpdump's four echo requests in the same interval (sequences 224–227). | **PASS — Suricata logged mirrored Victim unicast as flow telemetry; detection `alert` and Wazuh ingestion not proven** |

**Qualification:** These are three distinct 4-request flow events across three controlled ping rounds, not a single 12-packet flow. No echo replies were observed by these `ens5` flow records (`pkts_toclient:0`), consistent with a one-way mirror filter matching requests. The temporary mirror was independently removed before the EVE inspection. Historical Wazuh rule 86601 alert evidence relates to earlier tests, not to these flow records. A new EVE `event_type:alert` and Wazuh detection require separate verification.

[Build notes](build-notes.md#project-05--mirrored-windows-icmp-suricata-eve-flow-evidence-2026-10-20).

## Project 05 — Victim SID 1000002 End-to-End Suricata and Wazuh Alert (2026-10-20)

| ID | Verification | Evidence / qualification | Status |
|---|---|---|---|
| V-112 | Preserve original rule and validate new Victim rule | Original `/etc/suricata/rules/local.rules` contained Attack rule SID `1000001` for `10.10.40.10 → 10.10.40.1` ICMP echo requests. `suricata.yaml` configured `/var/lib/suricata/rules/suricata.rules` and `/etc/suricata/rules/local.rules`. SID `1000002` search returned no match; operator created and byte-compared backup `/etc/suricata/rules/local.rules.cyberblue-pre-victim-20261020-125610.bak`, then appended `alert icmp 10.10.30.10 any -> 10.10.30.1 any (msg:"CYBERBLUE TEST - Victim ICMP Echo Request"; itype:8; sid:1000002; rev:1;)`. `sudo timeout 120 suricata -T -c /etc/suricata/suricata.yaml` exited `0` after reporting configuration successfully loaded. | **PASS — backup and configuration test** |
| V-113 | Suricata live ruleset reload | `systemctl show suricata` returned `CanReload=yes` with `suricatasc -c reload-rules` available. Direct `sudo suricatasc -c reload-rules` returned `{"message":"done","return":"OK"}`. `sudo suricatasc -c ruleset-stats` returned `rules_loaded:53145`, `rules_failed:0`, `rules_skipped:0` (previous count 53144). Available commands do not include a direct loaded-SID list; the live rule's presence is subsequently established by the actual SID 1000002 alerts in V-114. | **PASS — reload returned OK, zero rule failures; live SID validated by V-114** |
| V-114 | Victim ICMP Suricata alert generation and restoration | TrueNAS `vnet13` qdisc was read back `noqueue` before test. R3 rollback timer `cyberblue-br30-mirror-rollback-r3.timer` created and independently verified `active (waiting)`, scheduled 11:32:55 PDT. Operator attached `clsact` and reused narrow `vnet13 ingress` flower ICMP filter for `10.10.30.10 → 10.10.30.1`, mirrored to `vnet11`; `tc -s filter show` confirmed match fields and mirror action before test (0 packets at that time). Windows `ping.exe -4 -S 10.10.30.10 -n 4 10.10.30.1` returned 4 sent, 4 received, 0 lost. Mint EVE JSON then showed **four** `event_type:"alert"` records on `ens5` at **2026-10-20 13:16:07–13:16:10 -0500**, for `src_ip 10.10.30.10`, `dest_ip 10.10.30.1`, ICMP type 8/code 0, `alert.signature_id:1000002`, `alert.action:"allowed"`, `alert.severity:3`; one shared flow ID and increasing request counts 1–4. Operator manually ran saved rollback script (completion message) and independently checked `sudo tc qdisc show dev vnet13` returned only `qdisc noqueue 8023: root refcnt 2` with no `clsact`. | **PASS — four Suricata detections and verified mirror cleanup** |
| V-115 | Wazuh manager receives Suricata SID 1000002 alerts | On `siem-01` manager, operator ran `sudo grep -F '1000002' /var/ossec/logs/alerts/alerts.json | tail -n 5 | cut -c 1-1000`. Four matching Wazuh alert JSON records appeared at `2026-10-20T18:16:07.444+0000`, `18:16:09.444+0000` (twice), and `18:16:11.446+0000`. Each had Wazuh `rule.id:"86601"`, `rule.level:3`, `rule.description:"Suricata: Alert - CYBERBLUE TEST - Victim ICMP Echo Request"`, `agent.id:"003"`, `agent.name:"cyberblue-suricata"`, `agent.ip:"192.168.1.92"`, `manager.name:"siem-01"`, `data.alert.signature_id:"1000002"`, `data.in_iface:"ens5"`, `src_ip 10.10.30.10`, `dest_ip 10.10.30.1`. `rule.firedtimes` advanced 1–4. The **fifth** grep match was an unrelated `sudo` rule `5402` event whose logged command contained the string `1000002`; it is **not** a fifth Suricata alert. | **PASS — four exact Suricata alerts ingested by Wazuh rule 86601** |

**Boundary:** End-to-end detection is validated **only** for the time-bounded, one-direction Windows-to-gateway ICMP test while the temporary `tc` mirror was installed. Current `vnet13` qdisc has no `clsact`; there is no persistent bridge-wide packet mirroring or full bidirectional coverage. Wazuh manager file ingestion is proven; dashboard screenshot/evidence is not yet claimed. Previous V-111 flow-only evidence remains historically accurate for its earlier test.

[Build notes](build-notes.md#project-05--victim-suricata-sid-1000002-and-wazuh-detection-2026-10-20).

## Project 05 — Uploaded Evidence for V-115 (2026-10-20)

**Evidence file verified in repository:** [`evidence/05-victim-suricata-wazuh-detection.png`](evidence/05-victim-suricata-wazuh-detection.png), uploaded by operator and confirmed present in the GitHub Project 05 `evidence/` directory. Operator stated the screenshot of the Wazuh manager terminal four-alert summary was saved and uploaded. The actual text results independently furnished in this validation session were `MATCHING ALERTS: 4` with agent `003`, Wazuh rule `86601`, and Suricata SID `1000002`. **Upload existence verified; pixel/content review not performed.** This is terminal evidence, not a Wazuh Threat Hunting dashboard screenshot.

[Evidence index](evidence/README.md#captured-screenshot--2026-10-20-victim-suricata-to-wazuh).

## Project 05 — Wazuh Victim Alert Dashboard and Screenshot Evidence (2026-10-20)

| ID | Verification | Evidence / scope | Result |
|---|---|---|---|
| V-116 | Wazuh Threat Hunting shows Victim detection | Operator opened **Threat Hunting**, filtered for Wazuh `rule.id:86601` around October 20, 2026, 13:16 CDT, and confirmed that an expanded alert's `data.alert.signature_id` was **`1000002`**. Because the expanded event did not fit into one screenshot, operator separately captured an overview and an alert-details view. GitHub evidence directory was independently inspected and contains both [`10-wazuh-victim-alert-overview.png`](evidence/10-wazuh-victim-alert-overview.png) and [`11-wazuh-victim-alert-details.png`](evidence/11-wazuh-victim-alert-details.png). The binary screenshots' contents have **not** been independently visually reviewed; the individual signature confirmation rests on operator's dashboard readback. | **PASS — operator-confirmed matching SID in Threat Hunting; both evidence files uploaded and present** |

**Qualification:** Four exact SID 1000002 Wazuh manager alerts are separately established in **V-115**. V-116 only establishes at least **one** matching dashboard alert; the operator did not count all four in the dashboard. Upload presence and names are checked; screenshot pixels are not verified. Earlier one-way temporary TrueNAS mirror remains removed after independently verified manual rollback. This does not certify ongoing collection of unicast Victim traffic.

[Evidence index](evidence/README.md#wazuh-victim-alert-dashboard--2026-10-20).

## Project 05 — R3 Scheduled Rollback Execution (2026-10-20)

| ID | Verification | Evidence / scope | Result |
|---|---|---|---|
| V-117 | Timed backup rollback R3 executes after earlier manual restoration | On TrueNAS the operator ran `sudo journalctl -u cyberblue-br30-mirror-rollback-r3.service --since "2026-10-20 11:32:00" --no-pager`. Journal shows at **Oct 20 11:32:57 PDT**: systemd **Started** `cyberblue-br30-mirror-rollback-r3.service`, script printed `CYBERBLUE BR30 MIRROR ROLLBACK COMPLETE`, and systemd recorded `Deactivated successfully`. Manual script rollback and the independent `noqueue`/no-`clsact` verification had already been completed around 11:18 PDT. Timer execution is therefore demonstrated as a redundant cleanup, not evidence of a second active mirror removal. No separate **post-timer** `tc` qdisc readback is included in this checkpoint. | **PASS — scheduled rollback script ran and service exited successfully** |

[Build notes](build-notes.md#project-05--r3-scheduled-rollback-journal-verification-2026-10-20).

## Project 05 — Post-R3-Timer Qdisc State Confirmed (2026-10-20)

| ID | Verification | Evidence / scope | Result |
|---|---|---|---|
| V-118 | Independent TrueNAS `vnet13` qdisc readback **after** scheduled R3 automatic rollback | After `cyberblue-br30-mirror-rollback-r3.service` executed at 2026-10-20 **11:32:57 PDT** (V-117), operator ran `sudo tc qdisc show dev vnet13`. Output: `qdisc noqueue 8023: root refcnt 2`, with **no `clsact`**. This directly confirms the temporary mirroring qdisc remains absent after the redundant scheduled rollback. Historical manual rollback and pre-timer clean qdisc verification are separately documented in V-114. | **PASS — clean post-timer network qdisc state** |

**Boundary:** `tc qdisc` validates this host VM interface's qdisc state; it does not prove absence of unrelated packet capture settings on other devices or ongoing full-network IDS coverage. The narrowly scoped Victim ICMP mirror was a temporary test and is off.

[Build notes](build-notes.md#project-05--post-r3-rollback-qdisc-verification-2026-10-20).
