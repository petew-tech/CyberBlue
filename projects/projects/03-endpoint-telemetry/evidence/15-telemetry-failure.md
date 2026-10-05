# Checkpoint 15 — Telemetry Failure

## Controlled Sysmon Telemetry Failure

A temporary Sysmon configuration was applied that suppressed File Create telemetry (Event ID 11).

### Configuration Change

Sysmon accepted the temporary configuration.

Observed result: Configuration file validated. Configuration updated.

The Sysmon service remained operational during the test.

### Controlled Activity

The following file was successfully created while File Create telemetry was suppressed:

`C:\CyberBlue\sysmon-blind-test.txt`

### Expected Telemetry

Under normal operation, the file creation would generate Sysmon Event ID 11 — File Create.

### Observed Result

A targeted Event ID 11 query for the test filename returned:

`PASS: No Sysmon Event ID 11 recorded for sysmon-blind-test.txt during the blind test`

### Impact

The endpoint activity occurred, but the expected telemetry was unavailable.

This demonstrates that:

> A functioning endpoint does not guarantee complete security telemetry.

### Recovery

The known-good Sysmon configuration was restored and subsequently verified.

A new test file generated a valid Event ID 11 event after restoration.

**Result: PASS**
