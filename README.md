# CyberBlue — Cybersecurity Home Lab & Portfolio

![CyberBlue SOC-style cybersecurity home lab and SIEM portfolio banner](CyberBlue-SOC-Banner.png)

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
| Ubuntu Server 24.04.5 LTS (`SIEM_01`) | Dedicated Wazuh 4.14.8 SIEM server — manager, indexer, dashboard, and Filebeat |
| Kali Linux 2026.2 | Controlled security testing |
| Linux Mint 22.3 | Linux lab endpoint |
| Windows 11 | Windows endpoint and security event testing |
| Sysmon | Enhanced Windows endpoint telemetry |
| auditd | Enhanced Linux security auditing |
| GitHub | Technical documentation and evidence portfolio |

## Network Architecture

**Current verified guest-network state (2026-10-18, Project 05 Module 06):** CyberBlue uses a home/management LAN and three TrueNAS-managed virtual lab bridges. The new zone layout is **partially implemented**, not yet a fully validated isolation boundary.

```text
               HOME / MANAGEMENT LAN
                   192.168.1.0/24
                          |
                  TrueNAS SCALE
                    KVM / QEMU
                          |
         +----------------+----------------+
         |                |                |
       br20             br30             br40
      SOC ZONE        VICTIM ZONE    TARGET ATTACK ZONE
   10.10.20.0/24     10.10.30.0/24     10.10.40.0/24
         |                |                |
   Ubuntu .20.10    Windows .30.10    Mint .40.10
   Mint   .20.11    Kali    .30.20*  (Suricata capture)

   * Kali's move from br30 to br40 is still pending.
```

Ubuntu's former `br30` VirtIO NIC remains present without an IPv4 address (it appeared as `ens3` after guest reboot); Mint also has a passive capture NIC on `br30`. The Windows Victim NIC uses `10.10.30.10/24`, after Ubuntu released it. Separate management NICs remain in use; the dedicated Wazuh SIEM is still on the management LAN.

Historically verified TrueNAS firewall rules block routed IPv4 forwarding between `br30` and `br40`, but **their effectiveness in the changing three-zone topology, IPv6 isolation, and alternative management-LAN paths have not been fully revalidated**. Guest reboot tests do not establish TrueNAS host bridge reboot persistence.

### Approved Target Zones — Migration In Progress

The planned end state is **`br20` for SOC (Ubuntu and Mint), `br30` for Windows Victim, and `br40` for Kali Attack**, while preserving required Mint sensor connections and management access. SOC connectivity and Windows addressing have been tested; Kali migration, final zone boundaries, and comprehensive passive monitoring remain open.

See the [approved target design and staged-migration safeguards](docs/architecture.md#13-approved-target-architecture--soc--victim--attack-segmentation-2026-10-16) and [Project 05 Module 06 results](projects/05-suricata-network-detection/README.md#module-06--staged-network-migration-2026-10-18).

---

## CyberBlue Project Index

| Project | Focus | Status |
|---|---|---|
| [01 — Virtualization Foundation](projects/01-virtualization-foundation/truenas/README.md) | TrueNAS VM provisioning, networking, lifecycle, and recovery | **Validated** |
| [02 — Virtual Networking & Segmentation](projects/02-networking-segmentation/README.md) | Persistent bridges, addressing, routing controls, and troubleshooting | **Validated** |
| [03 — Endpoint Telemetry & Logging](projects/03-endpoint-telemetry/README.md) | Windows Event Logs, Sysmon, Linux auditd, visibility gaps, and recovery | **Validated — historical evidence limitation documented** |
| [04 — Centralized Logging & SIEM Foundations](projects/04-centralized-logging/README.md) | Wazuh 4.14.8, Windows Sysmon alerting, Threat Hunting searches, and selected Windows 11 CIS hardening | **Complete — foundational SIEM deployment and Windows 11 telemetry validated** |
| [05 — Suricata Network Detection & Wazuh Integration](projects/05-suricata-network-detection/README.md) | Suricata/Wazuh validation and Module 06 staged SOC/Victim network migration | **Sensor validated; Module 06 guest migration validated; full zone isolation OPEN** |

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

**Status: Complete — Foundational SIEM Deployment**

Deployed Wazuh 4.14.8 on a dedicated Ubuntu Server VM and enrolled the Windows 11 endpoint. Validated centralized Sysmon alert generation, storage, and Threat Hunting dashboard searches. Performed Windows 11 CIS Security Configuration Assessment and documented selected hardening remediations through V-45.

Additional endpoint onboarding, complete raw-event ingestion coverage, and outstanding CIS findings remain future work.

[View Project 04 →](projects/04-centralized-logging/README.md)

---

## Project 05 — Suricata Network Detection & Wazuh Integration

**Status: Sensor Detection Validated; Module 06 Guest Network Migration Validated (zone isolation still open)**

Configured Suricata on Linux Mint to detect a controlled ICMP test, forward EVE JSON telemetry through the Wazuh agent, and generate a centralized Wazuh alert. A controlled Suricata process failure confirmed systemd automatic recovery; the sensor resumed detection and Wazuh continued receiving alerts. Point-in-time capture statistics reported zero kernel drops and zero capture errors.

Visibility into traffic exchanged between other VMs on TrueNAS bridge `br40` has not been demonstrated and remains a separate follow-up.

**Module 06 update (2026-10-18):** Ubuntu released `10.10.30.10` while retaining `192.168.1.243` management and `10.10.20.10` SOC addresses. Windows 11 moved from temporary `10.10.30.11` to `10.10.30.10` on `br30`, retaining management `192.168.1.151`. Guest reboot persistence and Windows gateway connectivity were verified with four evidence screenshots. Kali's zone migration and final isolation testing remain open. See [Module 06 evidence and validation](projects/05-suricata-network-detection/validation.md#module-06--staged-network-migration-validation-2026-10-18).

[View Project 05 →](projects/05-suricata-network-detection/README.md)

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
