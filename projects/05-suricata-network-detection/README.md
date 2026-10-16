# Project 05 — Suricata Network Detection & Wazuh Integration

## Overview

This project validates network intrusion-detection telemetry from Suricata on Linux Mint through the Wazuh agent to the centralized Wazuh SIEM. The project includes a custom ICMP detection rule, centralized alert verification, and a controlled Suricata process-failure/recovery test.

## Lab Components

| Component | Configuration / role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization host |
| Linux Mint 22.3 | Suricata sensor and Wazuh agent |
| Suricata | Version 7.0.3 |
| Suricata capture interface | `ens4`, `10.10.40.10/24` |
| Linux Mint management interface | `ens3`, `192.168.1.92/24` |
| Isolated lab bridge | TrueNAS `br40`, `10.10.40.1/24` |
| Wazuh manager | `siem-01`, `192.168.1.213` |
| Wazuh agent | `cyberblue-suricata`, agent ID `003` |
| Custom Suricata rule | SID `1000001` |
| Wazuh Suricata alert rule | Rule ID `86601`, level 3 |

## Implementation Summary

- Configured Suricata to capture on Linux Mint's isolated-network interface, `ens4`.
- Corrected the capture-interface configuration from the unavailable `eth0` name to `ens4`.
- Loaded the Suricata rule set and added a local ICMP echo-request detection rule for traffic from `10.10.40.10` to `10.10.40.1`.
- Confirmed Suricata generated EVE JSON alert records for custom SID `1000001`.
- Installed and configured the Wazuh agent on Linux Mint and confirmed connection to the manager at `192.168.1.213:1514/tcp`.
- Configured Wazuh log collection for `/var/log/suricata/eve.json`.
- Verified the Suricata alert in Wazuh alerts and the Threat Hunting dashboard.
- Tested automatic Suricata recovery by terminating the main process with SIGKILL. Systemd restarted the service and Suricata resumed generating alerts.
- Inspected TrueNAS `br40` and recorded the remaining network-visibility limitation: traffic delivered to the sensor is validated, but visibility into traffic between other VMs on the bridge is not established.

## Validation Summary

| Test | Observed result | Status |
|---|---|---|
| Suricata service startup configuration | `enabled` | PASS |
| Suricata service state | `active` | PASS |
| Detection rule loading after recovery | 53,144 rules loaded; 0 failed | PASS |
| ICMP test connectivity | 4 sent, 4 received, 0% loss | PASS |
| Custom Suricata detection | SID `1000001` generated an ICMP alert | PASS |
| Wazuh agent connectivity | Connected to `192.168.1.213:1514/tcp` | PASS |
| Wazuh centralized alert | Rule `86601`, agent `003`, source `/var/log/suricata/eve.json` | PASS |
| Automatic service recovery | Main PID changed from `3502` to `13422`; restart count became `1` | PASS |
| Detection after recovery | Fresh SID `1000001` alert reached Wazuh at approximately 09:29 CDT | PASS |
| Capture statistics after recovery | 278 packets, 0 kernel drops, 0 capture errors in sampled stats event | PASS |
| Full `br40` traffic visibility | Not demonstrated; deferred follow-up | OPEN |

## Scope and Limitations

The validated pipeline is: test traffic delivered to Linux Mint `ens4` → Suricata custom rule → EVE JSON → Wazuh agent → Wazuh manager and dashboard.

The TrueNAS bridge inspection showed `br40` UP at `10.10.40.1/24`, with one attached host-side virtual interface (`vnet4`) during the check. No ingress `tc` filters were reported on `vnet4` or `br40`. These observations do not prove that no other mirroring mechanism exists, but they do mean that mirroring and visibility into traffic between other VMs have not been demonstrated. A separate follow-up will assess the appropriate method for broader bridge visibility.

The capture statistics are a point-in-time sample, not proof of zero packet loss under all traffic loads. Promiscuous mode on the guest interface does not by itself guarantee visibility into traffic not delivered to that interface.

## Related Documentation

- [Build notes](build-notes.md)
- [Validation record](validation.md)
- [Project 04 — Centralized Logging & SIEM Foundations](../04-centralized-logging/README.md)
- [Project 02 — Virtual Networking & Segmentation](../02-networking-segmentation/README.md)
