# CyberBlue — Lab Architecture

## 1. Overview

CyberBlue is a hands-on cybersecurity lab built on TrueNAS SCALE 24.10.1 using KVM/QEMU virtualization.

The environment supports infrastructure administration, virtual networking, security testing, endpoint telemetry, controlled failure scenarios, troubleshooting, and recovery validation.

The architecture separates management connectivity from dedicated lab networks while retaining centralized management of the virtual machines through TrueNAS.

---

## 2. Physical Infrastructure

| Component | Configuration |
|---|---|
| Virtualization host | TrueNAS SCALE 24.10.1 (Electric Eel) |
| Processor | Intel Core i5-3330 @ 3.00 GHz |
| Installed memory | 32 GB |
| Virtualization | KVM/QEMU with libvirt |
| Primary storage | TrueNAS Pool1, RAIDZ1 |
| Physical network interface | `enp4s0` |
| Management network | `192.168.1.0/24` |
| TrueNAS management IP | `192.168.1.90` |

Virtual machines are created and managed through the TrueNAS web interface.

The host provides CPU, memory, storage, and virtual networking resources for the CyberBlue environment.

---

## 3. Virtual Machine Inventory

| Virtual Machine | Operating System | Role |
|---|---|---|
| Ubuntu Server | Ubuntu Server 26.04.1 LTS | CyberBlue management server and Linux telemetry endpoint |
| Kali Linux | Kali Linux 2026.2 | Controlled security testing |
| Linux Mint | Linux Mint 22.3 | Linux endpoint and isolated-network testing |
| Windows 11 | Windows 11 | Windows endpoint telemetry and security event validation |

All four virtual machines use TrueNAS-managed virtualization.

VM operating states and resource availability may vary during testing to accommodate the physical host's memory limitations.

---

## 4. Network Architecture

