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

## Threat Hunting Dashboard Validation — 2026-10-17

- Wazuh Threat Hunting dashboard screenshot showed `rule.id: 86601`, manager `siem-01`, sensor agent `003`, and **4 hits** for October 17 around 12:24 CDT, all `cyberblue-suricata` with level 3 and matching ICMP test alert description.
- Originally, the dashboard's existing `agent.id: 002` filter excluded the sensor; after opening agent `003` the correct alerts were visible. This was a dashboard filter-selection issue, not evidence of SIEM ingest failure.
- **End-to-end PASS** for tested ICMP event from Suricata to Wazuh Dashboard. Screenshot [09-wazuh-post-soc-migration-alerts.png](evidence/09-wazuh-post-soc-migration-alerts.png) was uploaded and verified in GitHub. Full victim/attack bridge mirroring and zone isolation remain open.

---

## Module 06 — Ubuntu and Windows IP Migration (2026-10-18)

### Ubuntu: retire `10.10.30.10/24` from the guest

- Edited `/etc/netplan/00-installer-config.yaml` to retain only `mgmt0` matched by MAC `00:a0:98:76:2c:c2`, with `dhcp4: true` and `dhcp6: true`. The separate SOC Netplan definition for `ens4` stayed at `10.10.20.10/24`.
- `sudo netplan generate` returned to the prompt with no errors; `sudo netplan get` listed `mgmt0` and `ens4`, **not** `lab0`. The former generated runtime file still configured `lab0` at `10.10.30.10/24` until the new configuration was activated, as shown by `networkctl status lab0 --no-pager`.
- Confirmed default IPv4 route `via 192.168.1.1 dev mgmt0` and SOC connected route `10.10.20.0/24 dev ens4`. Opened TrueNAS Ubuntu VM console as fallback before applying a potentially disruptive change to the SSH-managed guest.
- Ran `sudo netplan try --timeout 300`, observed no errors, verified live interfaces/routes from SSH, and accepted the configuration. `lab0` remained a live VirtIO interface but with IPv6 link-local only; the IPv4 route `10.10.30.0/24` disappeared.
- Captured [applied routing screenshot](evidence/project06-netplan-routing-validated.png), rebooted Ubuntu with `sudo reboot`, successfully reconnected over SSH, and repeated `ip -br addr && ip -4 route`. After reboot the former `lab0` appeared as default interface `ens3` without IPv4; `mgmt0 192.168.1.243/24` and `ens4 10.10.20.10/24` persisted with the expected routes. Captured [post-reboot screenshot](evidence/project06-netplan-post-reboot-validation.png).
- **Limit:** The old virtual NIC was not removed from TrueNAS, and absence of a guest IPv4 address is not proof of physical or Layer-2 isolation.

### Windows 11: take over the released Victim address

- Verified management `Ethernet 3`: `192.168.1.151/24`, gateway/DNS `192.168.1.1`; Victim NIC `Ethernet` initially had temporary `10.10.30.11/24`, Preferred.
- Examined apparent target-address ping responses before reassignment. A repeat test reported `Reply from 10.10.30.11: Destination host unreachable`, **not** echo replies from `10.10.30.10`; `Get-NetNeighbor` showed an `Unreachable` neighbor with all-zero MAC. No active claimant was observed, although these checks do not absolutely rule out a conflict.
- In elevated PowerShell ran `New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.30.10 -PrefixLength 24`. Initial `Tentative`/`Invalid` creation output became `Preferred` on follow-up for the new and old addresses.
- The new source IP successfully pinged TrueNAS `10.10.30.1` with 4/4 replies. Then ran `Remove-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "10.10.30.11" -Confirm:$false`.
- Validated `Ethernet 10.10.30.10/24` and `Ethernet 3 192.168.1.151/24`, both Preferred; the old address was absent. IPv4 routes showed `10.10.30.0/24` directly connected on `Ethernet` and default `0.0.0.0/0` via `192.168.1.1` on `Ethernet 3`.
- Before restart, gateway pings from each respective source IP passed (4/4, 0% loss). Captured and uploaded [pre-reboot Windows evidence](evidence/module06-windows-victim-ip-pre-reboot.png).
- Restarted Windows using `Restart-Computer`. Rechecked addresses, routes and connectivity: `10.10.30.10 → 10.10.30.1` returned 4/4 (0% loss, average 2 ms); `192.168.1.151 → 192.168.1.1` returned 4/4 (0% loss, average reported 0 ms). Captured and uploaded [post-reboot evidence](evidence/module06-windows-victim-post-reboot-validation.png).

**As of the Ubuntu/Windows checkpoint:** Kali attack-zone migration was still pending; it was completed and validated in the following 2026-10-18 section. Revised firewall/isolation tests, any planned Ubuntu virtual NIC removal, full passive sensor coverage, and independent TrueNAS host bridge reboot-persistence testing remain open. No final three-zone isolation claim is made here.

## Kali Attack-Zone Migration — NIC Order Recovery and MAC Persistence (2026-10-18)

### Initial state and staged change

