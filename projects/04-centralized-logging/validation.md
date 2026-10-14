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
| V-13 | Initial Threat Hunting UI queries | Initial Sysmon field/text queries returned no matching results; subsequent direct Event ID, provider, and channel searches succeeded (V-18–V-20) | RESOLVED |
| V-14 | Dashboard Sysmon rule lookup | Threat Hunting Events search `rule.id:92205` returned 10 matching alerts for `WIN11`, rule level 9, within Last 24 hours | PASS |
| V-15 | Dashboard second Sysmon rule lookup | Threat Hunting Events search `rule.id:92021` returned 8 matching alerts for `WIN11`, rule level 3, with Last 7 days selected | PASS |
| V-16 | Indexed Sysmon event fields | Opened a rule `92021` alert in Discover; verified provider `Microsoft-Windows-Sysmon`, channel `Microsoft-Windows-Sysmon/Operational`, Event ID `1`, decoder `windows_eventchannel`, and agent `002` / `WIN11` | PASS |
| V-17 | Indexed Sysmon File Create fields | Threat Hunting rule `92205` returned 10 matching alerts; inspected a record and verified Sysmon Event ID `11`, provider `Microsoft-Windows-Sysmon`, channel `Microsoft-Windows-Sysmon/Operational`, and agent `WIN11` | PASS |
| V-18 | Direct Event ID field search | `data.win.system.eventID:11` returned 92 matching alerts in Last 24 hours | PASS |
| V-19 | Direct Sysmon provider search | `data.win.system.providerName:"Microsoft-Windows-Sysmon"` returned 113 matching alerts in Last 24 hours | PASS |
| V-20 | Direct Sysmon channel search | `data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"` returned 113 matching alerts in Last 24 hours | PASS |
| V-21 | Windows agent health and identity | Wazuh agent `002` / `WIN11` showed Active, IP `192.168.1.151`, version `4.14.8`, group `default`, Windows 11 Pro `10.0.26200.9457`, cluster node `node01`, registration Oct 12 2026 19:13:15, and last keep alive Oct 13 2026 07:25:50 (dashboard display time) | PASS |
| V-22 | Windows CIS password-history remediation | Wazuh CIS check `26000` initially Failed with local password history `0`; Windows policy changed to `24` and independently verified; subsequent SCA scan completed Oct 13 2026 07:40:28 (dashboard time), check Passed, passed count 124→125, failed count 349→348, not applicable remained 9; displayed score stayed 26% due to rounding | PASS |
| V-23 | Windows CIS minimum-password-age remediation | Wazuh CIS check `26002` initially Failed with local minimum password age `0` days; changed to `1` day and independently verified using `net accounts`; subsequent Wazuh SCA check Passed, with totals changing from 125 Passed / 348 Failed / 9 Not applicable to 126 Passed / 347 Failed / 9 Not applicable after dashboard refresh | PASS |
| V-24 | Windows CIS minimum-password-length remediation | Wazuh CIS check `26003` initially Failed with local minimum password length `0` characters; Windows policy changed to `14` characters and independently verified with `net accounts`; subsequent Wazuh SCA check Passed, with totals changing from 126 Passed / 347 Failed / 9 Not applicable to 127 Passed / 346 Failed / 9 Not applicable | PASS |
| V-25 | Windows CIS relaxed minimum password length limits remediation | Wazuh CIS check `26004` initially Failed with local policy Not Defined; policy set to Enabled and independently verified via registry `HKLM:\SYSTEM\CurrentControlSet\Control\SAM\RelaxMinimumPasswordLengthLimits` = `1`; subsequent Wazuh SCA check Passed, with totals changing from 127 Passed / 346 Failed / 9 Not Applicable to 128 Passed / 345 Failed / 9 Not Applicable | PASS |
| V-26 | Windows CIS account lockout duration remediation | Wazuh CIS check `26005` initially Failed with local account lockout duration `10` minutes; Windows policy changed to `15` minutes and independently verified with `net accounts`; subsequent Wazuh SCA check Passed, with totals changing from 128 Passed / 345 Failed / 9 Not applicable to 129 Passed / 344 Failed / 9 Not applicable; displayed score 27%, scan ended Oct 13 2026 11:49:29 (dashboard time) | PASS |
| V-27 | Windows CIS account lockout threshold remediation | Wazuh CIS check `26006` initially Failed with local lockout threshold `10` invalid attempts; changed to `5` and independently verified with `net accounts` (lockout duration `15` minutes, observation window `10` minutes); subsequent SCA summary showed 130 Passed / 343 Failed / 9 Not applicable, score 27%, scan completed Oct 13 2026 11:58:25 (dashboard time) | PASS |
| V-28 | Windows CIS reset account lockout counter remediation | Wazuh CIS check `26007` previously Failed with reset account lockout counter set to `10` minutes; Windows policy changed to `15` minutes and independently verified using `net accounts` (threshold `5`, duration `15`, observation window `15`); following agent restart, Wazuh SCA totals changed from 130 Passed / 343 Failed / 9 Not applicable to 131 Passed / 342 Failed / 9 Not applicable; displayed score 27%, scan ended Oct 13 2026 12:06:20 (dashboard time) | PASS |
| V-29 | Windows CIS block Microsoft accounts remediation | Wazuh CIS check `26008` initially Failed with **Accounts: Block Microsoft accounts** set to Not Defined; Windows local security policy changed to **Users can't add or log on with Microsoft accounts**, and registry `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\NoConnectedUser` independently verified as `3`; following Wazuh agent restart, SCA totals changed from 131 Passed / 342 Failed / 9 Not applicable to 132 Passed / 341 Failed / 9 Not applicable; displayed score 27%, scan ended Oct 13 2026 12:24:09 (dashboard time) | PASS |
| V-30 | Windows CIS rename built-in Administrator account remediation | Wazuh CIS check `26011` initially Failed; the built-in `Administrator` account (SID ending `-500`) was renamed to `CBLocalRecovery` using elevated PowerShell. `Get-LocalUser` verified the renamed account retained SID ending `-500` and remained disabled; `cyberblue` remained enabled. Following Wazuh agent restart, SCA totals changed from 132 Passed / 341 Failed / 9 Not applicable to 133 Passed / 340 Failed / 9 Not applicable; displayed score increased from 27% to 28%, scan ended Oct 14 2026 09:37:20 (dashboard time) | PASS |
| V-31 | Windows CIS rename built-in Guest account remediation | Wazuh CIS check `26012` initially Failed; the built-in `Guest` account (SID ending `-501`) was renamed to `CBLocalVisitor` using elevated PowerShell. `Get-LocalUser` verified the renamed account retained SID ending `-501` and remained disabled; `cyberblue` remained enabled. After a successful Wazuh agent restart and subsequent SCA scan, check `26012` Passed; SCA totals changed from 133 Passed / 340 Failed / 9 Not applicable to 134 Passed / 339 Failed / 9 Not applicable; displayed score remained 28%, scan ended Oct 14 2026 00:48:27 (dashboard time) | PASS |
| V-32 | Windows CIS advanced audit policy subcategory override remediation | Wazuh CIS check `26013` initially Failed with **Audit: Force audit policy subcategory settings (Windows Vista or later) to override audit policy category settings** Not Defined; Windows Local Security Policy changed to Enabled and registry `HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\SCENoApplyLegacyAuditPolicy` independently verified as `1`. Wazuh check subsequently reported Passed; SCA totals changed from 134 Passed / 339 Failed / 9 Not applicable to 135 Passed / 338 Failed / 9 Not applicable; score remained 28%, scan ended Oct 14 2026 14:13:08 (dashboard time) | PASS |
| V-33 | Windows CIS printer driver installation restriction remediation | Wazuh CIS check `26015` initially Failed with **Devices: Prevent users from installing printer drivers** set to Disabled. Windows Local Security Policy changed to Enabled; registry `HKLM:\SYSTEM\CurrentControlSet\Control\Print\Providers\LanMan Print Services\Servers\AddPrinterDrivers` independently verified as `1`. After Wazuh agent restart, check `26015` reported Passed; SCA totals changed from 135 Passed / 338 Failed / 9 Not applicable to 136 Passed / 337 Failed / 9 Not applicable; score remained 28%, scan ended Oct 14 2026 14:19:36 (dashboard time) | PASS |
| V-34 | Windows CIS secure interactive logon remediation | Wazuh CIS check `26022` initially Failed with **Interactive logon: Do not require CTRL+ALT+DEL** Not Defined. Windows Local Security Policy changed to Disabled; registry `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\DisableCAD` independently verified as `0`. Wazuh subsequently displayed check `26022` as Passed, confirmed by individual check screenshot. SCA summary at scan end Oct 14 2026 14:26:25 (dashboard time) remained 136 Passed / 337 Failed / 9 Not applicable, score 28%; no overall count increase is attributed to this remediation | PASS |
| V-35 | Windows CIS last signed-in username protection remediation | Wazuh CIS check `26023` initially Failed with **Interactive logon: Don't display last signed-in** set to Disabled. Windows Local Security Policy changed to Enabled; registry `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\DontDisplayLastUserName` independently verified as `1`. Following Wazuh agent restart, check `26023` was reported Passed. SCA summary changed from 136 Passed / 337 Failed / 9 Not applicable (28%) to 138 Passed / 335 Failed / 9 Not applicable (29%), scan ended Oct 14 2026 14:40:05 (dashboard time); the two-check net increase is not attributed solely to check `26023` | PASS |
| V-36 | Windows CIS machine account lockout threshold remediation | Wazuh CIS check `26024` initially Failed with **Interactive logon: Machine account lockout threshold** Not Defined. Windows Local Security Policy configured to 10 invalid logon attempts; registry `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\MaxDevicePasswordFailedAttempts` independently verified as `10`. After Wazuh agent restart and subsequent assessment, check `26024` reported Passed. SCA totals changed from 138 Passed / 335 Failed / 9 Not applicable to 139 Passed / 334 Failed / 9 Not applicable; score remained 29%, scan ended Oct 14 2026 14:45:25 (dashboard time) | PASS |
| V-37 | Windows CIS machine inactivity limit remediation | Wazuh CIS check `26025` initially Failed with **Interactive logon: Machine inactivity limit** Not Defined. Windows Local Security Policy configured to 900 seconds (15 minutes); registry `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\InactivityTimeoutSecs` independently verified as `900`. Following Wazuh agent restart, check `26025` reported Passed. SCA totals changed from 139 Passed / 334 Failed / 9 Not applicable to 140 Passed / 333 Failed / 9 Not applicable; score remained 29%, scan ended Oct 14 2026 14:54:30 (dashboard time) | PASS |

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

