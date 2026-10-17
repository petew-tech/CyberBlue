# Project 05 — Build Notes

## Sensor and Network Configuration

Linux Mint initially used two interfaces; a third passive capture NIC was added on 2026-10-16 for dual-bridge monitoring:

| Interface | Address | Purpose |
|---|---|---|
| `ens3` | `192.168.1.92/24` | Management and Wazuh communication |
| `ens4` | `10.10.40.10/24` | Isolated-network Suricata capture (`br40`) |
| `ens5` | No IPv4; IPv6 link-local observed | New capture interface connected to `br30` |

Observed routes:

- `10.10.40.0/24` is directly connected through `ens4`.
- Wazuh manager `192.168.1.213` is reached through `ens3`.
- Default route is via `192.168.1.1` on `ens3`.

TrueNAS bridge `br40` was observed UP with address `10.10.40.1/24`. The bridge inspection listed host-side interface `vnet4` as its attached interface at that time.

## Suricata Configuration and Rule

- Suricata version: 7.0.3.
- Capture backend: AF_PACKET.
- Initial capture interface: `ens4` on `br40`; later configured AF_PACKET capture on both `ens4` (cluster ID `99`) and `ens5` (cluster ID `100`).
- Custom rule SID: `1000001`.
- Signature: `CYBERBLUE TEST - ICMP Echo Request`.
- EVE JSON output: `/var/log/suricata/eve.json`.

The custom rule detected ICMP echo requests from `10.10.40.10` to `10.10.40.1`. Suricata logged the alert with `event_type: alert`, protocol `ICMP`, and `icmp_type: 8`.

## Wazuh Agent Integration

- Wazuh manager: `192.168.1.213`.
- Agent name: `cyberblue-suricata`.
- Agent ID: `003`.
- The agent log confirmed connection to `192.168.1.213:1514/tcp`.
- Wazuh log collector reported analyzing `/var/log/suricata/eve.json`.
- Wazuh generated alert rule `86601`, described as `Suricata: Alert - CYBERBLUE TEST - ICMP Echo Request`.
- The dashboard search for rule ID `86601` was confirmed visible by the operator.

## Controlled Failure and Recovery

Before testing, systemd reported:

- `Restart=on-failure`
- `MainPID=3502`
- `NRestarts=0`
- Service active and running

The main Suricata process was terminated with SIGKILL as a controlled failure. Afterward, systemd reported:

- `MainPID=13422`
- `NRestarts=1`
- Service active and running

Suricata then loaded 53,144 rules with zero failures. A new ICMP test generated custom SID `1000001`, and the corresponding event was confirmed in Wazuh under rule `86601`.

## Packet Capture Statistics

A post-recovery EVE JSON statistics event reported:

- Kernel packets: 278
- Kernel drops: 0
- Capture errors: 0
- Detection rules loaded: 53,144
- Detection rules failed: 0
- Detection alert count in that statistics interval: 4
- Alert queue overflows: 0

These are point-in-time counters for the observed runtime period.

## Known Limitation — TrueNAS br40 Visibility

The following read-only checks were performed on TrueNAS:

- `ip -d link show br40`: bridge UP, MTU 1500, VLAN filtering disabled.
- `ip -o link show master br40`: showed `vnet4` attached.
- `tc filter show dev vnet4 ingress`: no output.
- `tc filter show dev br40 ingress`: no output.
- `tc qdisc show dev br40`: `noqueue` root qdisc.
- `ip -br addr show br40`: `10.10.40.1/24`.

These checks did not establish traffic mirroring. They also do not conclusively rule out every alternative mirroring method. Broader visibility into traffic between other VMs remains a separate follow-up task; do not change the working network configuration until that assessment is planned.

## Dual-Bridge Capture Follow-Up — 2026-10-16