- Kali (TrueNAS VM `6`) initially had `eth0 192.168.1.91/24` management and `eth1 10.10.30.20/24` on `br30`. Route `0.0.0.0/0` via `192.168.1.1 dev eth0`; `10.10.30.0/24` directly attached via `eth1`.
- Kali `eth1` initially reported MAC `00:a0:98:1d:59:aa`. TrueNAS `midclt call vm.device.query '[["vm","=",6],["dtype","=","NIC"]]'` showed management device `16` on `enp4s0` with MAC `00:a0:98:5b:5f:c8`; lab device `42` on `br30` with stored MAC `null`.
- TrueNAS `br40` verified UP with `10.10.40.1/24`. Proposed `10.10.40.20` received no replies to four TrueNAS `ping -I br40` probes; this was only a preliminary no-responder check, not a guarantee of uniqueness.
- On Kali, NetworkManager `cyberblue-lab` initially bound to `eth1`, with `10.10.30.20/24`, `ipv4.method=manual`, autoconnect enabled, no IPv4 gateway, and `ipv4.never-default=yes`. Management `eth0` gateway ping to `192.168.1.1` passed 4/4.
- Staged `sudo nmcli connection modify "cyberblue-lab" ipv4.addresses "10.10.40.20/24"` **without activating** it, then verified the saved profile.
- Powered off Kali. TrueNAS `vm.status 6` returned `STOPPED` / `SHUTOFF`. Changed **only** VM NIC device `42` from `br30` to `br40`; device `16` remained on `enp4s0`. TrueNAS `vm.device.query` verified both before starting Kali.

### Fault: guest interface order changed and SSH failed

- After restart, Kali showed `eth0` MAC `00:a0:98:76:2d:fd` physically attached to **Attack `br40`**, but NetworkManager had applied management profile `Wired connection 1` to `eth0`, giving it `192.168.1.91`. Kali `eth1` MAC `00:a0:98:5b:5f:c8` physically attached to **management `enp4s0`** but held `cyberblue-lab` and `10.10.40.20`.
- SSH failed, and `ping -I eth0 -c 4 192.168.1.1` returned 0/4 with local `Destination Host Unreachable`. IPs existed but on the **wrong physical networks**; this is an observed **interface-name/profile-mapping fault**, not a failure of the TrueNAS bridge reassignment.
- Recovered through Kali's TrueNAS SPICE console. Inspected `nmcli device status`; verified `Wired connection 1` contains static management `192.168.1.91/24`, gateway `192.168.1.1`.
- Saved profile changes: `sudo nmcli con mod "Wired connection 1" connection.interface-name eth1` and `sudo nmcli con mod "cyberblue-lab" connection.interface-name eth0`. Both saved interface names verified with `nmcli -g connection.interface-name`.
- Activated `sudo nmcli con up "Wired connection 1" ifname eth1`, then management ping `ping -I eth1 -c 4 192.168.1.1` passed 4/4 and PuTTY SSH access to `192.168.1.91` was restored.
- Activated `sudo nmcli con up "cyberblue-lab" ifname eth0`. `ip -br addr` showed `eth0 10.10.40.20/24` and `eth1 192.168.1.91/24`; routes pointed `10.10.40.0/24` to `eth0` and default via management `eth1`. Both bridge-gateway pings passed 4/4. Uploaded [pre-reboot Kali evidence](evidence/module06-kali-attack-network-pre-reboot.png).

### Persistent fix: explicitly configured NIC MAC addresses

- Confirmed Kali management `eth1` MAC `00:a0:98:5b:5f:c8`; Attack `eth0` MAC `00:a0:98:76:2d:fd`. The management TrueNAS device `16` already had a static MAC; Attack device `42` still had `mac: null` and had previously appeared under a different MAC, so interface-name-only binding was unsuitable.
- Changed saved Kali management profile to match stable MAC, removing its interface-name restriction:
  `sudo nmcli con mod "Wired connection 1" connection.interface-name "" 802-3-ethernet.mac-address "00:a0:98:5b:5f:c8"`. Verified interface-name `--`, MAC `00:A0:98:5B:5F:C8`.
- After Kali shutdown, `midclt call vm.status 6` confirmed `STOPPED`. TrueNAS `midclt call vm.device.query '[["dtype","=","NIC"]]'` showed no **explicitly configured** duplicate of proposed `00:a0:98:76:2d:fd`; entries with `mac: null` mean this was not a complete uniqueness guarantee.
- Set permanent MAC `00:a0:98:76:2d:fd` on TrueNAS VM `6` NIC device `42`, keeping `br40` / VirtIO and leaving management device `16` unchanged. Queried TrueNAS again: device `16` `enp4s0` MAC `00:a0:98:5b:5f:c8`, device `42` `br40` MAC `00:a0:98:76:2d:fd`.
- Started Kali and verified `eth0` was Attack MAC `00:a0:98:76:2d:fd`, `eth1` management MAC `00:a0:98:5b:5f:c8`. Changed saved Attack profile to match the stable MAC rather than an interface name:
  `sudo nmcli con mod "cyberblue-lab" connection.interface-name "" 802-3-ethernet.mac-address "00:a0:98:76:2d:fd"`; verified name `--` and MAC `00:A0:98:76:2D:FD`.
- Verified addresses `eth0 10.10.40.20/24`, `eth1 192.168.1.91/24`; default gateway `192.168.1.1 dev eth1`, direct `10.10.40.0/24 dev eth0`, no `10.10.30.0/24` connected route. Both gateways returned 4/4 replies.

### Final guest reboot-persistence validation

