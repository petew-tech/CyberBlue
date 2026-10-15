# Project 04 — Centralized Logging & SIEM Foundations
## Module 05 — Build Notes

**Build status:** Complete — foundational SIEM deployment and Windows 11 telemetry validation. Windows Sysmon-to-Wazuh alerting and Threat Hunting dashboard searches were validated, with Windows 11 CIS Security Configuration Assessment and selected hardening remediations documented through V-45 in [validation.md](validation.md). Complete raw Sysmon event ingestion coverage, additional endpoint onboarding, and outstanding CIS findings remain future scope; full CIS compliance is not claimed.

These notes distinguish verified lab actions from follow-up work. Commands are recorded for reference, not as an unattended installation script.

## 1. SIEM VM and Operating System

| Setting | Value |
|---|---|
| Hypervisor | TrueNAS SCALE 24.10.1 |
| VM | `SIEM_01` (TrueNAS VM ID 12) |
| Hostname | `siem-01` |
| OS | Ubuntu Server 24.04.5 LTS |
| Management IP | `192.168.1.213` (DHCP reservation) |
| vCPU | 4 |
| RAM | 8 GiB |
| Disk | 100 GB VirtIO |
| Swap | 4 GB |
| Wazuh | 4.14.8, all-in-one |

### OS Rebuild and Data Preservation

The initial Ubuntu Server 26.04.1 installation was replaced with Ubuntu Server 24.04.5 LTS for the SIEM VM. The earlier disk was **preserved**, not deleted.

- Detached original ZVOL: `Pool1/VM/SIEM_01-4qrn8p`
- Snapshot: `Pool1/VM/SIEM_01-4qrn8p@pre-ubuntu-24.04.5-rebuild`
- Backup clone: `Pool1/VM/SIEM_01-Ubuntu26-Backup`
- Current OS disk: `Pool1/VM/SIEM_01-Ubuntu24`

**Preservation warning:** Do not delete the detached original disk, snapshot, or clone as part of SIEM cleanup.

## 2. Wazuh All-in-One Installation

Wazuh 4.14.8 was installed on `siem-01` using the downloaded Wazuh installer:

```bash
sudo bash ~/wazuh-install.sh --all-in-one
```

An initial attempt encountered a transient `apt` lock. The installation succeeded on retry after the lock cleared.

The following components were subsequently verified active:

- `wazuh-manager`
- `wazuh-indexer`
- `wazuh-dashboard`

Dashboard address: `https://192.168.1.213`

The dashboard administrator password was retrieved locally from the protected installation archive `~/wazuh-install-files.tar` and was **not** committed to GitHub. Do not publish credentials or installation secrets.

## 3. Filebeat Troubleshooting and Recovery

**Symptom:** Filebeat failed because its registry was empty or missing `meta.json`.

**Recovery performed:** The affected registry was moved aside to `/var/lib/filebeat/registry.failed`; Filebeat's failed service state was cleared, and the service restarted.

```bash
sudo systemctl reset-failed filebeat
sudo systemctl restart filebeat
```

Filebeat then reported active. The output connectivity test succeeded:

```bash
sudo filebeat test output
```

Observed successful TLS connection to `https://127.0.0.1:9200`.

An initial Wazuh dashboard index-template warning subsequently cleared. This did **not** require reinstalling Wazuh.

## 4. Windows 11 Wazuh Agent

Windows endpoint: `WIN11` (observed IP `192.168.1.151`).

Installed Wazuh Windows agent version `4.14.8-1` from:

```text
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.8-1.msi
```

Manager: `192.168.1.213`  
Agent event connection: TCP `1514`  
Enrollment: TCP `1515`

An initial manually created agent record (`001`) was removed after a duplicate enrollment issue. Successful automatic enrollment was performed from elevated Windows PowerShell:

```powershell
& 'C:\Program Files (x86)\ossec-agent\agent-auth.exe' -m 192.168.1.213 -A WIN11
net start Wazuh
```

The enrolled endpoint was **agent ID `002`**, displayed as `WIN11` and Active in Wazuh. `Get-Service Wazuh` confirmed `WazuhSvc` was Running.

## 5. Sysmon Channel Collection

`Sysmon64` was running on the Windows endpoint. The local `Microsoft-Windows-Sysmon/Operational` log produced Sysmon events, including a process-termination Event ID `5`.

Agent configuration file:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

The original was backed up as `ossec.conf.bak`. A Sysmon `localfile` source was added inside the `ossec_config` root:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

**Note:** This XML is the relevant inserted fragment, not a replacement for the entire agent configuration.

