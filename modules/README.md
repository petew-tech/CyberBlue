# CyberBlue — Training Modules

This directory contains principle-first cybersecurity and infrastructure training materials supporting the CyberBlue lab.

## Training Method

Each module follows a structured learning process:

1. Understand the underlying principle.
2. Examine the architecture and dependencies.
3. Implement the capability.
4. Validate expected behavior.
5. Introduce controlled failures where appropriate.
6. Troubleshoot and restore functionality.
7. Document findings and lessons learned.

## CyberBlue Module Index — Documented Lab Progress (2026-10-22)

| Module | Documentation | Status |
|---|---|---|
| 02 — Range Foundation | [Module 02](module-02-range-foundation/README.md) | Core virtualization exercises validated; current TrueNAS storage managed separately |
| 03 — Virtual Networking & Segmentation | [Module 03](module-03-virtual-networking-segmentation/README.md) | Core network tests validated; full persistent three-zone enforcement deferred |
| 04 — Endpoint Telemetry & Logging | [Module 04](module-04-endpoint-telemetry-logging/README.md) | Core exercises validated; one historical partial item and one absent evidence screenshot disclosed |
| 05 — Centralized Logging & SIEM | [Module 05](module-05-centralized-logging-siem/README.md) | Foundational SIEM and Windows telemetry validated |
| 06 — Network Security Monitoring | [Module 06](module-06-network-security-monitoring/README.md) | Scoped Suricata/Wazuh pipeline validated; persistent bridge-wide mirroring open |
| 07 — Network Forensics & Session Investigation | [Module 07](module-07-network-forensics/README.md) | SSH, reconnaissance, HTTP recovery and packet-loss analysis validated; continuous Zeek monitoring open |

These are evidence-based completion descriptions, not claims of comprehensive isolation or production hardening. Historical configurations remain in their original dated sections, while each module's README describes the latest verified layout.

## Training Method and Documentation

Module directories now hold both training explanations and hands-on build/validation records (the former `projects/` locations were migrated). The `README.md`, `build-notes.md`, `validation.md`, and evidence folders are the source documentation; reference the specific dated checkpoint when comparing changes.

The training materials will be adapted to the existing TrueNAS SCALE environment rather than assuming a Proxmox or standalone Linux workstation configuration.
