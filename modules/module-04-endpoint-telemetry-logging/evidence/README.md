# Module 04 — Endpoint Telemetry Evidence

This directory contains screenshots and supporting evidence collected during the endpoint telemetry validation.

## Evidence Index

| Evidence | Description | Result |
|---|---|---|
| 01 | [Windows native process creation — Event ID 4688](05-windows-process-audit-restored.PNG) | PASS |
| 02 | [Windows service installation — Event ID 7045](06-windows-service-install-7045.PNG) | PASS |
| 03 | [Windows file access — Event ID 4663](07-windows-file-audit-4663.PNG) | PASS |
| 04 | [Linux sudo / administrative telemetry](08-linux-sudo-admin-activity.PNG) | PASS |
| 05 | [Linux native file visibility baseline — no matching journal entry](05-linux-native-file-visibility-gap.PNG) | PASS |
| 06 | [Linux auditd file telemetry — chmod validation](0-linux-auditd-file-activity.PNG) | PASS |
| 07 | [Sysmon process creation — Event ID 1](10-enhanced-windows-telemetry.PNG) | PASS |
| 08 | [Sysmon network connection — Event ID 3](10a-sysmon-network-connection.PNG) | PASS |
| 09 | [Sysmon file creation — Event ID 11](10b-sysmon-file-create-restored.PNG) | PASS |
| 10 | [Controlled Linux auditd interruption and recovery — systemd journal](10-linux-auditd-controlled-interruption.PNG) | PASS |
| 11 | [Linux auditd telemetry restoration — successful chmod event](11-enhanced-linux-audit.PNG) | PASS |
| 12 | [Controlled Windows Sysmon visibility gap — Event ID 11 absent](16-visibility-gap.PNG) | PASS |
| 13 | [Windows Sysmon telemetry restoration — Event ID 11](17-windows-sysmon-telemetry-restored.PNG) | PASS |
| 14 | Sysmon historical timeline — Events 1 and 3 supported by screenshots 07 and 08; original October 5 Event ID 11 no longer retained in the current Sysmon log | Partial evidence |
| 15 | Final Windows Sysmon service status — Running / Automatic (documented in validation records; screenshot `18-windows-sysmon-final-service-status.PNG` is **not present** in the repository) | SCREENSHOT MISSING — do not mark as uploaded |

**Repository reconciliation (2026-10-22):** Evidence entries 01–13 have linked files; entry 14 is Partial historical evidence; entry 15 references an absent screenshot. Service state may be documented elsewhere, but this file is not present and cannot count as screenshot evidence.

## Evidence Handling

Screenshots should show:

- The relevant command or event output
- Timestamp where useful
- Hostname where useful
- The controlled test activity
- No passwords, authentication secrets, or unnecessary sensitive information

Evidence should correspond to activities documented in `build-notes.md` and `validation.md`.