- Rebooted Kali via SSH; PuTTY reconnected after boot. `ip -br addr && ip -4 route` showed the same two addresses, management default route via `eth1`, Attack connected route via `eth0`, and no old Victim route.
- Post-reboot `ping -I eth0 -c 4 10.10.40.1`: 4/4, 0% loss, average **0.280 ms**.
- Post-reboot `ping -I eth1 -c 4 192.168.1.1`: 4/4, 0% loss, average **0.480 ms**.
- Saved [post-reboot Kali evidence](evidence/module06-kali-attack-network-post-reboot.png), uploaded to GitHub and verified.

**Result: PASS — Kali guest-network migration, permanent configured NIC MACs, MAC-bound NetworkManager profiles, restored SSH and guest reboot persistence.** Do not interpret this as a TrueNAS **host** reboot validation, complete security zone isolation, or Suricata coverage of arbitrary traffic. Prior `br30` ICMP mirroring records are historical because scoped `tc` filters were removed before the new topology.

## Module 06 — Victim ↔ Attack IPv4 Firewall Test Procedure (2026-10-19)

**Context:** After Ubuntu/Windows/Kali guest-network migration, TrueNAS `br30` is the Victim segment (`10.10.30.0/24`), and `br40` is the Attack segment (`10.10.40.0/24`). Both tested guests also have `192.168.1.0/24` management connections, so default route selection must be verified before firewall experiments.

### Read-only TrueNAS baseline

```bash
sudo iptables -S FORWARD
sudo iptables-legacy -S FORWARD
sudo iptables-legacy -S ts-forward
sudo iptables -L FORWARD -n -v --line-numbers
cat /proc/sys/net/ipv4/ip_forward
```

- Active `iptables` FORWARD policy `ACCEPT`, Docker chains/rules first; then `br40 → br30 DROP` and `br30 → br40 DROP`. Initial counters: **4 packets / 336 bytes each**, from earlier tests.
- A parallel `iptables-legacy` FORWARD chain had default `ACCEPT` and jumped to Tailscale `ts-forward`. That chain used Tailscale interface marks, an ACCEPT for marked packets, a DROP for certain `100.64.0.0/10` traffic exiting outside `tailscale0`, and an ACCEPT for `-o tailscale0`; no explicit CyberBlue bridge DROP rules there. The two backend rulesets need to be considered separately; the relevant counter increases were observed in `sudo iptables`.
- Kernel IPv4 forwarding `/proc/sys/net/ipv4/ip_forward` was `1`. A `sysctl` command was unavailable in the TrueNAS user's shell; `cat` on `/proc` provided the read-only value.

### Direction 1: Kali Attack → Windows Victim

Kali's original `ip -4 route get 10.10.30.10` selected management `192.168.1.1 dev eth1 src 192.168.1.91`. To avoid a false-negative test through management, the operator added a temporary **single-destination host route**:

```bash
sudo ip route add 10.10.30.10/32 via 10.10.40.1 dev eth0
ip -4 route get 10.10.30.10
ping -I eth0 -c 4 10.10.30.10
```

Route lookup selected `via 10.10.40.1 dev eth0 src 10.10.40.20`. Result: 4 transmitted / 0 replies (100% loss). TrueNAS `br40 → br30 DROP` counter increased from **4 / 336 bytes** to **8 / 672 bytes**; opposite rule stayed at 4 / 336 bytes.

Cleaned up and verified:

```bash
sudo ip route del 10.10.30.10/32 via 10.10.40.1 dev eth0
ip -4 route get 10.10.30.10
```

Post-cleanup lookup again selected `via 192.168.1.1 dev eth1 src 192.168.1.91`.

### Direction 2: Windows Victim → Kali Attack

Windows originally selected management `Ethernet 3`, `192.168.1.151`, next hop `192.168.1.1` for destination `10.10.40.20`. In elevated PowerShell, created a nonpersistent host route and verified selection:

```powershell
New-NetRoute -DestinationPrefix "10.10.40.20/32" -InterfaceAlias "Ethernet" -NextHop "10.10.30.1" -PolicyStore ActiveStore
Find-NetRoute -RemoteIPAddress 10.10.40.20
ping -4 -S 10.10.30.10 -n 4 10.10.40.20
```

`Find-NetRoute` returned source `10.10.30.10`, `Ethernet` interface index `3`, destination host route `10.10.40.20/32`, and gateway `10.10.30.1`. Ping timed out. TrueNAS `br30 → br40 DROP` count increased from **4 / 336 bytes** to **8 / 576 bytes**; opposite rule stayed at 8 / 672 bytes.

Cleaned up in elevated PowerShell:

```powershell
Remove-NetRoute -DestinationPrefix "10.10.40.20/32" -InterfaceAlias "Ethernet" -NextHop "10.10.30.1" -Confirm:$false
Find-NetRoute -RemoteIPAddress 10.10.40.20
```

Final route lookup again chose management `Ethernet 3`, source `192.168.1.151`, default `0.0.0.0/0` via `192.168.1.1`.

**Validated:** Firewall DROP counters increased by exactly four during each separately scoped ICMP test; routing configuration was restored after each test. No firewall modifications or reboot were required. **Open:** Full protocol matrix, SOC-to-other-zone boundary tests, IPv6, alternate management routing, TrueNAS host reboot and full IDS monitoring. Evidence here is operator-supplied CLI and PowerShell output, not newly uploaded screenshot files.

