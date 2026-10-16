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
