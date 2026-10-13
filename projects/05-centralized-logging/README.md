# Project 05 — Centralized Logging & SIEM Foundations

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

**Scope note:** This is a count of matching Sysmon *alerts* in the local alert log at the time of the test, not a count of all raw Sysmon events. The Threat Hunting searches attempted during validation returned no results for the chosen filters, while the server-side alert log independently confirmed that Sysmon alerts were generated and stored. Dashboard filter investigation remains a follow-up task.

## Evidence

A screenshot of the server-side Sysmon alert count was captured during validation but has not yet been uploaded to this repository.

## Status

**In progress — Windows Sysmon-to-Wazuh alerting validated.** Further endpoint onboarding, dashboard validation, and documentation may follow.
