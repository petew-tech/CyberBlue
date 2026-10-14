# Project 04 — Centralized Logging & SIEM Foundations

## Overview

This project documents the deployment and validation of a Wazuh SIEM server and centralized Windows endpoint telemetry in the CyberBlue lab.

## Lab Components

| Component | Configuration / role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization host |
| `SIEM_01` / `siem-01` | Ubuntu Server 24.04.5 LTS, Wazuh all-in-one SIEM |
| Wazuh | Version 4.14.8; manager, indexer, dashboard, and Filebeat |
| SIEM management IP | `192.168.1.213` |
| Windows 11 endpoint | `WIN11`, Wazuh agent ID `002` |
| Sysmon64 | Windows endpoint telemetry source |

## Implementation Summary

- Deployed Wazuh 4.14.8 on a dedicated Ubuntu Server VM.
- Verified Wazuh manager, indexer, dashboard, and Filebeat operation.
- Enrolled the Windows 11 Wazuh agent and confirmed an active connection.
- Added `Microsoft-Windows-Sysmon/Operational` as an `eventchannel` source in the Windows agent's `ossec.conf`.
- Backed up the original configuration, validated the updated XML, and restarted the Wazuh agent.
- Confirmed Sysmon64 and the Wazuh Windows service were running.
- Validated centralized Sysmon alert searches in Wazuh Threat Hunting by rule ID, Event ID, provider, and channel; see [validation.md](validation.md) for the recorded point-in-time results.
- Performed Windows 11 CIS Security Configuration Assessment and independently verified selected hardening remediations through V-45; the latest recorded assessment was 148 Passed / 325 Failed / 9 Not applicable (31%), with remaining findings not fully remediated.

## Validation

| Test | Observed result | Status |
|---|---|---|
| Windows agent registration | `WIN11` registered as agent `002` and active | PASS |
| Sysmon local event generation | Sysmon Operational log returned Event ID 5 (process termination) | PASS |
| Wazuh agent log collection | Agent log reported `Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'` | PASS |
| SIEM connectivity | Agent log reported connection to `192.168.1.213:1514/tcp` | PASS |
| Centralized Sysmon detection | `/var/ossec/logs/alerts/alerts.json` contained Sysmon alerts for `WIN11` | PASS |
| Process creation detection | Sysmon Event ID 1 triggered Wazuh rule `92021` (level 3) | PASS |
| File creation detection | Sysmon Event ID 11 triggered Wazuh rule `92205` (level 9) | PASS |
| Alert count | Command below returned `32` at the time of testing | PASS |

### Alert Count Evidence

```bash
sudo grep -c '"groups":\["sysmon"' /var/ossec/logs/alerts/alerts.json
```

Observed result: `32`.

**Scope note:** This is a count of matching Sysmon *alerts* in the local alert log at the time of the test, not a count of all raw Sysmon events. Subsequent Threat Hunting dashboard validation confirmed searchable centralized Sysmon alerts for Windows agent `002` (`WIN11`): `data.win.system.eventID:11` returned 92 matching alerts, `data.win.system.providerName:"Microsoft-Windows-Sysmon"` returned 113, and `data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"` returned 113 (point-in-time results within the Last 24 hours window). These counts represent matching alerts for the selected filters, not all raw Sysmon events or evidence of malicious activity. The earlier no-results dashboard observation was superseded by this successful validation.

## Evidence

Evidence is available in this repository, including the [server-side Sysmon alert-count screenshot](Complete%20pipeline.PNG) (32 matching alerts at the time of validation). Additional dashboard search evidence, indexed Sysmon event details, and Windows 11 CIS remediation screenshots are cataloged with their validation results in [validation.md](validation.md), including V-40 through V-45. The latest documented CIS assessment showed 148 Passed, 325 Failed, and 9 Not applicable (31% score); this is a point-in-time hardening assessment, not a claim of full compliance.

## Status

**Complete — foundational SIEM deployment and Windows 11 telemetry validation.** Wazuh all-in-one deployment, Windows Sysmon-to-Wazuh alerting, and Sysmon-specific Threat Hunting dashboard searches were validated. Windows 11 CIS Security Configuration Assessment and selected hardening remediations are documented through V-45 in [validation.md](validation.md); the latest recorded assessment was 148 Passed / 325 Failed / 9 Not applicable (31%), **not full CIS compliance**.

**Future scope:** Additional endpoint onboarding, complete raw event ingestion coverage, and review or remediation of outstanding CIS findings. These items were not prerequisites for completing this foundational Project 04 scope and are **not claimed as validated**.
