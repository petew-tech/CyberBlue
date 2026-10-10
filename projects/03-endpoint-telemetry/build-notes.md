# Project 03 — Endpoint Telemetry Build Notes

## Status

Implementation and validation completed.

Windows and Linux endpoint telemetry were tested,
including native logging, Sysmon, auditd,
controlled visibility gaps, and recovery validation.

## Evidence

Endpoint telemetry validation was completed using
controlled Windows and Linux activities.

Evidence captured during the exercises includes
native operating-system logging, Sysmon, auditd,
controlled telemetry blind spots, and recovery testing.

The GitHub evidence inventory has been reconciled.

Fourteen evidence entries are marked PASS, with supporting
screenshots linked in the evidence index.

Evidence item 14 is marked Partial evidence because the
original October 5, 2026 Sysmon Event ID 11 is no longer
retained in the current Windows event log.

This limitation is documented in validation.md and the
evidence index. No historical evidence was recreated.

## Environment

- Windows 11 endpoint: `DESKTOP-JLS9SOB`
- Ubuntu Server: `cbunbuntu01`
- Sysmon: 15.22
- Linux auditd: 4.1.2

## Objectives

- Establish native Windows telemetry.
- Establish native Linux telemetry.
- Add enhanced Windows telemetry with Sysmon.
- Add enhanced Linux telemetry with auditd.
- Validate process, network, file, authentication, service, and administrative activity.
- Demonstrate a controlled telemetry blind spot.
- Restore telemetry and verify recovery.
- Preserve evidence and document limitations.