**2026-10-13 — Additional confirmation:** With **Last 7 days** selected, the Events search `rule.id:92021` returned **8** alerts for `WIN11`, rule level **3**, including the previously confirmed record at `06:15:50.873`. The same search had returned no results with Last 24 hours selected; the cause of that discrepancy is not yet established.

This confirms that the Sysmon-related rule **is searchable in the dashboard by rule ID**. Earlier Sysmon provider/channel/text queries did not return results, but subsequent direct Event ID, provider, and channel searches succeeded. The cause of the earlier failed searches was not established; the issue is no longer reproducible for these three fields. A `manager.name: siem-01` filter chip was still visible in the successful search screenshot.

### Direct Field-Search Follow-up

**2026-10-13 — Successful direct searches (Last 24 hours):** `data.win.system.eventID:11` returned **92** matching alerts; `data.win.system.providerName:"Microsoft-Windows-Sysmon"` returned **113**; and `data.win.system.channel:"Microsoft-Windows-Sysmon/Operational"` returned **113**. These are dashboard search hit counts at the time of each query, not counts of unique malicious activities. This resolves the practical field-search validation gap, without establishing why the earlier attempts returned no results.

### Evidence Handling

Screenshot evidence: [Complete pipeline.PNG](Complete%20pipeline.PNG) — server-side Sysmon matching-alert count (`32`) captured during validation.

