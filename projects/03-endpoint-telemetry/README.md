
# Project 03 — Endpoint Telemetry & Logging

## Overview

This project establishes endpoint visibility across Windows and Linux systems in the CyberBlue lab.

The objective was to understand what endpoint activity is visible through native operating-system logging, identify visibility gaps, deploy enhanced telemetry, and validate that security-relevant activity can be reconstructed from endpoint evidence.

The project uses:

- Windows 11
- Ubuntu Server
- Windows Event Viewer
- Windows Security auditing
- Linux journald
- Linux authentication logs
- Sysmon
- Linux auditd
- PowerShell
- CyberBlue segmented lab networking

---

## Lab Environment

| System | Role |
|---|---|
| Windows 11 | Windows endpoint |
| Ubuntu Server | Linux endpoint / CyberBlue management server |
| Kali Linux | Security testing endpoint |
| TrueNAS SCALE | Virtualization platform |

The Windows and Linux endpoints were connected to the CyberBlue lab network created during the previous networking and segmentation project.

---

## Objectives

The project focused on the following objectives:

- Examine native Windows and Linux telemetry.
- Generate known endpoint activity.
- Locate corresponding security events.
- Validate authentication and administrative activity.
- Observe service activity.
- Examine process and file activity.
- Identify native logging limitations.
- Deploy Sysmon for enhanced Windows telemetry.
- Deploy auditd for enhanced Linux telemetry.
- Correlate endpoint activity into a timeline.
- Deliberately create a telemetry blind spot.
- Investigate the resulting visibility gap.
- Restore telemetry and validate recovery.
- Preserve evidence for repeatable analysis.

---

## Native Windows Telemetry

Windows Security and System logs were examined before enhanced telemetry was introduced.

Controlled activity demonstrated visibility into several security-relevant actions.

### Authentication Activity

Windows auditing was used to observe workstation authentication activity.

A controlled workstation unlock generated:

- Event ID 4801
- User identity
- Workstation identity
- Session information

### Process Activity

Process Creation auditing was enabled and validated.

A controlled process generated:

- Event ID 4688
- Process path
- User context
- Process information

### Service Activity

A temporary service named:

`CyberBlueTelemetryTest`

was created for the exercise.

Windows System Event ID 7045 recorded the service installation, including:

- Service name
- Service executable
- Service type
- Startup type
- Service account

The temporary service was removed after validation.

### File Activity

Windows File System auditing and a targeted SACL were used to monitor:

`C:\CyberBlue-File-Test.txt`

Security Event ID 4663 recorded:

- Target file
- User
- Process
- Access type
- Access mask

The test file and its associated SACL were removed after validation.

---

## Native Linux Telemetry

Native Linux telemetry was examined using:

- `journalctl`
- `/var/log/auth.log`
- systemd service logs
- traditional login databases

### SSH Authentication

A controlled SSH connection from the CyberBlue Kali endpoint to Ubuntu generated an SSH authentication event.

The event identified:

- Successful authentication
- User account
- Source lab IP
- SSH session establishment

### Administrative Activity

A controlled:

`sudo whoami`

operation was identified in `/var/log/auth.log`.

The log recorded:

- User
- Terminal
- Working directory
- Target user
- Executed command

### Service Activity

The `cron` service was deliberately restarted.

The systemd journal recorded:

- Service stopping
- Successful deactivation
- Service stopped
- Service started

### Native File Visibility Gap

A controlled Linux file was created and the native journal was searched for the filename.

No matching journal entry was found under the current configuration.

This demonstrated that ordinary file activity was not automatically visible through the journal search used in the exercise.

---

## Enhanced Windows Telemetry — Sysmon

Sysmon 15.22 was installed after validating the Microsoft digital signature.

The configuration enabled:

- Process creation
- Network connections
- File creation
- SHA-256 hashing

### Event ID 1 — Process Creation

A controlled Calculator launch produced Sysmon Event ID 1.

The event included:

- Process image
- Command line
- User
- Process ID
- Process GUID
- Integrity level
- SHA-256 hash
- Parent process
- Parent command line

