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

## Module 06 — Three-Zone Routed IPv4 ICMP Trial and Baseline Restoration (2026-10-19)

**Scoped result: PASS — six of six tested directional routed IPv4 ICMP paths blocked, corroborated by TrueNAS DROP counters.** This combines the earlier two Victim ↔ Attack tests on the existing baseline with four SOC-related directions tested under a **temporary** six-rule candidate.

| Tested direction | DROP chain match | Current trial evidence |
|---|---|---|
| Attack → Victim | `br40 → br30` | PASS — earlier baseline test; +4 packets |
| Victim → Attack | `br30 → br40` | PASS — earlier baseline test; +4 packets |
| SOC → Victim | `br20 → br30` | PASS — 4 packets / 336 bytes |
| SOC → Attack | `br20 → br40` | PASS — 4 packets / 336 bytes |
| Attack → SOC | `br40 → br20` | PASS — 4 packets / 336 bytes |
| Victim → SOC | `br30 → br20` | PASS — 4 packets / 240 bytes |

Each test used an explicitly verified, destination-specific temporary guest route to ensure traffic traversed the relevant TrueNAS bridge pair rather than the shared management LAN. After each test the guest route was removed and normal management routing was verified restored.

**Safety and current state:** Original POSTINIT firewall script backed up and left unchanged. Six-rule candidate was tested in three short windows, with automatic rollback timers, explicit manual rollback, `iptables` readback, and verification that each timer became inactive. **At completion, the active firewall reverted to the original two Victim ↔ Attack DROP rules; there are no active SOC DROP rules.** Do not mistake demonstrated candidate behavior for persistent enforcement.

**Still OPEN (Module 06):** persistent three-zone default-deny deployment and authorized exceptions; TCP/UDP and other protocols; IPv6; dual-homed guests and management-LAN bypass paths; bridge Layer-2 and host traffic; TrueNAS `br20`/firewall host-reboot persistence; comprehensive Suricata packet capture/mirroring.

