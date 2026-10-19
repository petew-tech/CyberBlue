# Project 05 — Suricata Network Detection & Wazuh Integration

## Overview

This project validates network intrusion-detection telemetry from Suricata on Linux Mint through the Wazuh agent to the centralized Wazuh SIEM. The project includes a custom ICMP detection rule, centralized alert verification, and a controlled Suricata process-failure/recovery test.

## Lab Components

| Component | Configuration / role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization host |
| Linux Mint 22.3 | Suricata sensor and Wazuh agent |
| Suricata | Version 7.0.3 |
| Suricata capture interface | `ens4`, `10.10.40.10/24` |
| Linux Mint management interface | `ens3`, `192.168.1.92/24` |
| Isolated lab bridge | TrueNAS `br40`, `10.10.40.1/24` |
| Wazuh manager | `siem-01`, `192.168.1.213` |
| Wazuh agent | `cyberblue-suricata`, agent ID `003` |
| Custom Suricata rule | SID `1000001` |
| Wazuh Suricata alert rule | Rule ID `86601`, level 3 |

## Implementation Summary

- Configured Suricata to capture on Linux Mint's isolated-network interface, `ens4`.
- Corrected the capture-interface configuration from the unavailable `eth0` name to `ens4`.
- Loaded the Suricata rule set and added a local ICMP echo-request detection rule for traffic from `10.10.40.10` to `10.10.40.1`.
- Confirmed Suricata generated EVE JSON alert records for custom SID `1000001`.
- Installed and configured the Wazuh agent on Linux Mint and confirmed connection to the manager at `192.168.1.213:1514/tcp`.
- Configured Wazuh log collection for `/var/log/suricata/eve.json`.
- Verified the Suricata alert in Wazuh alerts and the Threat Hunting dashboard.
- Tested automatic Suricata recovery by terminating the main process with SIGKILL. Systemd restarted the service and Suricata resumed generating alerts.
- Inspected TrueNAS `br40` and recorded the remaining network-visibility limitation: traffic delivered to the sensor is validated, but visibility into traffic between other VMs on the bridge is not established.

## Validation Summary

| Test | Observed result | Status |
|---|---|---|
| Suricata service startup configuration | `enabled` | PASS |
| Suricata service state | `active` | PASS |
| Detection rule loading after recovery | 53,144 rules loaded; 0 failed | PASS |
| ICMP test connectivity | 4 sent, 4 received, 0% loss | PASS |
| Custom Suricata detection | SID `1000001` generated an ICMP alert | PASS |
| Wazuh agent connectivity | Connected to `192.168.1.213:1514/tcp` | PASS |
| Wazuh centralized alert | Rule `86601`, agent `003`, source `/var/log/suricata/eve.json` | PASS |
| Automatic service recovery | Main PID changed from `3502` to `13422`; restart count became `1` | PASS |
| Detection after recovery | Fresh SID `1000001` alert reached Wazuh at approximately 09:29 CDT | PASS |
| Capture statistics after recovery | 278 packets, 0 kernel drops, 0 capture errors in sampled stats event | PASS |
| Full `br40` traffic visibility | Not demonstrated; deferred follow-up | OPEN |

## Scope and Limitations

The validated pipeline is: test traffic delivered to Linux Mint `ens4` → Suricata custom rule → EVE JSON → Wazuh agent → Wazuh manager and dashboard.

The TrueNAS bridge inspection showed `br40` UP at `10.10.40.1/24`, with one attached host-side virtual interface (`vnet4`) during the check. No ingress `tc` filters were reported on `vnet4` or `br40`. These observations do not prove that no other mirroring mechanism exists, but they do mean that mirroring and visibility into traffic between other VMs have not been demonstrated. A separate follow-up will assess the appropriate method for broader bridge visibility.

The capture statistics are a point-in-time sample, not proof of zero packet loss under all traffic loads. Promiscuous mode on the guest interface does not by itself guarantee visibility into traffic not delivered to that interface.

## Related Documentation

- [Build notes](build-notes.md)
- [Validation record](validation.md)
- [Project 04 — Centralized Logging & SIEM Foundations](../04-centralized-logging/README.md)
- [Project 02 — Virtual Networking & Segmentation](../02-networking-segmentation/README.md)

---

## Module 06 — Staged Network Migration (2026-10-18)

**Status: VALIDATED for Ubuntu, Windows and Kali guest network migrations; three-zone isolation and monitoring coverage remain OPEN.** This is Module 06 of **Project 05**, not a separate Project 06.

This module released Ubuntu's old `br30` IPv4 address `10.10.30.10/24`, preserved its management and SOC networking, migrated Windows 11 from temporary Victim address `10.10.30.11/24` to `10.10.30.10/24`, and migrated Kali from `br30` / `10.10.30.20/24` to `br40` / `10.10.40.20/24`.

| Check | Verified result | Status |
|---|---|---|
| Ubuntu Netplan update | `netplan generate` succeeded; `netplan try --timeout 300` accepted using TrueNAS console fallback | PASS |
| Ubuntu management/SOC addressing | `mgmt0 192.168.1.243/24`, `ens4 10.10.20.10/24`, default route via `192.168.1.1` | PASS |
| Ubuntu old `br30` address/route | `10.10.30.10/24` and `10.10.30.0/24` route removed, including after Ubuntu guest reboot | PASS |
| Windows new Victim address | `Ethernet 10.10.30.10/24` Preferred, old `10.10.30.11` absent | PASS |
| Windows management | `Ethernet 3 192.168.1.151/24`, default via `192.168.1.1` | PASS |
| Windows reboot persistence | Both required addresses and routes retained | PASS |
| Windows gateway connectivity after reboot | Victim `10.10.30.10 → 10.10.30.1` and management `192.168.1.151 → 192.168.1.1`, each 4/4 replies, 0% loss | PASS |
| Kali bridge reassignment and permanent MACs | TrueNAS VM `6`, device `42` now `br40`, MAC `00:a0:98:76:2d:fd`; device `16` management on `enp4s0`, MAC `00:a0:98:5b:5f:c8` | PASS |
| Kali address/route persistence | Attack `10.10.40.20/24` on `eth0`; management `192.168.1.91/24` on `eth1`; default through management; no `10.10.30.0/24` route after reboot | PASS |
| Kali post-reboot connectivity and SSH | Attack `10.10.40.20 → 10.10.40.1` and management `192.168.1.91 → 192.168.1.1`, each 4/4 replies, 0% loss; SSH reconnect successful | PASS |

