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