- Started Kali and verified its `eth1` address `10.10.30.20/24` and access to the TrueNAS `br30` gateway `10.10.30.1` (4 transmitted, 4 received, 0% loss).
- Added a third VirtIO NIC to Linux Mint attached to TrueNAS `br30`, then restarted Linux Mint to activate it.
- Linux Mint `ens5` was observed UP without an IPv4 address (with an IPv6 link-local address); TrueNAS host-side `vnet7` matched the NIC's MAC suffix. Kali was attached as `vnet5`. After reboot Linux Mint `ens4` mapped to TrueNAS `br40` through `vnet6`.
- Backed up `/etc/suricata/suricata.yaml` as `/etc/suricata/suricata.yaml.pre-dual-monitor`.
- Added `ens5` as a second AF_PACKET capture entry with `cluster-id: 100`, `cluster-type: cluster_flow`, and `defrag: yes`; retained `ens4` with cluster ID `99`. Did not enable packet-copying/IPS or bridge forwarding.
- Verified `suricata -T -c /etc/suricata/suricata.yaml` finished with exit status `0`; restarted Suricata successfully. Suricata main process PID `2922` reported two capture threads for each of `ens4` and `ens5` at 12:12:39 CDT.
- EVE JSON `in_iface: ens5` flow records documented DHCP and IPv6-ICMP traffic; this proves traffic reached the capture interface but not full unicast visibility on `br30`.
- Kali's 4-packet ICMP test to `10.10.30.1` succeeded with zero packet loss. On Linux Mint, a simultaneous `tcpdump -ni ens5 'icmp and host 10.10.30.20'` capture reported **0 packets captured, 0 packets received by filter, and 0 kernel drops**. A separate repeat of the same capture also reported zero packets (not an additional validated ping test).
- This is a demonstrated **observation gap for the tested Kali→TrueNAS flow**. It is consistent with Linux bridge forwarding and does not establish that the capture sensor can observe all inter-VM traffic. Traffic mirroring / passive capture architecture is not yet implemented or validated.
- Screenshots `03-suricata-dual-interface-configuration.png` and `04-suricata-dual-interface-initialization.png` were uploaded and verified. The zero-packet [tcpdump screenshot](evidence/05-br30-traffic-visibility-gap.PNG) was uploaded to GitHub and verified.
- The complementary TrueNAS `vnet5` packet capture observed 4 matching ICMP packets and zero kernel drops, documenting that the tested Kali-to-TrueNAS traffic was visible on the host while absent from Mint's `ens5` capture. [Host-side capture screenshot](evidence/06-br30-truenas-packet-capture.PNG) was uploaded and verified. These separate captures support a forwarding/visibility hypothesis; mirroring itself has not yet been implemented or tested.

## Controlled `br30` Mirroring Proof of Concept — 2026-10-16

- On TrueNAS, Kali's bridge port was `vnet5` and Mint's `br30` monitor port was `vnet7` (`ens5` inside Mint). The destination `vnet7` initially used a `noqueue` root qdisc.
- Installed temporary `clsact` attachment on source `vnet5` using `sudo tc qdisc add dev vnet5 clsact`. Verified with `sudo tc qdisc show dev vnet5`.
- Added one narrowly scoped ingress mirror with `sudo tc filter add dev vnet5 ingress protocol ip pref 10 flower ip_proto icmp src_ip 10.10.30.20 action mirred egress mirror dev vnet7`. No redirect/drop action or IPv4/IPv6 forwarding was enabled. This is runtime-only configuration, not a persistent TrueNAS networking setup.
- The installed flower filter showed `mirred (Egress Mirror to device vnet7)` and its initial counter was 0. During a 4-ping Kali test to `10.10.30.1`, Linux Mint `sudo tcpdump -ni ens5 -c 4 'icmp and host 10.10.30.20'` captured four echo requests and reported 0 kernel drops.
- TrueNAS `sudo tc -s filter show dev vnet5 ingress` then reported **Sent 336 bytes 4 pkt**, with **dropped 0**. Suricata EVE JSON recorded `in_iface: ens5`, source `10.10.30.20`, destination `10.10.30.1`, `flow.pkts_toserver:4`, `flow.pkts_toclient:0`, and `flow.alerted:false`. [Suricata mirrored-flow screenshot](evidence/07-suricata-br30-mirrored-flow.png) uploaded and verified.
- **Result: PASS for the single directed Kali ICMP request flow.** Replies were not mirrored or captured in this test. Coverage of other traffic/VMs and `br40` is not yet validated. No Suricata alert or new Wazuh event was demonstrated by this flow log. Treat `tc` rule as a proof of concept pending resource, safety, cleanup, and persistence planning.

## Bidirectional ICMP Mirroring Follow-Up — 2026-10-16