Detailed indexed-event evidence (Word document): [Wazuh-Sysmon-Event-Details.png.docx](Wazuh-Sysmon-Event-Details.png.docx) — event fields from Wazuh Discover, including Sysmon provider, channel, and rule `92021`.

Dashboard screenshot: [Wazuh-Sysmon-Rule-92021.png](Wazuh-Sysmon-Rule-92021.png) — Threat Hunting Events search showing 8 matching rule `92021` alerts (Last 7 days).

Detailed Sysmon Event ID 11 evidence (Word document): [Wazuh-Sysmon-EventID11-Details.docx](Wazuh-Sysmon-EventID11-Details.docx) — indexed File Create alert fields verified in Wazuh Threat Hunting.

Dashboard screenshot: [Wazuh-Sysmon-Rule-92205.png](Wazuh-Sysmon-Rule-92205.png) — Threat Hunting Events search showing 10 matching rule `92205` alerts. Screenshot uploaded during the lab session.

CIS remediation after-scan screenshot: [Wazuh-CIS-Remediation-After.png](Wazuh-CIS-Remediation-After.png) — Wazuh Windows 11 SCA summary after the password-history control was corrected, showing 125 Passed, 348 Failed, 9 Not applicable, and scan completion at 07:40:28 on 2026-10-13 (dashboard time).