[Complete V-52–V-60 validation](validation.md#module-06--three-zone-scoped-ipv4-icmp-firewall-trial-2026-10-19) · [Candidate, test and rollback procedure](build-notes.md#module-06--temporary-soc-default-deny-candidate-and-rollback-procedure-2026-10-19)

**Approved management-LAN access (2026-10-19):** The operator elected to retain Windows `192.168.1.151` → Kali `192.168.1.91` TCP/22 SSH reachability on the separate management subnet. Windows `Test-NetConnection` verified TCP connectivity via `Ethernet 3`; no SSH authentication was tested. This is **not** permission for arbitrary Victim `br30` → Attack `br40` traffic, and **no live firewall configuration was changed**. Future management access controls must explicitly account for this limited approved path. [Validation V-61–V-62](validation.md#module-06--approved-management-lan-ssh-access-2026-10-19).

## Module 06 — Dedicated CYBERBLUE_ZONES Chain Trial (2026-10-20)

**Temporary test: PASS — three of three exercised routed IPv4 ICMP directions were dropped by a dedicated, rollback-protected firewall chain.** This extends the earlier six-direction candidate evidence; it does **not** mean the full three-zone policy is permanently deployed.

- `iptables-nft` chain `CYBERBLUE_ZONES` was populated with all six directional `br20`/`br30`/`br40` DROP rules before inserting the jump at `FORWARD` rule 1. Original Victim ↔ Attack DROP rules remained at positions 2 and 3, with Docker entries following.
- **Attack → SOC:** Kali `10.10.40.20` to Ubuntu `10.10.20.10`: 4 ICMP packets sent, 0 received; `br40→br20` DROP counter **4 packets / 336 B**.
- **SOC → Attack:** Ubuntu `10.10.20.10` to Kali `10.10.40.20`: 4 sent, 0 received; `br20→br40` DROP counter **4 packets / 336 B**.
- **SOC → Victim:** Ubuntu `10.10.20.10` to Windows `10.10.30.10`: 4 sent, 0 received; `br20→br30` DROP counter **4 packets / 336 B**.
- Each probe used a temporary destination-specific guest route with a shell exit-cleanup trap; independent route readback was not recorded during this window. The remaining three directional chain rules were installed but not retested under this implementation.
- A verified 15-minute transient systemd rollback safeguard was armed. The operator manually removed the `FORWARD` jump after testing; readback showed only the original two Victim ↔ Attack DROP rules and unchanged Docker entries. The timer was verified **not loaded / inactive / dead**.

**Current deployed state:** The original two-rule TrueNAS POSTINIT script was not changed, and the six-rule chain is **not attached**. Full persistent three-zone default deny, cross-protocol/IPv6 protection, management-LAN restrictions, Layer-2/host paths, TrueNAS host reboot and full Suricata mirror coverage remain **OPEN**. The previously approved Windows management IP `192.168.1.151` → Kali management IP `192.168.1.91:22/tcp` exception is separate from these lab-bridge rules.

[Validation V-63–V-69](validation.md#module-06--dedicated-cyberblue_zones-chain-test-2026-10-20) · [Procedure and rollback evidence](build-notes.md#module-06--dedicated-chain-candidate-and-safe-rollback-2026-10-20)

## Module 06 — Hardened V2 Startup Candidate and Safe Restore (2026-10-20)

**Reversible test PASS; NOT deployed as persistent startup policy.** A new `/mnt/Pool1/CyberBlue/cyberblue-firewall-persistent-v2.sh` candidate was backed up, syntax-checked and reviewed. Its hardened design checks the exact six directional rules in `CYBERBLUE_ZONES`, rejects unexpected chain contents/references, and does not flush a populated detached chain.

During a fresh 15-minute rollback-protected window, V2 attached `CYBERBLUE_ZONES` first in TrueNAS IPv4 `FORWARD`, ahead of the original Victim ↔ Attack direct DROP rules and Docker forwarding entries. A second V2 run reported `CYBERBLUE CHAIN ALREADY ACTIVE AND VERIFIED`; readback confirmed **one** jump and no duplicate. Manual rollback removed the jump and restored the original two-rule baseline; the systemd rollback timer was verified **not-found/inactive/dead**. **No new guest-packet probes were run in the V2 idempotency window**; the three earlier directed ICMP checks remain separate evidence.

Independent post-test route reads confirmed Ubuntu and Kali's temporary /32 host routes absent and normal management route selection restored (V-70); the detached chain retained its six rules for inspection (V-71). The original TrueNAS POSTINIT startup script remains unchanged. **Persistent full three-zone isolation, reboot/restart ordering, exception enforcement outside the bridge rules, other protocols, IPv6 and full Suricata monitoring remain OPEN.**

[Detailed V2 validation V-72–V-76](validation.md#module-06--hardened-persistent-startup-v2-controlled-trial-2026-10-20) · [V2 build notes](build-notes.md#module-06--hardened-persistent-v2-reversible-test-2026-10-20)

## Module 06 — Docker-Compatible Firewall Trial (2026-10-20)

**Temporary structure and manual rollback PASS; persistent enforcement and fresh packet validation remain OPEN.** TrueNAS POSTINIT (enabled original two-rule script, 30-second timeout) completed before Docker initialization on the observed October 9 boot. Docker's existing service override sets IPv4/IPv6 `FORWARD` default policy ACCEPT at service startup. Docker's `DOCKER-USER` chain had just `RETURN`, providing a candidate insertion point before that rule without flushing Docker or Tailscale settings.

A separate `cyberblue-docker-user-candidate.sh` was syntax/content reviewed and run only with a verified active 15-minute automatic rollback timer. It inserted `DOCKER-USER → CYBERBLUE_ZONES` before `RETURN`; independent `FORWARD` readback confirmed the original Victim ↔ Attack drops and Docker rules remained. **No new guest traffic was probed**; inspected three nonzero DROP counters were **historical** from the prior dedicated-chain test and must not be reported as new proof.

The operator manually executed `cyberblue-docker-user-rollback.sh`; readback restored `DOCKER-USER` to just `RETURN`, and the scheduled timer was stopped and verified `not-found/inactive/dead`. The V2 candidate was also backed up and edited to use 12 bounded `iptables -w 2` calls (syntax and occurrence check PASS), but **was not** registered or deployed at POSTINIT. Global and bridge-specific IPv6 routing flags were observed off; IPv6 `FORWARD` policy remained ACCEPT, so IPv6 protection is **not** established. Existing Tailscale legacy rules were inspected and left unchanged.

**Actual state:** Only original Victim ↔ Attack direct IPv4 DROP rules are the deployed CyberBlue policy. Full three-zone persistence after TrueNAS/Docker restarts, new packet-block verification through `DOCKER-USER`, IPv6, management-LAN and dual-homed paths, Layer-2/host traffic and full Suricata mirror coverage remain **OPEN**. The separately approved Windows management IP → Kali management IP TCP/22 path remains unchanged.

[Validation V-77–V-85](validation.md#module-06--docker-compatible-docker-user-trial-2026-10-20) · [Build notes](build-notes.md#module-06--docker-startup-dependency-review-and-docker-user-trial-2026-10-20).