- Confirmed no existing `tc` egress filters on Kali's host-side TrueNAS interface `vnet5`.
- Installed additional scoped egress mirror on TrueNAS: `sudo tc filter add dev vnet5 egress protocol ip pref 11 flower ip_proto icmp src_ip 10.10.30.1 dst_ip 10.10.30.20 action mirred egress mirror dev vnet7`. Rule inspection confirmed flower match and `mirred (Egress Mirror to device vnet7)`.
- During a new 4-ping Kali `eth1` → `10.10.30.1` exchange, Linux Mint `sudo tcpdump -ni ens5 -c 8 'icmp and host 10.10.30.20'` captured **4 echo requests and 4 echo replies**, with **0 kernel drops**.
- Suricata remained `active`. Its log continued updating with statistics and other flows. Its cumulative ICMPv4 decoder count increased from **12 to 20** following another controlled four-ping exchange (consistent with 8 ICMP packets, but the counter is cumulative).
- The flow for the 13:06:00–13:06:04 exchange appeared in EVE JSON at 13:11:12 with `in_iface:ens5`, `src_ip:10.10.30.20`, `dest_ip:10.10.30.1`, `pkts_toserver:4`, `pkts_toclient:4`, `state:established`, `reason:timeout`, `alerted:false`. The later log inspection resolved the apparent temporary absence of that record. [Bidirectional Suricata flow screenshot](evidence/08-suricata-br30-bidirectional-flow.png) uploaded and verified.
- **Result: PASS for bidirectional ICMP monitoring of this Kali ↔ TrueNAS gateway exchange on `br30`.** The two TrueNAS `tc` rules are temporary and strictly filtered to this test traffic. Broader mirroring for other VM conversations and protocols, `br40`, reboot persistence, and Wazuh alert generation on mirrored `br30` traffic remain open. Do not treat this as bridge-wide coverage or an intrusion alert.


## SOC Network Expansion and Mirroring Cleanup — 2026-10-17

- As part of migration toward separated SOC, victim and attack zones, removed the temporary TrueNAS `tc` proof-of-concept rules using `sudo tc qdisc del dev vnet5 clsact`. `sudo tc qdisc show dev vnet5` then showed only the `noqueue` qdisc; the earlier mirrored packet evidence is historical and must not be described as a current, persistent capture path.
- Powered down the Mint VM and preserved management NIC (`enp4s0` / `ens3`), Suricata `br40` NIC (`ens4`), and `br30` monitoring NIC (`ens5`). Added a fourth TrueNAS-managed VirtIO NIC (device `53`, MAC `00:a0:98:7e:12:f8`) to newly configured SOC bridge `br20`; Mint detected the NIC as `ens6`.
- Updated only Mint NetworkManager connection `Wired connection 4` (`ens6`) to static `10.10.20.11/24`, no gateway and `ipv4.never-default yes`, IPv6 disabled. Confirmed activated profile, link UP, and assigned address.
- Verified Mint `ens6` → TrueNAS `br20` `10.10.20.1` with 4/4 replies and 0% loss, and Mint `ens6` → Ubuntu SOC `10.10.20.10` with 4/4 replies and 0% loss.
- After reboot, `sudo systemctl is-active suricata` and `sudo systemctl is-active wazuh-agent` each returned `active`. Agent log recorded Wazuh manager connection to `192.168.1.213:1514/tcp` at 2026-10-17 12:13:31.
- **Post-restart regression checks PASS:** Suricata 7.0.3 initialized two capture threads each for `ens4` and `ens5` and loaded 53,144 rules with zero failures. Fresh `ens5` IPv6-ICMP flows and statistics appeared in EVE JSON; sampled capture counters showed 696 kernel packets, zero drops, and zero errors. Mint's `ens4` four-ping test to `10.10.40.1` passed with 0% loss and produced four SID `1000001` Suricata alerts at about 12:24 CDT. The Wazuh manager `siem-01` `alerts.json` recorded the matching level-3 rule `86601` alerts from agent `003` (`cyberblue-suricata`), `firedtimes` reaching 4. Mint management `ens3` ping to `192.168.1.1` also passed 4/4 (0% loss). This validates end-to-end alert delivery to the Wazuh manager after the SOC NIC upgrade.
- **Open:** Fresh Wazuh dashboard confirmation, broader/persistent traffic mirroring on the finalized topology, TrueNAS `br20` reboot persistence, and full zone isolation tests. The former temporary `tc` mirror was removed and is not currently active.
