# Module 04 — Endpoint Telemetry Validation

## Validation Status

PASS — controlled endpoint telemetry exercises completed on Windows 11 and Ubuntu Server.

---

## 1. Native Windows Telemetry

### Authentication / Logon Activity
- Windows Security auditing was configured for the required logon/logoff activity.
- Controlled activity was verified in the Windows Security log.
- Result: PASS

### Process Creation
- Security Event ID 4688 was generated.
- Controlled process activity was identified.
- Result: PASS

### Service Installation
- Controlled service `CyberBlueTelemetryTest` was created.
- Windows Security Event ID 4697 confirmed the service installation.
- The test service was subsequently removed.
- Result: PASS

### File Activity
- File System auditing was enabled with a targeted SACL.
- Security Event ID 4663 captured access to:
  `C:\CyberBlue-File-Test.txt`
- The event identified the user, file, process, and requested access.
- Result: PASS

---

## 2. Native Linux Telemetry

### Authentication / Privilege Activity
- Ubuntu `auth.log` captured sudo activity.
- User and command execution were visible.
- Result: PASS

### Service Activity
- systemd journal activity was observed for the cron service.
- Result: PASS

### Native File Visibility Baseline
- A controlled file was created.
- Searching the native journal by filename produced no matching event.
- Result: Baseline demonstrated limited file visibility.

---

## 3. Enhanced Linux Telemetry — auditd

### Installation
- auditd 4.1.2 installed successfully.
- Service verified active.
- Result: PASS

### File Attribute Telemetry
A controlled `chmod` operation generated an audit event containing:

- File path
- User / audit user
- Process
- Executable
- Syscall
- Success status
- Audit key

Example:

`/home/cyberblue/cyberblue-file-test.txt`

Result: PASS

### Audit Rule Cleanup
- Temporary audit rules removed.
- `auditctl -l` returned `No rules`.
- Test files removed.
- Result: PASS

---

## 4. Enhanced Windows Telemetry — Sysmon

### Installation
- Sysmon v15.22 installed.
- Sysmon64 service verified Running / Automatic.
- Microsoft Authenticode signature verified as valid.
- Result: PASS

### Process Creation — Event ID 1
Controlled Calculator execution generated Sysmon Event ID 1.

Observed telemetry included:

- Process image
- Command line
- User
- Process GUID
- Parent process
- Parent command line
- Integrity level
- SHA-256 hash

Result: PASS

### Network Connection — Event ID 3
Controlled TCP connection:

`10.10.30.50 → 10.10.30.127:3389`

Sysmon identified:

- PowerShell process
- User
- Source IP
- Destination IP
- Destination port
- Protocol
- Initiated status

Result: PASS

### File Creation — Event ID 11
Controlled file creation generated Event ID 11.

Observed telemetry included:

- Target filename
- Creating process
- Process ID
- Process GUID
- User
- UTC timestamp

Result: PASS

---

## 5. Controlled Telemetry Failure / Recovery

### Linux

Sequence:

1. auditd confirmed active.
2. auditd deliberately stopped.
3. File created during the blind period.
4. auditd restarted.
5. Service returned to active.
6. No audit rules were present.
7. A new audit rule was deliberately restored.
8. Controlled `chmod` generated an audit event.
9. Temporary rule removed.
10. Test file removed.

Result: PASS

### Windows

Sequence:

1. Sysmon confirmed Running / Automatic.
2. Normal Sysmon configuration backed up.
3. File Create telemetry deliberately suppressed.
4. Controlled file created during blind period.
5. No matching Event ID 11 was recorded.
6. Known-good configuration restored.
7. New controlled file created.
8. Sysmon Event ID 11 returned.
9. Temporary blind configuration removed.
10. Test files removed.
11. Sysmon returned to the known-good configuration.

Result: PASS

### Windows

Sequence:

1. Sysmon confirmed Running / Automatic.
2. Normal Sysmon configuration backed up.
3. File Create telemetry deliberately suppressed.
...
---

## 6. Controlled Timeline

The following Sysmon events were correlated chronologically:

| Time | Event | Activity |
|---|---:|---|
| 2:13:06 PM | 1 | Calculator process created |
| 2:14:27 PM | 3 | TCP connection to Kali :3389 |
| 2:16:31 PM | 11 | CyberBlue test file created |

Result: PASS

---

## 7. Final Endpoint State

### Windows
- Sysmon64: Running
- Startup: Automatic
- Known-good configuration restored
- Temporary test artifacts removed

### Ubuntu
- auditd: Active
- Temporary audit rules removed
- Temporary test artifacts removed

---

## Overall Result

**PASS**

The lab demonstrated the difference between native endpoint telemetry and enhanced telemetry, including controlled telemetry loss, recovery, and validation.