## Module 06 — Temporary SOC Default-Deny Candidate and Rollback Procedure (2026-10-19)

**Scope:** Controlled validation of routed IPv4 ICMP isolation between SOC (`br20`), Victim (`br30`) and Attack (`br40`). These tests supplement the separately documented Victim ↔ Attack baseline checks. **Temporary candidate only:** no change was made to the registered POSTINIT startup script, `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh`.

### Change preparation and safeguards

- TrueNAS current startup job `initshutdownscript.query` identified an enabled POSTINIT script at `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh`, containing the original two directional `br30` ↔ `br40` DROP rules. Original script backed up to `cyberblue-firewall.sh.pre-br20-20261019.bak` and byte-for-byte comparison passed.
- Prepared `/mnt/Pool1/CyberBlue/cyberblue-firewall-candidate.sh`, using `/usr/sbin/iptables` to remove duplicate CyberBlue DROP rules for the six ordered pairs, then insert all six DROP rules at the front of `FORWARD`. Prepared `/mnt/Pool1/CyberBlue/cyberblue-rollback-br20.sh`, which removes **only** the four SOC-related `br20` ↔ `br30` and `br20` ↔ `br40` DROP rules. Both shell scripts passed `sh -n` checks.
- The candidate intentionally placed all six inter-zone DROP rules **before** Docker-related forwarding rules. Existing `iptables-legacy` tables and Tailscale chains were observed separately; the tested packet counter changes were in the default `iptables` backend.
- Three independent 15-minute transient systemd rollback timers were created (units `cyberblue-br20-rollback.timer`, `cyberblue-br20-rollback2.timer`, and `cyberblue-br20-rollback3.timer`) using `systemd-run --on-active=15m /bin/sh /mnt/Pool1/CyberBlue/cyberblue-rollback-br20.sh`. Each timer was checked `active (waiting)` before its corresponding candidate deployment.
- For each window, executed `sudo /bin/sh /mnt/Pool1/CyberBlue/cyberblue-firewall-candidate.sh`. TrueNAS printed `CANDIDATE APPLIED`; a fresh `sudo iptables -L FORWARD -n -v --line-numbers` showed six DROP rules 1–6 followed by Docker chains, with counters initially zero. Only then were ICMP probes sent.

### Four SOC-related directed probes and cleanup

| Direction | Temporary destination-specific guest route | Ping result | TrueNAS DROP match |
|---|---|---|---|
| Ubuntu SOC `10.10.20.10` → Windows Victim `10.10.30.10` | Ubuntu `10.10.30.10/32 via 10.10.20.1 dev ens4` | 4 sent, 0 replies | `br20 → br30` rule 6: 4 packets / 336 B |
| Ubuntu SOC `10.10.20.10` → Kali Attack `10.10.40.20` | Ubuntu `10.10.40.20/32 via 10.10.20.1 dev ens4` | 4 sent, 0 replies | `br20 → br40` rule 4: 4 packets / 336 B |
| Kali Attack `10.10.40.20` → Ubuntu SOC `10.10.20.10` | Kali `10.10.20.10/32 via 10.10.40.1 dev eth0` | 4 sent, 0 replies | `br40 → br20` rule 3: 4 packets / 336 B |
| Windows Victim `10.10.30.10` → Ubuntu SOC `10.10.20.10` | PowerShell ActiveStore `10.10.20.10/32` on `Ethernet`, next hop `10.10.30.1` | 4 sent, 0 replies | `br30 → br20` rule 5: 4 packets / 240 B |

Linux guest route lookup was checked with `ip -4 route get <destination>` before testing. Windows used `Find-NetRoute -RemoteIPAddress 10.10.20.10`, verifying Victim source `10.10.30.10`, interface index `3`, gateway `10.10.30.1`. Ubuntu used `ping -I ens4 -c 4 -W 1`, Kali used `ping -I eth0 -c 4 -W 1`, and Windows used `ping -4 -S 10.10.30.10 -n 4 10.10.20.10`.

One attempted `ping -I ens4` was issued at the TrueNAS prompt and returned `SO_BINDTODEVICE ens4: No such device` (twice); it sent no relevant test traffic. The proper Ubuntu guest ping was subsequently executed successfully as a **blocked** test, and the TrueNAS `br20 → br40` counter corroborated it.

After each directed test, removed the **single-host** guest route and read back management route selection:
- Ubuntu → Victim or Attack returned via `192.168.1.1 dev mgmt0 src 192.168.1.243`.
- Kali → SOC returned via `192.168.1.1 dev eth1 src 192.168.1.91`.
- Windows → SOC returned to `Ethernet 3`, source `192.168.1.151`, default `0.0.0.0/0` next hop `192.168.1.1`.

### Controlled rollback and end state

After the relevant tests in each window, executed:
```bash
sudo /bin/sh /mnt/Pool1/CyberBlue/cyberblue-rollback-br20.sh && echo "SOC ROLLBACK EXECUTED"
sudo iptables -S FORWARD
```

All three manual rollbacks reported success; every readback showed the original `-P FORWARD ACCEPT`, the two `br40` ↔ `br30` DROP rules and unchanged Docker-related entries, **without any `br20` DROP rules**. Each corresponding scheduled timer was stopped and `systemctl is-active` confirmed `inactive`. The original persistent startup script was not replaced.

