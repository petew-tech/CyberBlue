# Module 07 — Network Forensics & Session Investigation

## Overview

This module investigates controlled SSH, TCP reconnaissance, and HTTP sessions in the CyberBlue lab using **tcpdump, TShark, Zeek 8.0.10, Suricata 7.0.3, and Wazuh**. It validates the path from captured packets to protocol metadata, recoverable HTTP content, custom IDS alerts, and centralized SIEM evidence. A deliberately incomplete copy of an HTTP capture demonstrates how missing packets change forensic conclusions.

**Status: PASS for the scoped offline investigations and alert-correlation tests.** Continuous Zeek monitoring and bridge-wide third-party traffic visibility remain **OPEN/BLOCKED** because a safe, persistent packet-observation path has not been established. No TrueNAS host networking changes were made during this module.

## Lab Components

| Component | Configuration / role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization host; existing `br20`, `br30`, and `br40` networks preserved |
| Kali / BlueSOC | `10.10.40.20` (`eth0`) for Attack-zone test traffic; `192.168.1.91` management |
| Linux Mint 22.3 | `10.10.40.10` (`ens4`) Suricata sensor and temporary HTTP server; `192.168.1.92` management |
| Ubuntu SSH peer | `192.168.1.243:22` for controlled SSH capture |
| SIEM | `siem-01`, Ubuntu 24.04.5, `192.168.1.213`; Wazuh manager/dashboard |
| Zeek | 8.0.10 **core-only**, installed on `siem-01`; offline PCAP analysis with `/opt/zeek/bin/zeek` |
| Suricata | 7.0.3 on Linux Mint, AF_PACKET interfaces `ens4` and `ens5`; EVE at `/var/log/suricata/eve.json` |
| Wazuh | Agent `cyberblue-suricata` (003); matching alerts under rule `86601`, level 3 |
| Evidence | Five PCAP files on Kali; Zeek JSON logs on `siem-01`; 15 screenshots uploaded to the review branch (visual review pending) |

## Implementation Summary

1. Captured a complete SSH session from Kali `192.168.1.91` to Ubuntu `192.168.1.243:22`; reconstructed TCP behavior and SSH banners in TShark.
2. Copied the capture to `siem-01` and verified SHA-256. Initial Zeek analysis was affected by checksum offloading; `zeek -C` recovered a complete `SF` connection and `ssh.log`, including negotiated SSH algorithms.
3. Ran a controlled Nmap TCP connect scan of Mint `10.10.40.10` ports 1–50 from Kali `10.10.40.20`. Observed 49 refused ports and port 22 open. Offline Zeek recorded 49 `REJ` and one `RSTO` connection.
4. Backed up Mint's `local.rules`, then temporarily loaded Suricata SID `1000003` (10 SYN packets within 60 seconds, scoped to the two lab IPs). The repeated scan generated **five** matching EVE alerts and **five** Wazuh Threat Hunting results.
5. Served a temporary Python HTTP directory listing at `10.10.40.10:8080`; captured 11 packets and zero kernel drops. TShark and Zeek confirmed GET `/`, response `200 OK`, and a 1,756-byte `text/html` body.
6. Used TShark HTTP object export to recover the 1,756-byte HTML response as `%2f`; recorded its SHA-256 hash.
7. Created a **separate** 10-packet PCAP missing the original response-body packet. Zeek still recognized `GET /` and `200 OK`, but reported `missed_bytes:1757`, `conn_state:S2`, zero observed HTTP response-body bytes, and no `files.log`.
8. Stopped the temporary HTTP server. Verified the original Suricata rules backup, restored the configuration, passed `suricata -T`, reloaded the ruleset, and confirmed active service with **53,145 rules loaded, zero failed, zero skipped**.

## Validation Summary

| Test | Observed result | Status |
|---|---|---|
| SSH capture and integrity | 188 packets; checksum-corrected Zeek `SF`; `ssh.log` generated | PASS |
| Reconnaissance | 50 destination ports, 49 refused, SSH/22 open | PASS |
| Suricata → Wazuh correlation | SID `1000003`, five EVE alerts and five Wazuh hits; rule `86601` | PASS |
| HTTP capture | 11 captured; zero kernel drops; GET `/` → `200 OK` | PASS |
| HTTP file record | `seen_bytes:1756`, `total_bytes:1756`, `missing_bytes:0` | PASS |
| Extracted HTTP object | 1,756 bytes; SHA-256 recorded and rechecked | PASS |
| Controlled missing-packet copy | 11 → 10 packets; Zeek `SF` → `S2`, `missed_bytes:1757` | PASS |
| Temporary HTTP listener removal | Port 8080 no longer listening | PASS |
| Suricata rollback | Original rules hash restored; configuration and reload succeeded | PASS |
| Persistent Zeek / bridge-wide monitoring | No safe passive mirroring path established | OPEN / BLOCKED |

## Scope and Limitations

- The PCAPs were captured **at participating endpoints**. Their results do not establish passive observation of unrelated VM-to-VM traffic.
- `-C` was applied only for offline analysis after checksum-offload artifacts in the SSH capture; it disables Zeek checksum rejection and should not be interpreted as checksum validation.
- SID `1000003` detected repeated TCP SYN packets between specified lab IPs; it did **not** require distinct destination ports and did not block the scan. The EVE category string is not evidence of actual data exfiltration.
- The removed HTTP packet represents **missing evidence**, not an HTTP service failure. Zeek's 1,757 missed TCP sequence bytes include sequence accounting; it does not mean 1,757 additional application bytes were transferred.
- SID `1000003` was temporary. The original two local ICMP rules were restored; previously collected alerts remain available.
- All 15 screenshot files have been uploaded to this module's `evidence/` directory and their filenames verified. Their visual content and readability still require final review.

## Related Documentation

- [Build notes](build-notes.md)
- [Validation record](validation.md)
- [Evidence inventory and upload checklist](evidence/README.md)
- [Module 06 — Network Security Monitoring](../module-06-network-security-monitoring/README.md)

## Key Finding

**An endpoint capture can prove that a session occurred, but an incomplete capture can hide the very content needed for investigation.** In the complete HTTP PCAP, Zeek identified the file and all 1,756 response bytes. In an otherwise equivalent copy with one response-body packet removed, Zeek still knew the server replied `200 OK` but could not create a file record. This distinction is central to defensible network-forensic reporting.
