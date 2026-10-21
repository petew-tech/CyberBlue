# Project 05 — Suricata Network Detection & Wazuh Integration

## Overview

This project validates network intrusion-detection telemetry from Suricata on Linux Mint through the Wazuh agent to the centralized Wazuh SIEM. The project includes **two custom ICMP detection rules**: Attack-network SID `1000001` (`10.10.40.10 → 10.10.40.1`) and Victim-network SID `1000002` (`10.10.30.10 → 10.10.30.1`). Both were validated with centralized Wazuh alert evidence; the Victim test used a temporary scoped mirror that was subsequently removed. A controlled Suricata process-failure/recovery test was also completed. Comprehensive or persistent network-wide mirroring remains **OPEN**.

## Lab Components

| Component | Configuration / role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization host |
| Linux Mint 22.3 | Suricata sensor and Wazuh agent |
| Suricata | Version 7.0.3 |
| Suricata Attack capture interface | `ens4`, `10.10.40.10/24`, `br40` |
| Suricata Victim capture interface | `ens5`, no guest IPv4 address, `br30` (passive monitoring; unicast only when explicitly mirrored in tests) |
| Linux Mint management interface | `ens3`, `192.168.1.92/24` |
| Linux Mint SOC network interface | `ens6`, `10.10.20.11/24`, TrueNAS `br20`; no default gateway (added October 17) |
| TrueNAS SOC bridge | `br20`, `10.10.20.1/24` — SOC network |
| TrueNAS Victim bridge | `br30`, `10.10.30.1/24` — Windows Victim and passive Suricata capture NIC |
| TrueNAS Attack bridge | `br40`, `10.10.40.1/24` — Kali Attack and Suricata `ens4` |
| Wazuh manager | `siem-01`, `192.168.1.213` |
| Wazuh agent | `cyberblue-suricata`, agent ID `003` |
| Attack ICMP rule | SID `1000001`: `10.10.40.10 → 10.10.40.1` |
| Victim ICMP rule | SID `1000002`: `10.10.30.10 → 10.10.30.1` (added and validated October 20) |
| Wazuh Suricata alert rule | Rule ID `86601`, level 3 |

## Implementation Summary

- Initially configured Suricata capture on Linux Mint `ens4`; later enabled dual AF_PACKET monitoring on `ens4` (Attack) and `ens5` (Victim).
- Corrected the capture-interface configuration from the unavailable `eth0` name to `ens4`.
- Loaded the Suricata rule set and added local ICMP echo-request SID `1000001` for `10.10.40.10 → 10.10.40.1`; on October 20 added SID `1000002` for `10.10.30.10 → 10.10.30.1` after backing up the local rule file.
- Confirmed Suricata generated EVE JSON alerts for SID `1000001` on `ens4` and, with a time-bounded TrueNAS test mirror, four SID `1000002` alerts on `ens5`.
- Installed and configured the Wazuh agent on Linux Mint and confirmed connection to the manager at `192.168.1.213:1514/tcp`.
- Configured Wazuh log collection for `/var/log/suricata/eve.json`.
- Verified both custom Suricata signatures in Wazuh manager alerts and operator-confirmed matching events in Threat Hunting; the October 20 SID `1000002` test produced four exact matching Wazuh manager alerts.
- Tested automatic Suricata recovery by terminating the main process with SIGKILL. Systemd restarted the service and Suricata resumed generating alerts.
- Inspected TrueNAS `br40` and recorded the remaining network-visibility limitation: traffic delivered to the sensor is validated, but visibility into traffic between other VMs on the bridge is not established.

## Validation Summary