**Result:** Four temporary SOC-direction IPv4 ICMP tests PASS; with the two earlier baseline Victim ↔ Attack ICMP tests, **six of six** tested directed routed IPv4 ICMP paths PASS. **Current enforced baseline is only Victim ↔ Attack IPv4**, not the approved three-zone policy. Do not report complete isolation, IPv6 or management-bypass coverage, TCP/UDP filtering, TrueNAS host restart survival, or full Suricata visibility as validated.

[Detailed V-52–V-60 evidence](validation.md#module-06--three-zone-scoped-ipv4-icmp-firewall-trial-2026-10-19).

## Module 06 — Dedicated Chain Candidate and Safe Rollback (2026-10-20)

**Purpose:** Replace the earlier six-direct-rule test approach with a prepared, dedicated `CYBERBLUE_ZONES` chain, attaching it only after its six IPv4 DROP entries have been populated. This was a **temporary controlled trial**, not a modification of the registered TrueNAS POSTINIT firewall script.

**Saved TrueNAS scripts:**

- `/mnt/Pool1/CyberBlue/cyberblue-firewall-chain-candidate.sh` — uses `/usr/sbin/iptables -w`, refuses to rebuild a chain already attached to `FORWARD`, creates `CYBERBLUE_ZONES` if absent, flushes its **detached** contents, appends six directional bridge-pair DROP rules, then inserts a single `FORWARD` jump at position 1.
- `/mnt/Pool1/CyberBlue/cyberblue-firewall-chain-rollback.sh` — removes matching `FORWARD -j CYBERBLUE_ZONES` jumps with `iptables -w -D`; leaves the original Victim ↔ Attack rules and the now-inactive dedicated chain intact.
- Both files passed `sudo /bin/sh -n` and the operator pasted their full saved contents for review. The rollback script was later run successfully, not merely syntax-checked.

**Preflight:** `iptables -V` returned `iptables v1.8.9 (nf_tables)`; `nft -a list chain ip filter FORWARD` displayed the existing two bridge DROPs before Docker chains and warned the table was managed by `iptables-nft`. An initial direct chain query through `iptables` returned an ambiguous compatibility error; subsequent `nft` and `iptables-save -t filter` reads showed no pre-existing `CYBERBLUE_ZONES` chain or jump. No direct `nft` changes were made.

**Timed trial, October 20:** Created a transient 15-minute `cyberblue-chain-rollback-20261020.timer`, confirmed `active (waiting)` with scheduled trigger **02:51:23 PDT**, and then executed:

```bash
sudo /bin/sh /mnt/Pool1/CyberBlue/cyberblue-firewall-chain-candidate.sh
sudo iptables -L FORWARD -n -v --line-numbers
sudo iptables -L CYBERBLUE_ZONES -n -v --line-numbers
```

Candidate returned `CANDIDATE APPLIED`. `FORWARD` position 1 jumped to `CYBERBLUE_ZONES`; original `br40→br30` and `br30→br40` DROPs remained at positions 2–3, ahead of Docker rules 4–9. The dedicated chain contained six rules, initially with zero DROP hits. The `FORWARD` jump counter alone is **not** evidence of drops.

**Three probes run in the dedicated-chain window:**

| Direction | Temporary /32 route via lab gateway | ICMP sent/replies | Matching dedicated-chain DROP |
|---|---|---|---|
| Kali Attack `10.10.40.20` → Ubuntu SOC `10.10.20.10` | Kali via `10.10.40.1 dev eth0` | 4 / 0 | Rule 4, `br40→br20`: 4 packets / 336 B |
| Ubuntu SOC `10.10.20.10` → Kali Attack `10.10.40.20` | Ubuntu via `10.10.20.1 dev ens4` | 4 / 0 | Rule 3, `br20→br40`: 4 packets / 336 B |
| Ubuntu SOC `10.10.20.10` → Windows Victim `10.10.30.10` | Ubuntu via `10.10.20.1 dev ens4` | 4 / 0 | Rule 1, `br20→br30`: 4 packets / 336 B |

Guest commands used an `EXIT` trap intended to delete each temporary host route. A later independent route-table readback confirmed both Ubuntu test host routes (`10.10.40.20/32`, `10.10.30.10/32`) and Kali's `10.10.20.10/32` host route were absent. Kali `ip -4 route get 10.10.20.10` returned `via 192.168.1.1 dev eth1 src 192.168.1.91`. Subsequent Ubuntu `ip -4 route get 10.10.40.20` and `ip -4 route get 10.10.30.10` independently confirmed the management next hop and interface restored for both targets: `via 192.168.1.1 dev mgmt0 src 192.168.1.243`. The other chain directions were installed but **not re-probed** in this particular session.

**Cleanup:** The operator manually ran `sudo /bin/sh /mnt/Pool1/CyberBlue/cyberblue-firewall-chain-rollback.sh` and received `CYBERBLUE CHAIN ROLLBACK COMPLETE`. Read-back `sudo iptables -S FORWARD` showed the old two bridge DROPs and all Docker entries but **no dedicated-chain jump**. The transient timer was no longer loaded: `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`. The candidate remains a saved file, and the detached chain may remain defined in the kernel. **The original persistent firewall startup script was not changed.**

**Remaining work:** Repeat untested directions under dedicated-chain protection if required, finish other service connectivity checks, consider safe ongoing chain-update/persistence behavior, then seek explicit approval before permanent deployment. IPv6, TCP/UDP, management bypass, TrueNAS reboot persistence and Suricata mirror coverage remain open.

[Detailed validation V-63–V-69](validation.md#module-06--dedicated-cyberblue_zones-chain-test-2026-10-20).

## Module 06 — Hardened Persistent V2 Reversible Test (2026-10-20)

**Status:** Controlled activation, exact-chain verification, repeat execution and rollback **PASS** for the tested `iptables-nft` environment. **Not registered as a TrueNAS POSTINIT startup script; not yet persistent.** This follows the earlier dedicated-chain packet-drop trial rather than replacing its evidence.

### Preparation and saved files

- The new `/mnt/Pool1/CyberBlue/cyberblue-firewall-persistent-candidate.sh` passed shell syntax and full-content review. A stricter `cyberblue-firewall-persistent-v2.sh` added exact expected chain-rule comparison, single-jump checking, and verification that the jump is the first `FORWARD` rule.
- An interrupted heredoc paste initially left V2 syntactically incomplete. The operator corrected the saved script; later `sudo /bin/sh -n` returned `V2 SYNTAX PASS`, and the complete contents were reviewed.
- Saved `cyberblue-firewall-persistent-v2.pre-hardening.bak` using `cp -p`. A targeted edit changed the detached-chain path to **reuse an exact already-populated chain**, populate only an empty chain, and **refuse to flush** unexpected contents. Shell syntax returned `V2 HARDENED SYNTAX PASS`; readback of numbered lines 1–83 confirmed the intended safety checks.
- `sudo iptables -S CYBERBLUE_ZONES` returned a chain declaration and exactly six directional `br20`/`br30`/`br40` DROP rules, matching V2's `EXPECTED` content. This was a read-only baseline check.

### Timed activation, repeat run, and cleanup

1. Created a 15-minute transient safety timer `cyberblue-v2-rollback-20261020.timer`, invoking `/bin/sh /mnt/Pool1/CyberBlue/cyberblue-firewall-chain-rollback.sh`. `systemctl status` confirmed `active (waiting)`, scheduled for **2026-10-20 03:36:47 PDT**.
2. Executed the V2 script **only after checking that the timer was active**. Output: `CYBERBLUE CHAIN ACTIVATED`. `sudo iptables -S FORWARD` verified exactly one `-A FORWARD -j CYBERBLUE_ZONES` first, followed by the original two Victim ↔ Attack DROP rules and Docker chains/rules; policy `ACCEPT`.
3. Re-executed V2 with the same timer-active guard. Output: `CYBERBLUE CHAIN ALREADY ACTIVE AND VERIFIED`; an immediate second `iptables -S FORWARD` readback confirmed still one chain jump and unchanged original DROP/Docker rule ordering. No duplicate jump was added in this tested repeat-execution scenario.
4. Performed manual rollback: `sudo /bin/sh /mnt/Pool1/CyberBlue/cyberblue-firewall-chain-rollback.sh` printed `CYBERBLUE CHAIN ROLLBACK COMPLETE`. `sudo iptables -S FORWARD` then showed **no** `CYBERBLUE_ZONES` jump, while retaining the original `br40→br30` and `br30→br40` DROP rules and Docker entries.
5. Stopped the transient timer; `systemctl show cyberblue-v2-rollback-20261020.timer -p LoadState -p ActiveState -p SubState --no-pager` showed `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`.

**Limits:** This V2 window was a script-behavior trial only, with **no additional Kali/Ubuntu/Windows probe test**. It demonstrates neither full protocol/IPv6 enforcement nor persistence following host reboot or Docker/Tailscale changes. Exact-rule checks intentionally fail if unexpected live configuration is detected and **do not repair** out-of-order rules. Additional startup-order and failure-path review is necessary before replacing the registered POSTINIT firewall script. The approved management-LAN Windows→Kali TCP/22 exception is separate and unchanged.

[Validation V-72–V-76](validation.md#module-06--hardened-persistent-startup-v2-controlled-trial-2026-10-20).

## Module 06 — Docker Startup Dependency Review and DOCKER-USER Trial (2026-10-20)

**Status:** Read-only startup and ruleset inspection plus a reversible `DOCKER-USER` attachment **PASS**; **not deployed permanently**. The original TrueNAS POSTINIT job ID 1 remains enabled with a **30-second timeout**, and still invokes `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh`. The original two-rule script matched the saved `cyberblue-firewall.sh.pre-br20-20261019.bak` byte-for-byte.

**V2 preparation (no firewall invocation):** Backed up `cyberblue-firewall-persistent-v2.sh` to `cyberblue-firewall-persistent-v2.pre-wait.bak`; changed twelve `"$IPT" -w` invocations to `"$IPT" -w 2`. Both syntax and all twelve replacement locations were verified. This caps each lock acquisition wait, **not** the total runtime under a 30-second POSTINIT timeout. The saved V2 still targets `FORWARD` directly and was **not** installed as the startup script.

**Boot-order concern:** The current-boot journal showed `ix-postinit.service` start/end at **2026-10-09 12:11:57/12:11:58 PDT** and `docker.service` starting at **12:12:01**, ready at **12:12:09**. `ix-postinit.service` calls `midclt call -job initshutdownscript.execute_init_tasks POSTINIT`; its displayed dependencies did not order it after Docker. Docker is active and socket-activated (`docker.socket`); `/etc/systemd/system/docker.service.d/override.conf` configures `ExecStartPost=/bin/sh -c "iptables -P FORWARD ACCEPT && ip6tables -P FORWARD ACCEPT"`. This establishes that POSTINIT finished before Docker on the observed boot, **not** that the CyberBlue script definitely ran or that all reboots order identically. Docker lifecycle changes could affect policy/order; a durable post-start verification or equivalent mechanism is still needed.

**Baseline investigation:** IPv4 nft-backed `FORWARD` retained the two original Victim↔Attack DROPs ahead of `DOCKER-USER` and other Docker rules. The separate `iptables-legacy` `FORWARD` referenced Tailscale `ts-forward`, containing its mark/accept/source-drop/accept rules; no legacy edits were performed. Global IPv6 forwarding and each of `br20`, `br30`, `br40` interface forwarding returned `0`; IPv6 `FORWARD` has policy ACCEPT and a Docker jump, **no CyberBlue IPv6 drops**. Distinct bridge-port memberships: `br20` (`vnet12`, `vnet9`), `br30` (`vnet11`, `vnet13`, `vnet8`), `br40` (`vnet10`, `vnet15`). The unavailable `bridge` CLI was replaced with read-only `/sys/class/net/*/brif` inspection.

**Docker candidate and rollback files:**

- `/mnt/Pool1/CyberBlue/cyberblue-docker-user-candidate.sh`: verifies exact six-rule `CYBERBLUE_ZONES` contents; requires the initial `DOCKER-USER` chain to contain only `RETURN`; rejects pre-existing references; inserts a single `-I DOCKER-USER 1 -j CYBERBLUE_ZONES`. Saved file contents and shell syntax reviewed.
- `/mnt/Pool1/CyberBlue/cyberblue-docker-user-rollback.sh`: removes matching `-j CYBERBLUE_ZONES` references **from DOCKER-USER only**, with `iptables -w 2`. File contents and shell syntax reviewed. It leaves the six-rule dedicated chain intact.
- The `CYBERBLUE_ZONES` chain retained all six correct DROP rules **and historical counters** from prior tests.

**Temporary activation and recovery:** A 15-minute systemd transient `cyberblue-docker-user-rollback-20261020.timer` was verified `active (waiting)`, due **2026-10-20 04:19:22 PDT**. After checking that timer active, the candidate returned `CYBERBLUE DOCKER-USER CHAIN ACTIVATED`. `iptables -S DOCKER-USER` showed the CyberBlue jump **before** `RETURN`; `iptables -S FORWARD` showed the two original bridge DROPs, then `DOCKER-USER`, then Docker entries. Readback of `iptables -nvx -L CYBERBLUE_ZONES` gave **historical** counts 4/336B for `br20→br30`, `br20→br40`, `br40→br20`, zeros otherwise; **no new guest ICMP probes were issued during this window**. Operator manually executed rollback and got `CYBERBLUE DOCKER-USER ROLLBACK COMPLETE`. Independent `iptables -S DOCKER-USER` readback showed just `-j RETURN`. Timer was stopped, then verified `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`.

**Remaining:** Before permanent enforcement, test fresh traffic through the `DOCKER-USER` path with timer/rollback protections, address Docker post-start/restart ordering and failures, verify startup-time behavior, retest preservation of Docker/Tailscale/management access, and obtain operator approval to modify the persistent TrueNAS startup path. IPv6, management-LAN/dual-homed bypass, bridged Layer-2, host traffic and full IDS mirroring remain out of scope of this IPv4 forwarding test. The operator-approved Windows `192.168.1.151` → Kali `192.168.1.91:22/tcp` management path is still separate.

[Validation V-77–V-85](validation.md#module-06--docker-compatible-docker-user-trial-2026-10-20).

## Module 06 — Fresh Six-Direction IPv4 ICMP Validation and Rollback (2026-10-20)

**Outcome:** All six test directions produced four ICMP probes, zero responses and matching +4 IPv4 DROP-counter increments. **Four SOC-related paths matched the dedicated chain under `DOCKER-USER`; the two Victim ↔ Attack paths were stopped earlier by the original direct `FORWARD` DROP rules.** This was a distinct test window following the earlier structure-only Docker trial.

**Setup / safety:** A new transient 30-minute `cyberblue-docker-traffic-rollback-20261020.timer` was created, independently checked `active (waiting)`, due **04:49:27 PDT**. The candidate was only executed after a positive timer-active guard and printed `CYBERBLUE DOCKER-USER CHAIN ACTIVATED`. `iptables -S DOCKER-USER` confirmed the `CYBERBLUE_ZONES` jump at the top before `RETURN`. Baseline `CYBERBLUE_ZONES` counters were 4/336 B for SOC→Victim, SOC→Attack and Attack→SOC, and zero in the other three directions.

| Path | Guest-side temporary routing and probe | TrueNAS DROP evidence | Enforcement location |
|---|---|---|---|
| SOC Ubuntu `10.10.20.10` → Attack Kali `10.10.40.20` | Ubuntu `ens4`, /32 via `10.10.20.1`; ping -4 sourced from `10.10.20.10`: 4 sent, 0 received | `br20→br40` 4→8 packets, 336→672 B | `CYBERBLUE_ZONES` via `DOCKER-USER` |
| Attack Kali `10.10.40.20` → SOC Ubuntu `10.10.20.10` | Kali `eth0`, /32 via `10.10.40.1`; ping sourced from `10.10.40.20`: 4 sent, 0 received | `br40→br20` 4→8 packets, 336→672 B | `CYBERBLUE_ZONES` via `DOCKER-USER` |
| SOC Ubuntu `10.10.20.10` → Victim Windows `10.10.30.10` | Ubuntu `ens4`, /32 via `10.10.20.1`; 4 sent, 0 received | `br20→br30` 4→8 packets, 336→672 B | `CYBERBLUE_ZONES` via `DOCKER-USER` |
| Victim Windows `10.10.30.10` → SOC Ubuntu `10.10.20.10` | Windows `Ethernet` index 3, ActiveStore /32 via `10.10.30.1`; `ping.exe -S 10.10.30.10 -n 4`, 4 sent, 0 received | `br30→br20` 0→4 packets, 0→240 B | `CYBERBLUE_ZONES` via `DOCKER-USER` |
| Attack Kali `10.10.40.20` → Victim Windows `10.10.30.10` | Kali `eth0`, /32 via `10.10.40.1`; 4 sent, 0 received | `br40→br30` 0→4 packets, 0→336 B | **Original direct `FORWARD` DROP** |
| Victim Windows `10.10.30.10` → Attack Kali `10.10.40.20` | Windows `Ethernet` index 3, ActiveStore /32 via `10.10.30.1`; 4 sent, 0 received | `br30→br40` 0→4 packets, 0→240 B | **Original direct `FORWARD` DROP** |

**Route hygiene:** Linux probes used `EXIT` traps to delete test /32 routes; read-only exact-route queries afterward returned no entries for Ubuntu destinations `10.10.40.20/32` and `10.10.30.10/32` and Kali destinations `10.10.20.10/32` and `10.10.30.10/32`. Windows PowerShell tests used `try/finally` with `New-NetRoute -PolicyStore ActiveStore` and `Remove-NetRoute`, displaying `TEMP ROUTE REMOVED`. Read-only `Get-NetRoute` queries confirmed both named Windows /32 destinations absent after testing. One initial command accidentally entered on the TrueNAS host failed with `Cannot find device "ens4"` before the intended packet probes, after which the SOC guest interface was positively verified.

**Restoration:** Before the 04:49:27 PDT deadline, manually executed `cyberblue-docker-user-rollback.sh` and received completion output. Independent `iptables -S DOCKER-USER` showed only `-N DOCKER-USER` and `-A DOCKER-USER -j RETURN`. Stopped the 30-minute timer; `systemctl show` returned `LoadState=not-found`, `ActiveState=inactive`, `SubState=dead`. Exact final Windows `10.10.40.20/32` route query returned no output. Original TrueNAS POSTINIT startup script was **not modified**.

**Not established:** These are scoped **IPv4 ICMP routing** tests, not full protocol coverage, not a TrueNAS/Docker restart or reboot persistence test, and not proof of full L2/IPv6/management network isolation. The original direct `FORWARD` DROP rules protect Victim ↔ Attack in these tests; four SOC directions use `DOCKER-USER`. A read-only final `FORWARD` chain check after this rollback is still outstanding. Next work: verify baseline, design service-lifecycle-safe persistence, retest Docker/Tailscale and approved Windows management `192.168.1.151` → Kali management `192.168.1.91:22/tcp`, obtain explicit operator approval before enabling a permanent implementation.

[Validation V-86–V-95](validation.md#module-06--fresh-six-direction-ipv4-icmp-test-via-docker-user-2026-10-20).

**Final post-rollback `FORWARD` readback (2026-10-20):** Operator ran `sudo /usr/sbin/iptables -S FORWARD` after the temporary `DOCKER-USER` jump was removed and timer confirmed inactive. Output retained `-P FORWARD ACCEPT`, the original `br40→br30` and `br30→br40` direct DROP rules, then `DOCKER-USER`, `DOCKER-ISOLATION-STAGE-1` and all displayed Docker forwarding entries. No `FORWARD → CYBERBLUE_ZONES` jump was present. This closes the previously pending *read-only baseline-check* item, **not** the Docker lifecycle or permanent persistence tasks. [V-96](validation.md#module-06--fresh-six-direction-ipv4-icmp-test-via-docker-user-2026-10-20).

**Post-test approved management SSH reachability (2026-10-20):** Windows 11 `Test-NetConnection -ComputerName 192.168.1.91 -Port 22 -InformationLevel Detailed` returned `InterfaceAlias: Ethernet 3`, `SourceAddress: 192.168.1.151`, `RemoteAddress: 192.168.1.91`, `RemotePort: 22`, `TcpTestSucceeded: True` after the six-direction firewall test and rollback. This verifies the previously approved Windows-management → Kali-management TCP connection is still available, **not** SSH login or access control over the lab bridges. No additional firewall rule was installed. [Validation V-97](validation.md#module-06--fresh-six-direction-ipv4-icmp-test-via-docker-user-2026-10-20).