**Historical Project 02 baseline (superseded for guest placement by later stages):** The following diagram and table show the original two-lab-bridge configuration. The [2026-10-18 migration record](#module-06--ubuntu-and-windows-ip-migration-2026-10-18) below provides the latest verified guest addressing.

| Network | Subnet | Interface / Bridge | Purpose |
|---|---|---|---|
| Home / Management LAN | `192.168.1.0/24` | `enp4s0` | Administrative and management connectivity |
| Security Lab | `10.10.30.0/24` | `br30` | Ubuntu Server and Kali Linux |
| Isolated Lab | `10.10.40.0/24` | `br40` | Linux Mint |

### Logical Network Diagram

```text
                    HOME / MANAGEMENT LAN
                        192.168.1.0/24
                               |
                         TrueNAS SCALE
                         192.168.1.90
                               |
                   KVM/QEMU Virtualization
                               |
                  +------------+------------+
                  |                         |
                 br30                      br40
            SECURITY LAB               ISOLATED LAB
            10.10.30.0/24              10.10.40.0/24
                  |                         |
             +----+----+                    |
             |         |                    |
           Ubuntu     Kali               Linux Mint
         10.10.30.10 10.10.30.20         10.10.40.10

              Routed IPv4 traffic between
              br30 and br40 is blocked
              by persistent firewall rules.
```

### Virtual Machine Network Placement

| Virtual Machine | Management IP | Lab IP | Lab Bridge |
|---|---|---|---|
| Ubuntu Server | `192.168.1.245/24` | `10.10.30.10/24` | `br30` |
| Kali Linux | `192.168.1.91/24` | `10.10.30.20/24` | `br30` |
| Linux Mint | `192.168.1.92/24` | `10.10.40.10/24` | `br40` |
| Windows 11 | Management network | Not assigned | None |

These addresses represent the documented Project 02 network configuration.

---

## 5. Network Segmentation and Security Boundaries

The CyberBlue Security Lab and Isolated Lab use separate persistent TrueNAS-managed Linux bridges.

Persistent firewall rules on TrueNAS block routed IPv4 forwarding between `br30` and `br40`.

In the **Project 02 baseline**, Ubuntu and Kali shared the Security Lab, while Mint used the Isolated Lab. The later guest migration changes this placement; the historical firewall rules alone do not establish final three-zone isolation.

### Important Security Limitation

All three Linux virtual machines also retain connections to the shared home/management network.

Therefore, blocking IPv4 forwarding between `br30` and `br40` does not establish complete isolation between the virtual machines.

The current architecture demonstrates controlled lab-network segmentation rather than comprehensive management-plane isolation.

---

## 6. Virtualization Architecture

CyberBlue uses the following virtualization components:

- **TrueNAS SCALE:** Host operating system and VM management interface.
- **KVM:** Linux kernel virtualization capability.
- **QEMU:** Virtual machine execution and emulated/virtual hardware.
- **libvirt:** Virtual machine management interface used by the platform.
- **VirtIO:** Paravirtualized guest devices for storage and networking.

Virtual machines are configured through TrueNAS management rather than directly editing TrueNAS-owned persistent libvirt definitions.

### Historical Networking Development

Project 02 initially experimented with libvirt-managed `virbr30` and `virbr40` networks and temporary guest NIC attachments.

Those temporary attachments did not persist through VM restarts.

The lab subsequently migrated to persistent TrueNAS-managed `br30` and `br40` bridges with static lab IP addresses.

The historical implementation and troubleshooting findings are preserved in Project 02.

---

## 7. Endpoint Telemetry Architecture

CyberBlue Project 03 introduced security telemetry testing on Windows and Linux endpoints.

### Windows 11

- Windows Security and System event logs
- Native process, authentication, service, and file auditing
- Sysmon process, network, and file telemetry
- Controlled telemetry suppression and restoration

### Ubuntu Server

- systemd journal
- Linux authentication logs
- SSH and sudo activity
- Linux auditd
- Controlled audit visibility failure and recovery

Telemetry was investigated locally on the endpoints.

Centralized SIEM ingestion is planned for a future project and is not represented as an implemented capability.

---

## 8. Storage and Recovery

TrueNAS Pool1 provides the storage infrastructure for the CyberBlue virtual machines.

Project 01 validated Ubuntu VM shutdown, restart, and snapshot recovery.

These tests establish a foundation for controlled experimentation and recovery workflows.

Snapshot recovery does not replace a comprehensive backup strategy.

---

## 9. Resource Constraints

The CyberBlue lab operates on a physical host with approximately 32 GB of installed RAM.

Running multiple virtual machines simultaneously requires resource planning.

Before adding resource-intensive services such as a SIEM, available memory and VM allocations must be reviewed.

Virtual machines may be shut down when not required for a particular exercise.

---

## 10. Related Projects

- [Project 01 — Virtualization Foundation](../projects/01-virtualization-foundation/truenas/README.md)
- [Project 02 — Virtual Networking & Segmentation](../projects/02-networking-segmentation/README.md)
- [Project 03 — Endpoint Telemetry & Logging](../projects/03-endpoint-telemetry/README.md)

---

## 11. Architecture Status

**Current documented capabilities:**

- TrueNAS-managed KVM/QEMU virtualization
- Ubuntu, Kali, Linux Mint, and Windows virtual machines
- Persistent lab bridges and static addressing
- Controlled routed IPv4 segmentation
- Endpoint telemetry testing
- Controlled failure, troubleshooting, and recovery validation

**Future capabilities under consideration:**

- Centralized logging and SIEM
- Network security monitoring
- Security event correlation
- Detection engineering
- Vulnerability management

These future capabilities are not claimed as completed until their implementations and validation results are documented.

---

## 12. Documentation Principle

CyberBlue architecture documentation reflects observed and documented lab configurations.

Historical configurations, security limitations, and validation results are preserved so the portfolio demonstrates both technical implementation and engineering judgment.


---

## 13. Approved Target Architecture — SOC / Victim / Attack Segmentation (2026-10-16)

**Status: GUEST NETWORK PLACEMENT VALIDATED — SOC ON `br20`, WINDOWS VICTIM ON `br30`, KALI ATTACK ON `br40`; TESTED VICTIM ↔ ATTACK IPv4 ICMP FORWARDING DENIED; FINAL ISOLATION AND MONITORING COVERAGE REMAIN OPEN.** Project 02 documents the historical baseline. The newest observed guest changes, troubleshooting, and reboot checks are recorded in [Project 05 Module 06](../projects/05-suricata-network-detection/README.md#module-06--staged-network-migration-2026-10-18).

| Zone | Planned TrueNAS bridge | Planned subnet | Intended systems and role |
|---|---|---|---|
| Home / management | Existing management interface `enp4s0` | `192.168.1.0/24` | TrueNAS management `192.168.1.90`; retain existing management connectivity during migration |
| SOC | **Operational: two guests attached** `br20` | `10.10.20.0/24` | Ubuntu `cbubuntu01` `10.10.20.10/24` and Linux Mint `10.10.20.11/24` (VM-to-VM ping verified) |
| Victim | Existing `br30` | `10.10.30.0/24` | Windows 11 validated at `10.10.30.10/24`; Mint passive `ens5` attached; Ubuntu old NIC present without IPv4 |
| Attack | Existing `br40` | `10.10.40.0/24` | Kali Linux validated at `10.10.40.20/24` after reboot; Mint Suricata capture `ens4 10.10.40.10/24` previously validated |

### Validated Guest Placement — Logical Layout (isolation testing still open)

```text
                HOME / MANAGEMENT LAN
                    192.168.1.0/24
                           |
                TrueNAS SCALE 24.10.1
                   192.168.1.90
                           |
         +-----------------+-----------------+
         |                 |                 |
        br20              br30              br40
       SOC ZONE        VICTIM ZONE        ATTACK ZONE
   10.10.20.0/24     10.10.30.0/24     10.10.40.0/24
         |                 |                 |
   Ubuntu Server       Windows 11        Kali Linux
   Linux Mint sensor
```

**Diagram notes:** Ubuntu/Mint SOC, Windows Victim, and Kali Attack guest placement and selected connectivity have been validated in stages. This diagram does **not** prove bridge isolation, comprehensive IDS visibility, or host-reboot persistence. Linux Mint retains dedicated Suricata capture interfaces on `br30` and `br40`, plus its SOC and management connectivity. The separate Wazuh SIEM is `SIEM_01` (`192.168.1.213`); do not conflate that dedicated VM with the Ubuntu SOC/endpoint VM.



### First Implementation Milestone — SOC Bridge Created (2026-10-16)

- TrueNAS SCALE Network → Interfaces configuration created a new bridge named `br20` with **no physical bridge members** and IPv4 alias `10.10.20.1/24`.
- The first temporary application reverted when the confirmation window expired; the operator repeated the configuration and saved it through TrueNAS.
- TrueNAS shell check `ip -4 addr show br20` showed interface administratively **UP**, `NO-CARRIER` / operational `DOWN`, with `inet 10.10.20.1/24`. No guest ports were attached yet, so lack of carrier was expected.
- TrueNAS `midclt call interface.query` confirmed `{name: br20, type: BRIDGE, aliases: [{type: INET, address: 10.10.20.1, netmask: 24}]}`.
- **Validated:** Bridge existence, IP assignment, TrueNAS-managed configuration, and Ubuntu ↔ Linux Mint `br20` guest-to-guest ping. **Not yet validated:** host reboot persistence, inter-zone routing/firewall boundaries involving `br20`, and victim/attack workload migrations. Existing management, Suricata and Wazuh integrations must be retained throughout staged changes.

### First Ubuntu SOC Connection — 2026-10-16

- Renamed the TrueNAS VM display name from `cbunbubtu01` to **`cbubuntu01`**, VM ID `8`, while stopped. The guest hostname was corrected afterward as documented below.
- Preserved original VirtIO NICs: TrueNAS device `24` → `enp4s0` / Ubuntu `mgmt0` (MAC `00:a0:98:76:2c:c2`) and device `41` → `br30` / Ubuntu `lab0` (MAC `00:a0:98:11:81:0a`).
- Added TrueNAS VirtIO NIC device `52` → `br20` / Ubuntu `ens4` (MAC `00:a0:98:64:1e:3c`), verified through `vm.device.query`.
- On Ubuntu, existing Netplan `/etc/netplan/00-installer-config.yaml` retained management DHCP on `mgmt0` and static `10.10.30.10/24` on `lab0`. Created separate `/etc/netplan/10-cyberblue-soc.yaml`, MAC-matched `ens4` with static `10.10.20.10/24`, DHCPv4 and DHCPv6 off, no new default gateway; permissions `600`. `sudo netplan generate` passed and the configuration was activated with `sudo netplan try --timeout 180`.
- `ip -4 addr show ens4` confirmed link UP, `inet 10.10.20.10/24`.
- **Connectivity validation (all 4 transmitted / 4 received / 0% loss):** Ubuntu `ens4` → TrueNAS `br20` `10.10.20.1` (average RTT 0.318 ms); `mgmt0` → home gateway `192.168.1.1` (0.507 ms); `lab0` → TrueNAS `br30` `10.10.30.1` (0.300 ms).
- **Result:** One Ubuntu SOC participant connected and its existing management/lab connectivity preserved. No inter-guest SOC test, bridge reboot test, new inter-zone firewall validation or remaining VM move has been completed.
- **Hostname correction (2026-10-16):** TrueNAS VM display name is `cbubuntu01` (VM ID `8`). Ubuntu's system hostname originally remained `cbunbuntu01`. Ran `sudo hostnamectl set-hostname cbubuntu01`, verified `hostnamectl --static` returned `cbubuntu01`, changed `/etc/hosts` mapping from `127.0.1.1 cbunbuntu01` to `127.0.1.1 cbubuntu01` using `sudo sed -i.bak 's/cbunbuntu01/cbubuntu01/' /etc/hosts` (backup `/etc/hosts.bak`), then verified `/etc/hosts` contents. Existing session's prompt still showed old name, expected until a fresh shell/login; no restart required.


### Linux Mint SOC Connection — 2026-10-17

- Before changing Mint's NICs, removed the **temporary** Kali `vnet5` traffic-control mirroring attachment with `sudo tc qdisc del dev vnet5 clsact`; verified `sudo tc qdisc show dev vnet5` returned only the `noqueue` root qdisc. Historical mirroring test evidence remains in Project 05, but that mirror is **no longer active**.
- Stopped Linux Mint (TrueNAS VM ID `7`) and retained existing VirtIO devices: `20` management `enp4s0` / guest `ens3`, `43` `br40` / guest `ens4`, and `51` `br30` / guest `ens5`. Added VirtIO device `53`, MAC `00:a0:98:7e:12:f8`, attached to `br20`; guest recognized it as `ens6`.
- Configured Mint's NetworkManager profile `Wired connection 4` (only `ens6`) with static `10.10.20.11/24`, `ipv4.never-default yes`, no gateway, and IPv6 disabled. Activated it with `sudo nmcli connection up "Wired connection 4" ifname ens6`; `ip -4 addr show ens6` confirmed link UP and `10.10.20.11/24`.
- **Validated:** From Mint `ens6`, pinged TrueNAS `10.10.20.1` (4/4, 0% loss, average 0.153 ms) and Ubuntu `cbubuntu01` `10.10.20.10` (4/4, 0% loss, average 0.461 ms). The latter validates **VM-to-VM traffic** on the SOC bridge.
- **Service recovery check:** Mint `systemctl is-active suricata` and `systemctl is-active wazuh-agent` both returned `active`. Wazuh agent log at `2026/10/17 12:13:31` reported successful connection to SIEM `192.168.1.213:1514/tcp`.
- **Post-restart verification (2026-10-17):** Suricata 7.0.3 initialized two capture threads on each of Mint `ens4` and `ens5`, loaded 53,144 rules without errors, and logged fresh `ens5` traffic. Mint sent 4 ICMP probes using `ens4` to TrueNAS `br40` (4/4 replies); Suricata EVE JSON recorded four corresponding alerts with SID `1000001`. Wazuh manager `siem-01` recorded matching rule `86601` level-3 alerts from agent `003`, confirming end-to-end alert delivery. Mint management `ens3` ping to `192.168.1.1` also passed (4/4 replies, zero loss). Detailed evidence is in Project 05 validation.
- **At this 2026-10-17 checkpoint:** Wazuh dashboard view, TrueNAS `br20` host reboot persistence, full bridge mirroring and inter-zone firewall/isolation tests were open. The later dashboard validation is recorded below; remaining infrastructure checks are still open.

### Current-versus-target boundary

- **As verified through 2026-10-18:** `br20` carries Ubuntu `ens4` / `10.10.20.10` and Mint `ens6` / `10.10.20.11` (earlier guest-to-guest ping). Windows Victim NIC on `br30` uses `10.10.30.10/24`, retained after restart. Ubuntu's previous `br30` NIC remains attached without IPv4 (default guest name `ens3` after reboot). **Kali has now migrated from `br30` / `10.10.30.20` to `br40` / `10.10.40.20`** and retained both Attack and management networks after reboot; SSH was restored following an interface-name mapping fault. Mint's passive `br30` `ens5` and `br40` capture `10.10.40.10` remain in the documented sensor layout.
- **Project 05 proof of concept, now cleaned up:** Temporary, narrowly scoped TrueNAS `tc` ingress and egress filters on Kali's former `vnet5` mirrored one Kali ↔ TrueNAS ICMP exchange to Mint's `vnet7`/`ens5`. Suricata logged four requests and four replies. The temporary `clsact` filters were subsequently **removed** and removal verified; this does **not** demonstrate active or persistent full-bridge monitoring.
- **Current segmentation caveat:** Existing shared management-LAN NICs create other potential network paths; bridge separation alone does not establish complete end-to-end VM isolation. Existing firewall controls must be reassessed after any new bridge or migration.

### Migration and module-growth guardrails

1. Preserve a verified rollback plan, TrueNAS access, and the functioning Suricata → EVE JSON → Wazuh integration. Avoid removing current NICs or bridges prematurely.
2. Validate resources (32 GB RAM, four-core Intel i5), VM states, host bridge identities, and precise interface/IP mappings before modifying topology.
3. Add `br20` as a planned change, then validate an initial SOC participant and management reachability before moving any existing detection services.
4. Stage Windows 11 onto `br30` and Kali onto `br40` only after checking their guest interfaces, current routes, security boundaries and test connectivity.
5. Plan explicit, narrowly scoped and reversible inter-zone test paths if an attack-zone exercise must reach the victim zone; do **not** permanently merge bridges, enable unrestricted routing or assume isolation based on zone labels.
6. Reassess TrueNAS packet mirroring after VM port mappings change. The current `vnet5` and `vnet7` references are runtime-specific and must not be hardcoded as future persistent topology.
7. For each new module: record its zone, systems, security controls, validation, evidence and rollback steps. Update this architecture only when implemented facts are independently verified; keep the target diagram clearly designated as proposed until complete.

**Related evidence:** [Project 05 — Suricata build notes](../projects/05-suricata-network-detection/build-notes.md) and [validation record](../projects/05-suricata-network-detection/validation.md).

### Post-Migration Threat Hunting Confirmation — 2026-10-17

- Wazuh dashboard Threat Hunting screenshot verified **4 hits** matching DQL query `rule.id: 86601`, manager `siem-01`, agent `003` / `cyberblue-suricata`, at ~12:24 CDT on October 17. All four rows show level 3 and `Suricata: Alert - CYBERBLUE TEST - ICMP Echo Request`.
- **Result: PASS** for tested Suricata -> Wazuh agent -> manager -> dashboard alert path after adding Linux Mint's SOC NIC. The earlier `agent.id: 002` dashboard filter had excluded the sensor; opening agent 003 and filtering by rule displayed the correct events.
- Screenshot [09-wazuh-post-soc-migration-alerts.png](../projects/05-suricata-network-detection/evidence/09-wazuh-post-soc-migration-alerts.png) has been uploaded and verified in GitHub. Unrestricted cross-zone routing, final-state isolation, and comprehensive passive capture are not established.


### Windows 11 Victim Bridge Onboarding — 2026-10-18

- TrueNAS WIN11 VM ID `11` was cleanly shut down before modifying NICs. Its pre-existing VirtIO management NIC (device `36`, MAC `00:a0:98:0a:82:2b`, attached to `enp4s0`) was preserved.
- Added second TrueNAS-managed VirtIO NIC (device `54`, MAC `00:a0:98:31:78:bd`, attached to `br30`). `vm.device.query` confirmed both attachments before the VM restarted.
- Windows identified management as `Ethernet 3` (interface index `19`, address `192.168.1.151`, default gateway/DNS `192.168.1.1`) and the new Victim NIC as `Ethernet` (interface index `3`).
- The new NIC initially received APIPA `169.254.133.82` without DHCP. TrueNAS `ping -I br30 -c 4 -W 1 10.10.30.11` returned no replies; this was a preliminary availability check, **not conclusive proof** no other device used the address. `arping` was not installed.
- Configured Windows Victim NIC via elevated PowerShell `New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.30.11 -PrefixLength 24` with no additional default gateway. Initial `Tentative`/ `Invalid` state in command output resolved; `Get-NetIPAddress` confirmed `10.10.30.11/24`, `Preferred`.
- **Connectivity (all four replies, 0% loss):** Windows `10.10.30.11` → TrueNAS `br30` `10.10.30.1` (average RTT reported 0 ms); Windows `10.10.30.11` → Ubuntu `lab0` `10.10.30.10` (3 ms); Windows management `192.168.1.151` → home gateway `192.168.1.1` (0 ms).
- **Historical onboarding result:** Windows initially used temporary Victim IP `10.10.30.11` while Ubuntu still held `10.10.30.10`. **Superseded by Module 06 below**; the initial setup and gateway-ping results remain valid historical evidence. Other guests and network paths remain; full isolation is unverified.


### Module 06 — Ubuntu and Windows IP Migration (2026-10-18)

- **Ubuntu SOC/management preserved:** `mgmt0 192.168.1.243/24` and `ens4 10.10.20.10/24` persisted through an Ubuntu guest reboot; default gateway remained `192.168.1.1`. After applying corrected Netplan, the former `lab0` interface lost `10.10.30.10/24` and its connected route. The VirtIO NIC still exists: after the guest reboot it appeared as `ens3` with IPv6 link-local only.
- **Windows Victim address migration:** Windows `Ethernet` moved from `10.10.30.11/24` to `10.10.30.10/24`. The original address was removed. Management `Ethernet 3 192.168.1.151/24` and default route via `192.168.1.1` remained. The Windows guest reboot preserved both IPs and both routes.
- **Connectivity after Windows reboot:** `ping -S 10.10.30.10 -n 4 10.10.30.1` and `ping -S 192.168.1.151 -n 4 192.168.1.1` each received 4/4 responses (0% loss).
- **Validated guest-level changes at this checkpoint:** Ubuntu and Windows address/routing persistence. The later Kali migration and guest reboot are recorded directly below. **Still not established:** TrueNAS host reboot persistence for `br20`, current cross-zone firewall enforcement, complete isolation, or broad Suricata mirroring. Guest IP removal does not imply guest NIC detachment.

Four screenshots were uploaded to [Project 05 evidence](../projects/05-suricata-network-detection/evidence/README.md#module-06--network-migration-evidence-2026-10-18), with detailed [build notes](../projects/05-suricata-network-detection/build-notes.md#module-06--ubuntu-and-windows-ip-migration-2026-10-18) and [validation](../projects/05-suricata-network-detection/validation.md#module-06--staged-network-migration-validation-2026-10-18).

### Module 06 — Kali Attack-Zone Migration and Recovery (2026-10-18)

- **TrueNAS reattachment:** Kali VM `6` retained management NIC device `16` on `enp4s0`, MAC `00:a0:98:5b:5f:c8`. Its lab NIC device `42` was moved from `br30` to `br40` while Kali was shut down. After the change, the guest's interface names mapped differently than the original IP profiles.
- **Observed and recovered fault:** Kali initially booted with the management address on the Attack-connected guest NIC and the Attack address on the management-connected NIC; SSH was unavailable and the management ping failed. The TrueNAS SPICE console enabled inspection and recovery. NetworkManager profile bindings were corrected to match the actual NIC connections, restoring management ping and PuTTY SSH.
- **Persistent mapping:** Kali Attack device `42` was assigned explicit MAC `00:a0:98:76:2d:fd`; device `16` retained management MAC `00:a0:98:5b:5f:c8`. Both Kali NetworkManager profiles were configured to match these MACs, rather than `eth0`/`eth1` name restrictions.
- **Post-reboot result:** Kali `eth0 10.10.40.20/24` on `br40`, `eth1 192.168.1.91/24` on management, default via `192.168.1.1 dev eth1`, `10.10.40.0/24` connected on `eth0`, no `10.10.30.0/24` connected route. SSH reconnected. Post-reboot `10.10.40.1` and `192.168.1.1` gateway tests each succeeded with 4/4 replies and 0% loss.
- **Scope:** Kali guest NIC persistence, IPs, routing, gateway reachability and SSH **PASS**. This is not an inter-zone firewall test, a TrueNAS **host** reboot test, or a demonstration of bridge-wide Suricata visibility.

[Pre-reboot Kali evidence](../projects/05-suricata-network-detection/evidence/module06-kali-attack-network-pre-reboot.png) · [Post-reboot Kali evidence](../projects/05-suricata-network-detection/evidence/module06-kali-attack-network-post-reboot.png) · [Detailed validation](../projects/05-suricata-network-detection/validation.md#kali-attack-zone-migration-and-recovery--2026-10-18)

### Module 06 — Controlled Victim ↔ Attack IPv4 Firewall Tests (2026-10-19)

The deployed TrueNAS `iptables` FORWARD rules include `-i br40 -o br30 -j DROP` and `-i br30 -o br40 -j DROP`. `iptables-legacy` also carries a separate Tailscale `ts-forward` chain; inspection found no explicit CyberBlue bridge rules in that legacy chain. TrueNAS kernel IPv4 forwarding is enabled (`ip_forward=1`).

**Attack → Victim:** Kali's normal route to Windows used the management LAN; an explicitly scoped temporary host route directed only `10.10.30.10/32` through `eth0 10.10.40.20` and `br40 10.10.40.1`. Four ICMP probes received no reply. TrueNAS `br40 → br30` DROP counter increased 4 → 8 (+4; 336 → 672 B). Removed route and verified original management route was restored.

**Victim → Attack:** Windows likewise preferred management routing. A temporary, nonpersistent route for `10.10.40.20/32` through Victim `Ethernet 10.10.30.10` and `br30 10.10.30.1` was verified with `Find-NetRoute`. Four ICMP probes timed out, while TrueNAS `br30 → br40` DROP counter increased 4 → 8 (+4; 336 → 576 B). Removed route and verified Windows resumed normal management routing via `Ethernet 3 192.168.1.151` and `192.168.1.1`.

**Validation result: PASS — specific bidirectional routed IPv4 ICMP blocking between the Attack and Victim guests.** The SOC bridge `br20`, other IP protocols, IPv6, shared management access, TrueNAS host reboot persistence, and broad Suricata packet visibility have **not** been validated by these tests. No new evidence screenshot is represented as uploaded.

[Project 05 validation V-46–V-51](../projects/05-suricata-network-detection/validation.md#module-06--victim--attack-ipv4-isolation-tests-2026-10-19) · [Repeatable test and rollback notes](../projects/05-suricata-network-detection/build-notes.md#module-06--victim--attack-ipv4-firewall-test-procedure-2026-10-19)

## 14. Approved Default-Deny Inter-Zone Policy — Design Decision (2026-10-19)

**Decision: APPROVED DESIGN; IMPLEMENTATION AND VALIDATION INCOMPLETE.** Adopt the logical intent of the partner's Cyber Forge zone drawing for CyberBlue's **TrueNAS SCALE** environment. The reference drawing depicts Proxmox and absence of routing; this lab instead uses TrueNAS-managed `br20`, `br30`, and `br40` and currently has kernel IPv4 forwarding enabled. Match the **security outcome**, not its hypervisor-specific implementation.

**Required default policy:** Deny unauthorized inter-zone communication in both directions among SOC (`br20` / `10.10.20.0/24`), Victim (`br30` / `10.10.30.0/24`), and Attack (`br40` / `10.10.40.0/24`). Permit only narrowly scoped, documented exceptions needed for monitoring, administration, or controlled lab exercises. Time-bound test allowances must be reversible and revalidated after cleanup.

| Directed inter-zone path | Approved baseline |
|---|---|
| SOC `br20` ↔ Victim `br30` | DENY unless explicitly authorized |
| SOC `br20` ↔ Attack `br40` | DENY unless explicitly authorized |
| Victim `br30` ↔ Attack `br40` | DENY unless explicitly authorized |

**Before firewall changes:** Inspect the current TrueNAS firewall script `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh`, its installed `POSTINIT` startup behavior, Docker and Tailscale forwarding chains, and paths required for existing TrueNAS management, guest SSH/PuTTY/SPICE recovery, Suricata capture, Wazuh agent-to-SIEM delivery, and authorized administration. An exact exception list has **not yet been approved**. Preserve access and establish a rollback procedure before changing effective policy.

**Security scope:** The current guests have management-LAN NICs, and Mint has monitoring NICs on both Attack and Victim bridges. `FORWARD` rules cannot by themselves prohibit same-bridge Layer-2 traffic or TrueNAS-host `INPUT`/`OUTPUT` access; alternate management and IPv6 paths must be assessed independently. Passive packet capture also requires a separately validated mirror/capture delivery mechanism. No broad blanket routing exceptions should be assumed.

**As-tested state:** On 2026-10-19, scoped routed IPv4 ICMP between Kali and Windows was dropped in both directions by the existing `br40` ↔ `br30` rules, with matching counter increments. Neither `br20` isolation nor comprehensive default-deny enforcement has passed. This section records policy intent, **not a new configuration change, test result, or completion claim**.

[Networking implementation reference](networking.md) · [Project 05 scoped firewall validation](../projects/05-suricata-network-detection/validation.md#module-06--victim--attack-ipv4-isolation-tests-2026-10-19)