### Event ID 3 — Network Connection

A controlled TCP connection was generated from the Windows endpoint:

`10.10.30.50 → 10.10.30.127:3389`

Sysmon recorded:

- PowerShell as the initiating process
- User
- TCP protocol
- Source address and port
- Destination address and port
- Connection initiation state

### Event ID 11 — File Creation

A controlled file creation generated Sysmon Event ID 11.

The event identified:

- Target filename
- Creating process
- Process ID
- Process GUID
- User
- Creation timestamp

---

## Enhanced Linux Telemetry — auditd

Linux `auditd` 4.1.2 was installed and validated.

A targeted runtime audit rule was used to monitor a controlled test file.

A controlled `chmod` operation generated audit telemetry containing:

- File path
- Audit user
- Process
- Executable
- Syscall
- Success status
- Session information
- Audit key

The exercise also demonstrated an important limitation: an earlier shell append did not generate the expected keyed event under the tested path rule, while the controlled file attribute change did.

Temporary audit rules and test files were removed after validation.

---

## Controlled Activity Timeline

Sysmon telemetry was used to correlate several controlled Windows activities.

| Time | Event ID | Activity |
|---|---:|---|
| 2:13:06 PM | 1 | Calculator process created |
| 2:14:27 PM | 3 | TCP connection to Kali on port 3389 |
| 2:16:31 PM | 11 | CyberBlue test file created |

This demonstrated how process, network, and file telemetry can be correlated chronologically during an investigation.

---

## Telemetry Failure and Visibility Gap

A major objective of this project was demonstrating that a running endpoint does not necessarily mean complete telemetry is available.

### Linux

`auditd` was deliberately stopped.

Controlled activity occurred while the telemetry source was unavailable.

The service was subsequently restarted and verified active.

A new audit rule and controlled `chmod` operation demonstrated that audit telemetry had been restored.

### Windows

Sysmon remained operational while a temporary configuration deliberately suppressed File Create telemetry.

During the blind period, the following file was created:

`C:\CyberBlue\sysmon-blind-test.txt`

A targeted search found no corresponding Sysmon Event ID 11.

This demonstrated a telemetry blind spot:

> The endpoint activity occurred, but the expected security telemetry was unavailable.

The known-good Sysmon configuration was then restored.

A second controlled file:

`C:\CyberBlue\sysmon-restored-test.txt`

generated a valid Sysmon Event ID 11, confirming that File Create visibility had returned.

---

## Key Findings

This project demonstrated several important SOC concepts:

1. Native operating-system logs provide useful security evidence but do not provide complete endpoint visibility.

2. Logging configuration matters as much as whether a logging service is running.

3. Sysmon substantially improves Windows process, network, and file visibility.

4. Linux auditd provides detailed syscall and file-related telemetry when appropriate audit rules are configured.

5. Missing telemetry does not prove that an activity did not occur.

6. Analysts must understand telemetry coverage and blind spots before drawing conclusions from an investigation.

7. Controlled activity is valuable for validating that monitoring controls actually produce the expected evidence.

8. Telemetry recovery should be validated by generating new activity after restoration rather than assuming that restarting a service or restoring a configuration solved the problem.

---

## Evidence

Supporting screenshots are stored in:

`evidence/`

Evidence includes:

- Native Linux SSH telemetry
- Native Windows Security events
- Windows workstation unlock auditing
- Windows process auditing
- Windows service installation telemetry
- Windows file auditing
- Linux sudo activity
- Linux service activity
- Sysmon process telemetry
- Sysmon network telemetry
- Sysmon file telemetry
- Linux auditd telemetry
- Controlled telemetry failure
- Visibility-gap validation
- Telemetry restoration

---

## Documentation

Additional project documentation:

- [`build-notes.md`](build-notes.md) — implementation notes and observations
- [`validation.md`](validation.md) — validation results
- [`evidence/`](evidence/) — screenshots and supporting evidence

---

## Project Status

**PASS — Endpoint telemetry, visibility-gap testing, and telemetry restoration successfully validated.**
