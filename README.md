# CyberBlue — Cybersecurity Home Lab & Portfolio

<img width="1774" height="887" alt="CyberBlue cybersecurity lab banner" src="https://github.com/user-attachments/assets/66fcc580-105e-49e6-88cb-af3863f1498a" />

## Overview

**CyberBlue** is my hands-on cybersecurity engineering lab and technical portfolio, built using TrueNAS SCALE, KVM/QEMU virtualization, Linux, Windows, and open-source security tools.

The purpose of this project is to demonstrate practical skills in infrastructure engineering, network segmentation, endpoint monitoring, security validation, troubleshooting, and technical documentation.

My approach is to build each capability, understand how it works, validate its behavior, deliberately test failure scenarios where appropriate, and document the results with supporting evidence.

This repository records actual lab work, including successful tests, troubleshooting findings, configuration limitations, and recovery procedures.

---

## Lab Environment

| Component | Role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization and storage host |
| Intel Core i5-3330 / 32 GB RAM | Physical lab hardware |
| KVM/QEMU | Virtualization technology |
| Ubuntu Server 26.04.1 LTS | Linux endpoint and CyberBlue management server |
| Kali Linux 2026.2 | Controlled security testing |
| Linux Mint 22.3 | Linux lab endpoint |
| Windows 11 | Windows endpoint and security event testing |
| Sysmon | Enhanced Windows endpoint telemetry |
| auditd | Enhanced Linux security auditing |
| GitHub | Technical documentation and evidence portfolio |

## Network Architecture

CyberBlue uses a management network and two dedicated virtual lab segments.

```text
                 HOME / MANAGEMENT LAN
                     192.168.1.0/24
                            |
                     TrueNAS SCALE
                      KVM / QEMU
                            |
               +------------+------------+
               |                         |
              br30                      br40
         SECURITY LAB               ISOLATED LAB
         10.10.30.0/24              10.10.40.0/24
               |                         |
          +----+----+                    |
          |         |                    |
        Ubuntu     Kali              Linux Mint
       .30.10     .30.20              .40.10

         Routed IPv4 forwarding between
         br30 and br40 is blocked
         by persistent firewall rules.
```

The Linux VMs also retain separate management-network connectivity. Therefore, the lab bridge firewall controls cross-segment IPv4 forwarding but does not provide complete isolation across every possible network path.

Windows 11 currently uses the management network and has been used for endpoint telemetry validation.

See [Project 02 — Network Segmentation](projects/02-networking-segmentation/README.md) for implementation details and limitations.

---

## CyberBlue Project Index

| Project | Focus | Status |
|---|---|---|
| [01 — Virtualization Foundation](projects/01-virtualization-foundation/truenas/README.md) | TrueNAS VM provisioning, networking, lifecycle, and recovery | **Validated** |
| [02 — Virtual Networking & Segmentation](projects/02-networking-segmentation/README.md) | Persistent bridges, addressing, routing controls, and troubleshooting | **Validated** |
| [03 — Endpoint Telemetry & Logging](projects/03-endpoint-telemetry/README.md) | Windows Event Logs, Sysmon, Linux auditd, visibility gaps, and recovery | **Validated — historical evidence limitation documented** |
| 04 — Centralized Logging & SIEM Foundations | Resource assessment and SIEM architecture planning | **Planning** |
| [05 — Centralized Logging & SIEM Foundations](projects/05-centralized-logging/README.md) | Wazuh 4.14.8 deployment, Windows agent enrollment, and Sysmon alert validation | **In progress — Windows Sysmon integration validated** |

Additional projects will be added only as implementation and validation progress.

---

## Project 01 — Virtualization Foundation

Established the TrueNAS SCALE virtualization environment and validated Ubuntu Server VM deployment.

**Capabilities demonstrated:**
- VM resource provisioning
- Virtual storage and VirtIO networking
- Guest operating system installation
- Remote SSH administration
- VM shutdown and restart testing
- Snapshot recovery validation
- Infrastructure documentation

[View Project 01 →](projects/01-virtualization-foundation/truenas/README.md)

---

## Project 02 — Virtual Networking & Segmentation

Implemented dedicated TrueNAS-managed Linux bridges to create separate CyberBlue lab networks.

**Capabilities demonstrated:**
- Linux bridge networking
- Persistent VM network interfaces
- Static IPv4 addressing
- Routing and gateway validation
- Firewall-controlled IPv4 forwarding
- Troubleshooting macvtap and libvirt network behavior
- Controlled failure and recovery testing
- Distinguishing network segmentation from complete host isolation

[View Project 02 →](projects/02-networking-segmentation/README.md)

---

## Project 03 — Endpoint Telemetry & Logging

Investigated native and enhanced security telemetry across Windows and Linux systems.

**Capabilities demonstrated:**
- Windows Security and System event analysis
- Windows process, service, authentication, and file auditing
- Sysmon process, network, and file telemetry
- Linux journald and authentication log investigation
- Linux auditd monitoring
- Controlled logging failures
- Visibility-gap analysis
- Telemetry restoration and recovery validation

**Evidence note:** Fourteen evidence entries are marked PASS, and one historical Sysmon timeline evidence item is marked Partial because the original event is no longer retained in the current event log.

[View Project 03 →](projects/03-endpoint-telemetry/README.md)

---

## Project 04 — Centralized Logging & SIEM Foundations

**Status: Planning / Resource Assessment**

This project will evaluate and implement centralized security event collection and analysis using a SIEM platform selected to fit the CyberBlue lab's hardware and operational requirements.

Initial work includes measuring host memory availability, evaluating VM resource requirements, and planning how Linux and Windows endpoint telemetry will be collected centrally.

No SIEM deployment or event ingestion validation is claimed at this stage.

---

## Technical Skills Demonstrated

**Infrastructure & Virtualization**
- TrueNAS SCALE administration
- KVM/QEMU virtual machines
- Virtual storage and resource management
- Linux and Windows guest administration

**Networking & Troubleshooting**
- IPv4 networking and routing
- Linux bridges and virtual NICs
- Network segmentation
- Firewall policy validation
- Connectivity testing and fault isolation

**Security Operations**
- Windows Event Viewer
- Windows Security auditing
- Sysmon
- Linux journald and auditd
- Security event investigation
- Telemetry coverage analysis
- Controlled failure and recovery testing

**Engineering Practices**
- Evidence-based validation
- Root cause investigation
- Technical documentation
- Reproducible testing
- GitHub portfolio management

---

## Engineering Workflow

```text
UNDERSTAND
    |
    v
DESIGN
    |
    v
BUILD
    |
    v
VALIDATE
    |
    v
TEST / TROUBLESHOOT
    |
    v
RECOVER
    |
    v
DOCUMENT
    |
    v
IMPROVE
```

The goal is not simply to install security products. It is to understand the underlying capabilities, validate them through repeatable testing, and document what the results actually demonstrate.

---

## Documentation and Evidence

Each CyberBlue project is intended to contain:

- `README.md` — objectives, architecture, implementation overview, and results
- `build-notes.md` — configuration details, commands, troubleshooting, and lessons learned
- `validation.md` — tests performed, observed results, and evidence references
- Screenshots or other supporting artifacts where available

Validation results and evidence limitations are documented rather than assumed.

---

## Portfolio Purpose

CyberBlue demonstrates practical infrastructure and security engineering skills through hands-on projects that can be reviewed independently.

The repository is intended to serve as a technical learning record, troubleshooting journal, cybersecurity portfolio, and foundation for future security operations and detection engineering projects.

**Repository maintained by:** [petew-tech](https://github.com/petew-tech)