| Test | Observed result | Status |
|---|---|---|
| Suricata service startup configuration | `enabled` | PASS |
| Suricata service state | `active` | PASS |
| Detection rule loading after earlier service recovery | 53,144 rules loaded; 0 failed (historical checkpoint) | PASS |
| Live ruleset after SID `1000002` addition | 53,145 rules loaded; 0 failed; Suricata test exited 0 | PASS |
| ICMP test connectivity | 4 sent, 4 received, 0% loss | PASS |
| Attack custom detection | SID `1000001` generated an ICMP alert | PASS |
| Victim custom detection | SID `1000002` generated four ICMP alerts on `ens5` under temporary mirroring | PASS |
| Wazuh agent connectivity | Connected to `192.168.1.213:1514/tcp` | PASS |
| Wazuh centralized alert | Rule `86601`, agent `003`, source `/var/log/suricata/eve.json` | PASS |
| Automatic service recovery | Main PID changed from `3502` to `13422`; restart count became `1` | PASS |
| Detection after recovery | Fresh SID `1000001` alert reached Wazuh at approximately 09:29 CDT | PASS |
| Capture statistics after recovery | 278 packets, 0 kernel drops, 0 capture errors in sampled stats event | PASS |
| Full `br40` traffic visibility | Not demonstrated; deferred follow-up | OPEN |

## Scope and Limitations

The validated baseline pipeline is: test traffic delivered to Linux Mint `ens4` → Suricata SID `1000001` → EVE JSON → Wazuh agent → Wazuh manager and dashboard. A separate October 20 test validated Windows Victim ICMP traffic mirrored **temporarily** to Mint `ens5` → Suricata SID `1000002` → EVE alert → Wazuh agent/manager → operator-confirmed dashboard matching alert. The TrueNAS test mirror was rolled back and independently checked, so continuous Victim unicast visibility is **not** established.

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

**Historical V-77–V-85 checkpoint: temporary structure and manual rollback PASS; persistent enforcement OPEN, fresh packet validation not yet performed.** The later **V-86–V-95** trial verified fresh routed IPv4 ICMP DROP counters for six tested directions (four through `DOCKER-USER → CYBERBLUE_ZONES`, two through original direct `FORWARD` rules). That follow-up did **not** deploy a persistent three-zone policy. TrueNAS POSTINIT (enabled original two-rule script, 30-second timeout) completed before Docker initialization on the observed October 9 boot. Docker's existing service override sets IPv4/IPv6 `FORWARD` default policy ACCEPT at service startup. Docker's `DOCKER-USER` chain had just `RETURN`, providing a candidate insertion point before that rule without flushing Docker or Tailscale settings.

A separate `cyberblue-docker-user-candidate.sh` was syntax/content reviewed and run only with a verified active 15-minute automatic rollback timer. It inserted `DOCKER-USER → CYBERBLUE_ZONES` before `RETURN`; independent `FORWARD` readback confirmed the original Victim ↔ Attack drops and Docker rules remained. **No new guest traffic was probed**; inspected three nonzero DROP counters were **historical** from the prior dedicated-chain test and must not be reported as new proof.

The operator manually executed `cyberblue-docker-user-rollback.sh`; readback restored `DOCKER-USER` to just `RETURN`, and the scheduled timer was stopped and verified `not-found/inactive/dead`. The V2 candidate was also backed up and edited to use 12 bounded `iptables -w 2` calls (syntax and occurrence check PASS), but **was not** registered or deployed at POSTINIT. Global and bridge-specific IPv6 routing flags were observed off; IPv6 `FORWARD` policy remained ACCEPT, so IPv6 protection is **not** established. Existing Tailscale legacy rules were inspected and left unchanged.

**Deployed policy / subsequent verification:** Only original Victim ↔ Attack direct IPv4 DROP rules are the deployed CyberBlue policy. Fresh scoped IPv4 ICMP packet-block testing through `DOCKER-USER → CYBERBLUE_ZONES` was subsequently completed as **PASS for four SOC-related directions** (V-86–V-95); the two Victim ↔ Attack directions matched the original direct `FORWARD` rules. The temporary `DOCKER-USER` attachment was then removed. **Still OPEN:** permanent three-zone enforcement and TrueNAS/Docker restart persistence, other protocols and IPv6, management-LAN/dual-homed bypass analysis, Layer-2/host traffic, and full Suricata mirror coverage. The separately approved Windows management IP → Kali management IP TCP/22 path remained available after rollback (V-97).

