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

## Training Module Index

| Module | Topic | Status |
|---|---|---|
| Prerequisite 01 | Linux and infrastructure fundamentals | Planned |
| Prerequisite 02 | KVM/QEMU virtualization foundations | Draft available |
| Module 02 | CyberBlue range foundation | Planned |
| Module 03 | Virtual networking and segmentation | Planned |
| Module 04 | Endpoint telemetry and logging | Planned |
| Module 05 | Centralized logging and SIEM foundations | Planned |

These statuses describe training documentation, not the completion status of the hands-on projects.

## Training vs. Projects

- **`modules/`** — Learning objectives, technical principles, exercises, and qualification questions.
- **`projects/`** — Actual configurations, implementation notes, validation results, troubleshooting, and evidence.

The training materials will be adapted to the existing TrueNAS SCALE environment rather than assuming a Proxmox or standalone Linux workstation configuration.
