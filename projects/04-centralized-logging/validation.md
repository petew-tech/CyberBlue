# Project 04 — Centralized Logging & SIEM Foundations
## Module 05 — Validation Record

**Validation date:** 2026-10-13  
**Status:** In progress — Windows Sysmon-to-Wazuh alerting validated

### Environment

| Component | Observed configuration |
|---|---|
| Virtualization host | TrueNAS SCALE 24.10.1 |
| SIEM VM | `SIEM_01`, hostname `siem-01` |
| SIEM OS | Ubuntu Server 24.04.5 LTS |
| Wazuh | 4.14.8 all-in-one |
| SIEM IP | `192.168.1.213` |
| Windows endpoint | `WIN11`, Wazuh agent ID `002` |
| Windows endpoint IP | `192.168.1.151` |
| Windows event source | `Microsoft-Windows-Sysmon/Operational` |

### Validation Results

| ID | Test | Evidence observed | Result |
|---|---|---|---|
| V-01 | SIEM services | Wazuh manager, indexer, and dashboard services active; Filebeat recovered and active | PASS |
| V-02 | Filebeat connectivity | `sudo filebeat test output` confirmed TLS connection to `https://127.0.0.1:9200` | PASS |
| V-03 | Windows agent enrollment | `WIN11` enrolled and reported Active as agent `002` | PASS |
| V-04 | Windows agent service | `Get-Service Wazuh` showed `WazuhSvc` Running | PASS |
| V-05 | Local Sysmon service | `Sysmon64` running; local Sysmon Operational log produced an Event ID 5 record | PASS |
| V-06 | Agent Sysmon configuration | Agent `ossec.conf` backed up; Sysmon Operational channel added as `eventchannel`; saved XML validated | PASS |
| V-07 | Event channel collection | Windows agent log reported `Analyzing event log: 'Microsoft-Windows-Sysmon/Operational'` | PASS |
| V-08 | Manager connection | Agent log reported `Connected to the server ([192.168.1.213]:1514/tcp)` | PASS |
| V-09 | Centralized Sysmon detection | SIEM `/var/ossec/logs/alerts/alerts.json` contained actual Sysmon alerts from agent `002` / `WIN11` | PASS |
| V-10 | Sysmon process event detection | Sysmon Event ID 1, Wazuh rule `92021`, level 3, present in manager alert log | PASS |
| V-11 | Sysmon file event detection | Sysmon Event ID 11, Wazuh rule `92205`, level 9, present in manager alert log | PASS |
| V-12 | Sysmon alert count | Server-side matching-alert command returned `32` at test time | PASS |
| V-13 | Initial Threat Hunting UI queries | Initial Sysmon field/text queries returned no matching results | FOLLOW-UP |
| V-14 | Dashboard Sysmon rule lookup | Threat Hunting Events search `rule.id:92205` returned 10 matching alerts for `WIN11`, rule level 9, within Last 24 hours | PASS |

### Server-side Verification Commands

To inspect matching alert records:

```bash
sudo grep -i -m 3 'sysmon' /var/ossec/logs/alerts/alerts.json
```

**Important:** A match on the text `SysmonLog` in an SCA policy was *not* evidence of a Sysmon event. Later matching records contained the actual Windows event channel and provider fields, with decoder `windows_eventchannel`, agent `WIN11`, and Sysmon-specific Wazuh rules.

To count alert entries whose `rule.groups` begins with `sysmon`:

```bash
sudo grep -c '"groups":\["sysmon"' /var/ossec/logs/alerts/alerts.json
```

**Observed output:** `32`

This is a **point-in-time count of matching alert entries**, not the total number of raw Sysmon events, nor proof that every Sysmon event is indexed or displayed.

### Confirmed Alert Examples

| Sysmon Event ID | Wazuh Rule | Level | Wazuh rule description |
|---|---|---|---|
| 1 | 92021 | 3 | Powershell was used to delete files or directories |
| 11 | 92205 | 9 | Powershell process created an executable file in Windows root folder |

These are **rule descriptions**, not independent confirmation of malicious activity.

### Dashboard Follow-up

**2026-10-13 — Partial resolution:** Threat Hunting → Events, with **Last 24 hours** selected, successfully returned **10** matching alerts for the query `rule.id:92205`. The screenshot showed `WIN11`, rule `92205`, level `9`, and included the previously verified alert at `06:15:57.717`.

This confirms that the Sysmon-related rule **is searchable in the dashboard by rule ID**. Earlier Sysmon provider/channel/text queries did not return results; their field mapping or syntax remains to be investigated separately. A `manager.name: siem-01` filter chip was still visible in the successful search screenshot.

### Evidence Handling

Screenshot evidence: [Complete pipeline.PNG](Complete%20pipeline.PNG) — server-side Sysmon matching-alert count (`32`) captured during validation.

Dashboard screenshot: [Wazuh-Sysmon-Rule-92205.PNG](Wazuh-Sysmon-Rule-92205.PNG) — Threat Hunting Events search showing 10 matching rule `92205` alerts. Screenshot uploaded during the lab session; verify the link resolves.

### Scope of Validation

**Validated:** Sysmon telemetry from the Windows endpoint was collected by the Wazuh agent and produced centrally stored Wazuh alerts.  
**Not yet validated:** Sysmon-specific dashboard search results, complete raw event ingestion coverage, or additional endpoint onboarding.