[Validation V-77–V-85](validation.md#module-06--docker-compatible-docker-user-trial-2026-10-20) · [Build notes](build-notes.md#module-06--docker-startup-dependency-review-and-docker-user-trial-2026-10-20).

## Module 06 — Six-Direction Docker-Compatible IPv4 ICMP Validation (2026-10-20)

**Controlled test PASS: six of six routed IPv4 ICMP directions blocked with matching fresh +4 packet DROP-counter increments.** A *second* 30-minute rollback-protected window attached `CYBERBLUE_ZONES` to `DOCKER-USER` ahead of `RETURN`; its transient timer was confirmed active before candidate activation, with rollback due **04:49:27 PDT**.

| Direction | New DROP count | Matching active rule |
|---|---:|---|
| SOC → Attack | +4 (336 bytes) | `CYBERBLUE_ZONES br20→br40` |
| Attack → SOC | +4 (336 bytes) | `CYBERBLUE_ZONES br40→br20` |
| SOC → Victim | +4 (336 bytes) | `CYBERBLUE_ZONES br20→br30` |
| Victim → SOC | +4 (240 bytes) | `CYBERBLUE_ZONES br30→br20` |
| Attack → Victim | +4 (336 bytes) | **Original direct `FORWARD br40→br30`** |
| Victim → Attack | +4 (240 bytes) | **Original direct `FORWARD br30→br40`** |

Each test used a temporary guest /32 route explicitly via the respective lab interface and four ICMP probes (zero replies). Independent queries confirmed all named test routes absent afterward. **The two Victim ↔ Attack directions do not validate dedicated-chain matching:** the original direct DROP rules take precedence.

**Post-test:** The operator manually removed the `DOCKER-USER` jump, independently read back its original `RETURN`-only configuration, stopped the timed rollback, and verified the timer `not-found/inactive/dead`. Original two-rule TrueNAS POSTINIT startup script **unchanged**; full permanent three-zone isolation **not deployed**. IPv6, TCP/UDP and other protocols, management-LAN bypass, Layer-2/host forwarding, Docker/TrueNAS restart persistence, and comprehensive Suricata mirroring remain OPEN.

[Validation V-86–V-95](validation.md#module-06--fresh-six-direction-ipv4-icmp-test-via-docker-user-2026-10-20) · [Build notes](build-notes.md#module-06--fresh-six-direction-ipv4-icmp-validation-and-rollback-2026-10-20).

**Final post-test baseline verification:** An independent `iptables -S FORWARD` readback after rollback confirmed the original two Victim ↔ Attack direct DROP rules, `FORWARD ACCEPT`, and the existing Docker jumps/rules with no direct `CYBERBLUE_ZONES` jump. `DOCKER-USER` had already been verified restored to `RETURN` only; the transient timer was verified inactive. No persistent three-zone policy was deployed. [Validation V-96](validation.md#module-06--fresh-six-direction-ipv4-icmp-test-via-docker-user-2026-10-20).

**Post-test approved management SSH verification:** Windows `192.168.1.151` (`Ethernet 3`) → Kali management `192.168.1.91:22/tcp` remained reachable after temporary firewall rollback (`Test-NetConnection: TcpTestSucceeded True`). SSH authentication and other management-LAN permissions were not tested; no exception rule was added. [Validation V-97](validation.md#module-06--fresh-six-direction-ipv4-icmp-test-via-docker-user-2026-10-20).

## Module 06 — Docker Lifecycle Discovery / V3 Read-Only Preflight (2026-10-20)

**V3 preflight PASS; no persistent policy activated.** Operator-backed-up lifecycle V1/V2 candidates were reviewed, and `cyberblue-firewall-lifecycle-v3.sh` passed syntax, readback and read-only `--check` (`CYBERBLUE V3 PREFLIGHT PASS - CHAIN DETACHED`). Existing nft-backed `CYBERBLUE_ZONES` contains all six DROP rules and remains detached. A V3 activation would require a deliberately armed rollback timer; it has **not** been executed.

Docker is active with `Restart=always` and `docker.socket` currently active and listening, though both unit files report `disabled`. The existing Docker `ExecStartPost` sets `FORWARD` defaults to ACCEPT. TrueNAS `ix-postinit.service` is `Type=oneshot`, `After=multi-user.target` without `After=docker.service`; earlier boot logs showed POSTINIT finished before Docker initialization. This does **not** provide automatic re-application on Docker restart.

**Infrastructure follow-up (deferred to Project 02):** The V-98–V-102 discovery identified the need for separately managed, Docker-lifecycle-aware CyberBlue enforcement with controlled rollback and explicit operator approval. **The subsequent Project 05 scope reset deferred this design and permanent deployment work to Project 02; it is not the next Project 05 Suricata task.** No POSTINIT registration, Docker override, permanent service or active firewall-rule change was made in this discovery phase. The original TrueNAS Victim ↔ Attack two-rule baseline remains the deployed policy. [Validation V-98–V-102](validation.md#module-06--docker-lifecycle-read-only-discovery-and-v3-preflight-2026-10-20) · [Project 02 follow-up](../02-networking-segmentation/README.md#open-follow-up--permanent-three-zone-firewall-lifecycle-2026-10-20).

## Project Scope Reset — Suricata Detection Work (2026-10-20)

**Firewall lifecycle engineering deferred to Project 02.** Module 06's temporary routed IPv4 ICMP isolation experiment, rollback and read-only V3 preflight are retained as supporting network evidence, but the systemd/Docker persistence project is **not** a prerequisite to continuing this Suricata build. The original two-rule TrueNAS firewall baseline remains unchanged; `CYBERBLUE_ZONES` is detached, and no new persistent service has been installed.

**Return to Project 05 objectives:** Verify the Suricata/Wazuh sensor health, confirm current dual-bridge capture interfaces `ens4` (Attack) and `ens5` (Victim), then plan safe and scoped traffic-visibility testing on the finalized VM topology. Previous `tc` mirroring was explicitly removed during the SOC migration on October 17, and full inter-VM visibility is **not proven**. Existing custom SID `1000001` and Wazuh rule `86601` alert validations remain historical PASS evidence, not claims of current mirror coverage.

[Project 02 open firewall follow-up](../02-networking-segmentation/README.md#open-follow-up--permanent-three-zone-firewall-lifecycle-2026-10-20).

## Project 05 — Current Suricata Visibility Check (2026-10-20)

**Sensor runtime validated; passive Victim unicast visibility remains OPEN.** Linux Mint Suricata and Wazuh services are active. Suricata PID 1056 is running AF_PACKET with two worker threads each on `ens4` (Attack, cluster 99) and `ens5` (Victim, cluster 100). Recent EVE JSON includes both interfaces and Windows multicast/broadcast activity.

During a coordinated test, Windows `10.10.30.10` successfully pinged the Victim gateway `10.10.30.1` **4/4**, but a simultaneously reported Mint `ens5` tcpdump filtered for those echo requests recorded **0 matching packets**, **0 kernel drops**. This was **a scoped observation gap at V-103–V-104**, not evidence of complete sensor failure. The earlier October 16 `tc` mirror had been removed on October 17; **no new firewall, mirror, or service changes were made during this initial check**. The planned follow-up was **subsequently completed**: TrueNAS `br30` VM port inventory and a rollback-protected, narrow Windows-to-Mint ICMP mirror were validated under **V-105–V-110**; Suricata EVE flow ingestion was separately confirmed under **V-111**. A later R3 mirror with Victim-specific SID `1000002` produced four Suricata and four matching Wazuh alerts under **V-112–V-115**. The temporary mirror was removed, with scheduled rollback and clean post-timer qdisc verification under **V-117–V-118**. **Persistent or bridge-wide unicast visibility remains OPEN.**

[Validation V-103–V-104](validation.md#project-05--sensor-runtime-and-windows-unicast-visibility-2026-10-20) · [Build notes](build-notes.md#project-05--coordinated-victim-unicast-visibility-test-2026-10-20).

## Project 05 — Scoped Victim Mirror Validation (2026-10-20)

**PASS — temporary Windows unicast mirror into Mint sensor, with verified cleanup.** TrueNAS confirmed Windows VM 11 on `vnet13` and Linux Mint VM 7 on `vnet11` under Victim bridge `br30`. A narrow `tc` ingress flower rule mirrored IPv4 ICMP echo requests `10.10.30.10 → 10.10.30.1` from `vnet13` toward `vnet11`. The mirror action recorded four packets; separate, coordinated Windows ping rounds yielded **4/4 packets** on the host-side `vnet11` capture (sequences 220–223) and **4/4 packets** on the guest-side Mint `ens5` capture (sequences 224–227), with **zero kernel drops** in each capture. Both Windows ping rounds received 4/4 gateway replies.

**Recovery PASS:** A 15-minute TrueNAS `systemd-run` rollback first executed and was verified independently. During the successful second test, the user manually executed the saved rollback script and confirmed `vnet13` returned to `noqueue` with **no `clsact`**. This removed the temporary mirror; no permanent traffic-mirroring policy was deployed.

**Status at the V-105–V-110 packet-capture checkpoint:** Those captures alone did **not yet** prove Suricata EVE ingestion or Wazuh detection. Subsequent **V-111** read-only EVE inspection established three `event_type:"flow"` records from the earlier, temporary mirrored Windows ICMP rounds; these are **flow telemetry, not alerts**. A **separate R3 mirrored ping after adding SID `1000002`** then generated four Suricata EVE `event_type:"alert"` records and four matching Wazuh alerts (**V-112–V-115**), with at least one matching event confirmed in Threat Hunting (**V-116**). **Still OPEN:** persistent mirroring and full-network/bidirectional traffic visibility. The temporary mirror was removed and its post-timer absence verified (**V-117–V-118**).

[Validation V-105–V-110](validation.md#project-05--scoped-tc-mirror-proof-packet-capture-and-recovery-2026-10-20) · [Build notes](build-notes.md#project-05--scoped-br30-unicast-mirror-and-restoration-2026-10-20).

## Project 05 — Suricata EVE Victim Flow Confirmation (2026-10-20)

**PASS — packet capture AND Suricata EVE flow ingestion for the scoped mirrored ICMP tests.** Following the successful timed test and verified manual `tc` cleanup, Linux Mint's `/var/log/suricata/eve.json` contained three `ens5` `event_type:"flow"` records for Windows `10.10.30.10 →` gateway `10.10.30.1` ICMP echo requests. Each recorded four request-direction packets; the final flow's 12:40:08–12:40:11 (-0500) window correlated with the Mint tcpdump capturing four requests, sequences 224–227. No client-direction packets were recorded in the one-way mirror flows. **This proves EVE flow telemetry, not a Suricata detection alert or Wazuh correlation.** The mirror is no longer active; persistent full-network visibility remains OPEN.

[Validation V-111](validation.md#project-05--suricata-eve-flow-ingestion-of-mirrored-victim-traffic-2026-10-20) · [Build notes](build-notes.md#project-05--mirrored-windows-icmp-suricata-eve-flow-evidence-2026-10-20).

## Project 05 — End-to-End Victim Detection to Wazuh (2026-10-20)

**PASS — controlled Windows Victim ICMP → TrueNAS scoped mirror → Linux Mint Suricata → Wazuh manager.** Preserved existing Attack local rule SID `1000001`, verified and backed up `local.rules`, and added a distinct Victim ICMP echo rule SID **1000002** for Windows `10.10.30.10` → gateway `10.10.30.1`. `suricata -T` completed with exit 0; live ruleset reloaded with **53,145 rules loaded, 0 failed/skipped**. Under verified 20-minute TrueNAS rollback protection, narrow `vnet13`→`vnet11` request-direction mirroring enabled and Windows ping returned **4/4** replies.

**Detection evidence:** Suricata `/var/log/suricata/eve.json` logged **four** `event_type:"alert"` records, `in_iface:"ens5"`, `alert.signature_id:1000002`, timestamps 2026-10-20 13:16:07–13:16:10 -0500. Wazuh `siem-01` `alerts.json` showed **four matching alerts** under Wazuh rule **86601**, level **3**, from agent **003** (`cyberblue-suricata`). One additional grep match was a separate sudo log entry referring to the query text, not a fifth Suricata alert.

**Recovery and evidence scope:** The operator ran the mirror rollback script and independently verified TrueNAS `vnet13` returned to `noqueue` with **no `clsact`**. The temporary mirror was **removed**; no persistent traffic monitoring path is claimed. At this V-112–V-115 checkpoint, the specific ICMP flow had generated four Suricata alerts and four matching Wazuh manager alerts, but dashboard confirmation was **not yet recorded**. **Subsequent V-116** confirmed at least one matching SID `1000002` in Wazuh Threat Hunting, supported by two uploaded screenshot files (file presence verified; screenshot contents not independently inspected). **V-117–V-118** later confirmed scheduled backup rollback execution and clean post-timer `vnet13` state. These tests do **not** establish full Victim-network visibility, continuous mirroring, or broad protocol coverage.

[Validation V-112–V-115](validation.md#project-05--victim-sid-1000002-end-to-end-suricata-and-wazuh-alert-2026-10-20) · [Build notes](build-notes.md#project-05--victim-suricata-sid-1000002-and-wazuh-detection-2026-10-20).

## Project 05 — Victim Detection Evidence Uploaded (2026-10-20)

**Evidence present:** [`evidence/05-victim-suricata-wazuh-detection.png`](evidence/05-victim-suricata-wazuh-detection.png) was uploaded by the operator and the file's existence verified in GitHub. It is a saved Wazuh-manager terminal screenshot associated with the four alert summaries for Suricata SID `1000002`, Wazuh rule `86601`, agent `003` (V-115). The independently pasted CLI results validate the alert fields; the screenshot pixels have not been independently reviewed. This is **not** a Wazuh dashboard screenshot, and the temporary mirror remains rolled back.

[Evidence index](evidence/README.md#captured-screenshot--2026-10-20-victim-suricata-to-wazuh) · [Validation record](validation.md#project-05--uploaded-evidence-for-v-115-2026-10-20).

## Project 05 — Wazuh Threat Hunting Victim Dashboard Evidence (2026-10-20)

**Dashboard verification: PASS for at least one matching alert.** The operator observed Wazuh `rule.id:86601` in Threat Hunting for the October 20 Victim test and confirmed the expanded event's `data.alert.signature_id` equals **`1000002`**. For a readable GitHub portfolio, the operator uploaded separate [alert overview](evidence/10-wazuh-victim-alert-overview.png) and [alert details](evidence/11-wazuh-victim-alert-details.png) screenshot files; both exist in the repository. Image contents were not independently inspected. The **four-event** Wazuh count is separately verified by the manager CLI evidence under V-115, not by these screenshot uploads. Scoped temporary ICMP mirroring was removed, so continuous Victim network detection has not been demonstrated.

[Validation V-116](validation.md#project-05--wazuh-victim-alert-dashboard-and-screenshot-evidence-2026-10-20) · [Evidence index](evidence/README.md#wazuh-victim-alert-dashboard--2026-10-20).

## Project 05 — R3 Backup Rollback Confirmed (2026-10-20)

**R3 scheduled safety rollback: PASS.** After earlier successful manual mirror removal and independent `vnet13` `noqueue` check, TrueNAS `journalctl` showed `cyberblue-br30-mirror-rollback-r3.service` started at **11:32:57 PDT** on October 20, printed `CYBERBLUE BR30 MIRROR ROLLBACK COMPLETE`, and deactivated successfully. This confirms the timed cleanup script also ran. The separate post-timer qdisc check was pending at the V-117 checkpoint, then **completed as PASS under V-118**: `vnet13` showed only `noqueue`, without `clsact`. No permanent mirror is claimed.

[Validation V-117](validation.md#project-05--r3-scheduled-rollback-execution-2026-10-20).

## Project 05 — R3 Post-Timer Restoration Check (2026-10-20)

**Recovery verification complete (V-118):** After R3 scheduled rollback ran at 11:32:57 PDT, a fresh TrueNAS `sudo tc qdisc show dev vnet13` returned only `qdisc noqueue 8023: root refcnt 2`. No `clsact` is present. Thus the temporary ingress mirroring attachment was absent both after manual cleanup and after the redundant scheduled timer. This proves safe rollback of the scoped test, not continuous Victim traffic monitoring.

[Validation V-118](validation.md#project-05--post-r3-timer-qdisc-state-confirmed-2026-10-20).