**Verified evidence:** [Ubuntu applied routing](evidence/project06-netplan-routing-validated.png), [Ubuntu post-reboot](evidence/project06-netplan-post-reboot-validation.png), [Windows pre-reboot](evidence/module06-windows-victim-ip-pre-reboot.png), [Windows post-reboot](evidence/module06-windows-victim-post-reboot-validation.png). The existing `project06-` screenshot filenames are retained to avoid breaking uploaded links; this work belongs to Project 05 Module 06.

**Not established:** Ubuntu's unused VirtIO NIC was not detached from TrueNAS `br30` (after reboot it appeared as `ens3`, with no IPv4 address). Bidirectional **IPv4 ICMP** forwarding denial between `br30` and `br40` is now verified in controlled tests dated 2026-10-19 (details below). TrueNAS **host** reboot persistence of `br20`, SOC zone policy, IPv6 and alternate management-LAN paths, and comprehensive Suricata traffic mirroring remain open. Successful gateway pings do not prove full segmentation.

See [build notes](build-notes.md#module-06--ubuntu-and-windows-ip-migration-2026-10-18) and [validation](validation.md#module-06--staged-network-migration-validation-2026-10-18).

**Kali evidence:** [pre-reboot configuration](evidence/module06-kali-attack-network-pre-reboot.png) and [post-reboot configuration](evidence/module06-kali-attack-network-post-reboot.png). NetworkManager initially bound management to `eth0` and Attack to `eth1` after NIC ordering changed, causing SSH loss. The guest was recovered using the TrueNAS SPICE console; both profiles were rebound to explicitly configured stable MAC addresses and validated after another reboot. See [Kali recovery build notes](build-notes.md#kali-attack-zone-migration--nic-order-recovery-and-mac-persistence-2026-10-18).

## Module 06 — Controlled Victim ↔ Attack IPv4 Firewall Verification (2026-10-19)

**Result: PASS — tested routed ICMP in both directions was dropped by the TrueNAS `FORWARD` firewall rules.** This is a scoped update to the Module 06 guest-migration work, not proof of comprehensive three-zone isolation.

- TrueNAS `sudo iptables -S FORWARD` showed `-P FORWARD ACCEPT` and explicit `-i br40 -o br30 -j DROP` / `-i br30 -o br40 -j DROP` following Docker-related rules. A warning revealed parallel `iptables-legacy` tables. `sudo iptables-legacy -S FORWARD` showed a Tailscale `ts-forward` jump; `sudo iptables-legacy -S ts-forward` contained Tailscale mark/ACCEPT/DROP rules but no explicit `br20`/`br30`/`br40` rules.
- TrueNAS `cat /proc/sys/net/ipv4/ip_forward` returned `1`. Prior DROP counters were 4 packets / 336 bytes in each direction; these earlier hits were not accepted as current test proof.
- **Attack → Victim:** Kali normally routed to Windows via its management NIC `eth1`. A temporary host route redirected **only** `10.10.30.10/32` via `10.10.40.1 dev eth0` (source `10.10.40.20`); `ip -4 route get 10.10.30.10` confirmed the Attack path. Four ICMP probes returned 0 replies. TrueNAS `br40 → br30` DROP count rose **4 → 8 packets, 336 → 672 bytes**, with the reverse counter unchanged. Kali temporary route was deleted, and the normal route via management `192.168.1.1 dev eth1` was verified restored.
- **Victim → Attack:** Windows `Find-NetRoute -RemoteIPAddress 10.10.40.20` initially selected management `Ethernet 3 192.168.1.151` via `192.168.1.1`. Elevated PowerShell added a **nonpersistent**, destination-specific `10.10.40.20/32` route via `10.10.30.1` on Victim `Ethernet`. `Find-NetRoute` verified source `10.10.30.10`, interface index `3`, and next hop `10.10.30.1`. A four-probe `ping -4 -S 10.10.30.10 -n 4 10.10.40.20` timed out; TrueNAS `br30 → br40` DROP count rose **4 → 8 packets, 336 → 576 bytes**, without further change to the other rule. The host route was removed and `Find-NetRoute` confirmed normal management routing returned (`Ethernet 3`, source `192.168.1.151`, next hop `192.168.1.1`).

**Scope and limitations:** Exact paired four-packet increments corroborate firewall enforcement for the controlled IPv4 ICMP flows. Tests did not examine other protocols, SOC bridge `br20`, IPv6, other virtual bridge/management-LAN bypass paths, policy persistence across TrueNAS reboot, or complete Suricata observation. Raw CLI/PowerShell output was supplied during the guided validation; no new screenshots have been asserted as uploaded. See [V-46–V-51](validation.md#module-06--victim--attack-ipv4-isolation-tests-2026-10-19) and [test procedure](build-notes.md#module-06--victim--attack-ipv4-firewall-test-procedure-2026-10-19).