Minimum-password-age remediation after-scan screenshot: [Wazuh-CIS-Minimum-Password-Age-After.PNG](Wazuh-CIS-Minimum-Password-Age-After.PNG) — Wazuh Windows 11 SCA summary showing 126 Passed, 347 Failed, and 9 Not applicable after CIS check `26002` passed.

Minimum-password-length remediation after-scan screenshot: [Wazuh-CIS-Minimum-Password-Length-After.PNG](Wazuh-CIS-Minimum-Password-Length-After.PNG) — Wazuh Windows 11 SCA summary after CIS check `26003` passed, showing 127 Passed, 346 Failed, and 9 Not applicable.

Relaxed password length limits remediation screenshot: [Wazuh-CIS-Relax-Password-Length-After.PNG](Wazuh-CIS-Relax-Password-Length-After.PNG) — Wazuh WIN11 CIS assessment summary after check `26004` passed, showing 128 Passed, 345 Failed, and 9 Not Applicable.

Account lockout duration remediation screenshot: [Wazuh-CIS-Account-Lockout-Duration-After.PNG](Wazuh-CIS-Account-Lockout-Duration-After.PNG) — Wazuh WIN11 CIS assessment summary showing 129 Passed, 344 Failed, 9 Not applicable, score 27%, scan completed Oct 13 2026 11:49:29 (dashboard time).

Account lockout threshold remediation screenshot: [Wazuh-CIS-Account-Lockout-Threshold-After.PNG](Wazuh-CIS-Account-Lockout-Threshold-After.PNG) — Wazuh WIN11 CIS assessment summary showing 130 Passed, 343 Failed, 9 Not applicable, score 27%, scan completed Oct 13 2026 11:58:25 (dashboard time).

Account lockout counter remediation screenshot: [Wazuh-CIS-Account-Lockout-Counter-After.PNG](Wazuh-CIS-Account-Lockout-Counter-After.PNG) — Wazuh WIN11 CIS assessment summary showing 131 Passed, 342 Failed, 9 Not applicable, score 27%, scan completed Oct 13 2026 12:06:20 (dashboard time).

