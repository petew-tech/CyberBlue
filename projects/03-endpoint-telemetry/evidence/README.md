# Module 04 — Evidence

This directory contains screenshots and supporting evidence collected during the endpoint telemetry validation.

## Evidence Index

| Evidence | Description | Result |
|---|---|---|
| 01 | Windows native process creation — Event ID 4688 | PASS |
| 02 | Windows service installation — Event ID 4697 | PASS |
| 03 | Windows file access — Event ID 4663 | PASS |
| 04 | Linux sudo / administrative telemetry | PASS |
| 05 | Linux native file visibility baseline | Baseline |
| 06 | Linux auditd file telemetry | PASS |
| 07 | Sysmon process creation — Event ID 1 | PASS |
| 08 | Sysmon network connection — Event ID 3 | PASS |
| 09 | Sysmon file creation — Event ID 11 | PASS |
| 10 | Controlled Linux telemetry failure | PASS |
| 11 | Linux telemetry restoration | PASS |
| 12 | Controlled Windows telemetry failure | PASS |
| 13 | Windows telemetry restoration | PASS |
| 14 | Sysmon controlled timeline | PASS |
| 15 | Final Windows Sysmon configuration | PASS |

## Evidence Handling

Screenshots should show:

- The relevant command or event output
- Timestamp where useful
- Hostname where useful
- The controlled test activity
- No passwords, authentication secrets, or unnecessary sensitive information

Evidence should correspond to activities documented in `build-notes.md` and `validation.md`.
