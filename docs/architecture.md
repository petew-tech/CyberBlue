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

The CyberBlue environment uses three primary IPv4 networks.

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

Ubuntu Server and Kali Linux share the Security Lab, while Linux Mint uses the separate Isolated Lab.

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

**Status: PARTIALLY IMPLEMENTED — `br20` OPERATIONAL FOR UBUNTU ↔ MINT; VICTIM/ATTACK VM MIGRATION NOT YET DONE.** The existing, validated Project 02 networking above remains the historical/current documented implementation until individual migration stages are tested. This plan follows the general roles of a peer's Proxmox lab but retains TrueNAS SCALE and CyberBlue's own addressing, workloads and resource constraints.

| Zone | Planned TrueNAS bridge | Planned subnet | Intended systems and role |
|---|---|---|---|
| Home / management | Existing management interface `enp4s0` | `192.168.1.0/24` | TrueNAS management `192.168.1.90`; retain existing management connectivity during migration |
| SOC | **Operational: two guests attached** `br20` | `10.10.20.0/24` | Ubuntu `cbubuntu01` `10.10.20.10/24` and Linux Mint `10.10.20.11/24` (VM-to-VM ping verified) |
| Victim | Existing `br30` | `10.10.30.0/24` | Windows 11 as intended victim endpoint, subject to VM NIC and addressing validation |
| Attack | Existing `br40` | `10.10.40.0/24` | Kali Linux as intended controlled attack host, subject to NIC and addressing validation |

### Proposed Logical Layout

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

**Diagram notes:** This is logical placement, not evidence of connectivity, completed VM moves or full isolation. Linux Mint may retain dedicated receive-only capture NICs on the monitored victim/attack bridges while using a separate SOC/management connection. The Wazuh SIEM is already deployed separately as `SIEM_01` (`192.168.1.213`), and the existing working telemetry path must remain usable during migration. Do not conflate the existing Ubuntu management/endpoint VM with the dedicated SIEM VM.



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
- **Still open:** Post-restart Suricata capture and alert generation, live Wazuh dashboard event verification, management/isolated-network connectivity checks, TrueNAS `br20` reboot persistence, and full firewall/mirroring coverage on the final topology.

### Current-versus-target boundary

- **Existing setup as last verified:** `br20` now connects Ubuntu `ens4` / `10.10.20.10` and Mint `ens6` / `10.10.20.11` with guest-to-guest ping verified. `br30` still hosts Kali `10.10.30.20`, Ubuntu `lab0` `10.10.30.10`, and Mint passive `ens5`. Mint continues to use `10.10.40.10` on `br40`. Windows 11 and Kali have not yet moved to their target victim/attack placements.
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