The updated file was validated as XML in PowerShell, saved using UTF-8 encoding, and validated again from disk. The agent was restarted:

```powershell
Restart-Service Wazuh
```

The agent service remained Running.

### Agent-Side Verification

The Windows agent log (`ossec.log`) showed:

```text
2026/10/13 04:15:50 wazuh-agent: INFO: (1951): Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'.
2026/10/13 04:15:50 wazuh-agent: INFO: (4102): Connected to the server ([192.168.1.213]:1514/tcp).
```

These messages verify that the agent was configured to analyze the Sysmon channel and had a connection to the manager; server-side alerts below establish that Sysmon-related detections reached the SIEM.

## 6. Server-Side Detection Verification

On `siem-01`, the alert file was inspected:

```bash
sudo grep -i -m 3 'sysmon' /var/ossec/logs/alerts/alerts.json
```

**Interpretation caution:** The first match contained `SysmonLog` in an SCA policy description and was **not** a Sysmon event. Subsequent matches were actual Windows Sysmon alerts from `WIN11`, agent `002`, decoded as `windows_eventchannel`, with provider `Microsoft-Windows-Sysmon` and channel `Microsoft-Windows-Sysmon/Operational`.

Confirmed examples:

| Sysmon Event ID | Wazuh rule | Level | Rule description |
|---|---|---|---|
| 1 | `92021` | 3 | Powershell was used to delete files or directories |
| 11 | `92205` | 9 | Powershell process created an executable file in Windows root folder |

These are **alert rule descriptions**, not proof of malicious activity.

A point-in-time matching alert count was obtained using:

```bash
sudo grep -c '"groups":\["sysmon"' /var/ossec/logs/alerts/alerts.json
```

**Observed result: `32`.**

This counts lines containing the matching beginning of the `rule.groups` array, **not** every raw Sysmon event and not necessarily all Sysmon alerts. The server-side matching-alert count screenshot is uploaded as [Complete pipeline.PNG](Complete%20pipeline.PNG).

## 7. Dashboard Search Troubleshooting and Resolution

Early Wazuh Threat Hunting searches for Sysmon-specific events returned no results with these filters:

```text
agent.name:"WIN11" AND win.system.providerName:"Microsoft-Windows-Sysmon"
agent.name:"WIN11" AND Sysmon
win.system.channel:"Microsoft-Windows-Sysmon/Operational"
```

These were historical unsuccessful queries; the server-side `alerts.json` had already independently confirmed Sysmon alert generation and storage. The precise reason those initial dashboard searches failed was not established.

**Subsequent dashboard validation (2026-10-13):** Threat Hunting → Events successfully returned **10** matching alerts for `rule.id:92205` with Last 24 hours selected, and **8** matching alerts for `rule.id:92021` with Last 7 days selected.

**Direct field-search validation (Last 24 hours):**

| Dashboard query | Matching alerts |
|---|---:|
| `data.win.system.eventID:11` | 92 |
| `data.win.system.providerName:"Microsoft-Windows-Sysmon"` | 113 |
| `data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"` | 113 |

These point-in-time search hit counts confirm that Sysmon-related Wazuh alerts were searchable through the dashboard using the validated fields. They are **not** counts of all raw Sysmon events and do not establish complete ingestion coverage or malicious activity. The earlier search failure is no longer reproducible with these validated queries, although its original cause remains unknown. See [validation.md](validation.md) for the supporting evidence.

## 8. Operational Notes and Next Actions

- The Windows VM console did not support convenient clipboard paste; commands were entered manually. Preserve short, carefully checked PowerShell instructions for future Windows changes.
- Continue to protect Wazuh credentials and enrollment secrets.
- Completed: uploaded the [server-side Sysmon alert-count screenshot](Complete%20pipeline.PNG) and linked it in [validation.md](validation.md).
- Completed: validated Sysmon-specific Threat Hunting searches by rule ID, event ID, provider, and channel; see [Section 7](#7-dashboard-search-troubleshooting-and-resolution) and [validation.md](validation.md) for point-in-time results. The original unsuccessful queries remain documented; their cause was not established.
- Expand endpoint onboarding only after separate configuration and validation.

## Result

**Validated:** Windows Sysmon event source → Wazuh Windows agent `002` → Wazuh manager on `siem-01` → Sysmon-specific rules and centrally stored alerts.

**Not yet validated:** Complete raw Sysmon event ingestion coverage or additional endpoint onboarding. Sysmon-specific Wazuh alert searches have been successfully validated in Threat Hunting.