Block Microsoft accounts remediation screenshot: [Wazuh-CIS-Block-Microsoft-Accounts-After.PNG](Wazuh-CIS-Block-Microsoft-Accounts-After.PNG) — Wazuh WIN11 CIS assessment summary showing 132 Passed, 341 Failed, 9 Not applicable, score 27%, scan completed Oct 13 2026 12:24:09 (dashboard time).

Rename Administrator account remediation screenshot: [Wazuh-CIS-Rename-Administrator-After.PNG](Wazuh-CIS-Rename-Administrator-After.PNG) — Wazuh WIN11 CIS assessment summary showing 133 Passed, 340 Failed, 9 Not applicable, score 28%, scan completed Oct 14 2026 09:37:20 (dashboard time).

Rename Guest account remediation screenshot: [Wazuh-CIS-Rename-Guest-After.PNG](Wazuh-CIS-Rename-Guest-After.PNG) — Wazuh WIN11 CIS assessment summary showing 134 Passed, 339 Failed, 9 Not applicable, score 28%, scan completed Oct 14 2026 00:48:27 (dashboard time).

Advanced audit policy override remediation screenshot: [Wazuh-CIS-Advanced-Audit-Policy-After.PNG](Wazuh-CIS-Advanced-Audit-Policy-After.PNG) — Wazuh WIN11 CIS assessment summary showing 135 Passed, 338 Failed, 9 Not applicable, score 28%, scan completed Oct 14 2026 14:13:08 (dashboard time).

Printer driver installation restriction remediation screenshot: [Wazuh-CIS-Printer-Driver-Restriction-After.PNG](Wazuh-CIS-Printer-Driver-Restriction-After.PNG) — Wazuh WIN11 CIS assessment summary showing 136 Passed, 337 Failed, 9 Not applicable, score 28%, scan completed Oct 14 2026 14:19:36 (dashboard time).

Secure interactive logon remediation screenshot: [Wazuh-CIS-Secure-Interactive-Logon-After.PNG](Wazuh-CIS-Secure-Interactive-Logon-After.PNG) — Wazuh WIN11 CIS check `26022` showing Passed; assessment summary remained 136 Passed / 337 Failed / 9 Not applicable, score 28% at scan end Oct 14 2026 14:26:25 (dashboard time).

Last signed-in user protection remediation screenshot: [Wazuh-CIS-Hide-Last-Signed-In-After.PNG](Wazuh-CIS-Hide-Last-Signed-In-After.PNG) — evidence for Wazuh WIN11 CIS check `26023` reported Passed. Assessment summary at Oct 14 2026 14:40:05 (dashboard time): 138 Passed / 335 Failed / 9 Not applicable, score 29%; the net change of two checks is not attributed solely to this remediation.

Machine account lockout threshold remediation screenshot: [Wazuh-CIS-Machine-Account-Lockout-After.PNG](Wazuh-CIS-Machine-Account-Lockout-After.PNG) — evidence for Wazuh WIN11 CIS check `26024` reported Passed. Assessment summary at Oct 14 2026 14:45:25 (dashboard time): 139 Passed / 334 Failed / 9 Not applicable, score 29%.

Machine inactivity limit remediation screenshot: [Wazuh-CIS-Machine-Inactivity-Limit-After.PNG](Wazuh-CIS-Machine-Inactivity-Limit-After.PNG) — evidence for Wazuh WIN11 CIS check `26025` reported Passed. Assessment summary at Oct 14 2026 14:54:30 (dashboard time): 140 Passed / 333 Failed / 9 Not applicable, score 29%.

### Scope of Validation

**Validated:** Sysmon telemetry from the Windows endpoint was collected by the Wazuh agent and produced centrally stored Wazuh alerts.  
**Not yet validated:** Complete raw event ingestion coverage or additional endpoint onboarding.
