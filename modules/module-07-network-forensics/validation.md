# Module 07 — Validation Record

**Validation period:** October 26–27, 2026 (US Central)  
**Lab:** CyberBlue on TrueNAS SCALE 24.10.1  
**Sensors/tools:** Kali, TShark, tcpdump, Zeek 8.0.10 core, Linux Mint Suricata 7.0.3, Wazuh `siem-01`.

## Test Results

| ID | Test | Evidence / observed result | Status |
|---|---|---|---|
| V-01 | SSH endpoint capture | 188 packets captured, zero kernel drops | PASS |
| V-02 | SSH PCAP transfer integrity | SHA-256 `b8e805f8…333fed7` consistent on Kali, Mint, and SIEM | PASS |
| V-03 | Zeek SSH checksum handling | Initial checksum rejection investigated; offline `-C` yielded `SF`, `missed_bytes:0`, and `ssh.log` | PASS — scoped |
| V-04 | SSH protocol reconstruction | Client/server banners and SSH algorithm negotiation recorded | PASS |
| V-05 | TCP reconnaissance | Nmap `-sT` on ports 1–50: port 22 open; 49 closed | PASS |
| V-06 | Recon PCAP and TShark | 102 packets, zero kernel drops, 50 distinct SYN destination ports | PASS |
| V-07 | Zeek recon classification | 49 `REJ`, 1 `RSTO` on port 22 | PASS |
| V-08 | IDS test-rule safety | Original `local.rules` backed up and hash verified before addition | PASS |
| V-09 | Suricata SID `1000003` validation | Config test successful; rules hot-reloaded | PASS |
| V-10 | Suricata repeated SYN detection | Five EVE alerts for Kali `10.10.40.20` to Mint `10.10.40.10` | PASS |
| V-11 | Wazuh correlation | Five Threat Hunting hits; SID `1000003`; Wazuh rule `86601` level 3 | PASS |
| V-12 | HTTP test | `HEAD /` and `GET /` returned `200 OK`; 1,756-byte GET response | PASS |
| V-13 | HTTP PCAP integrity | 11 packets, zero kernel drops, SHA-256 `d6ba0fca…099e094` consistent | PASS |
| V-14 | Zeek HTTP transaction | `http.log`: GET `/`, 200, `response_body_len:1756` | PASS |
| V-15 | Zeek file metadata | `files.log`: MIME `text/html`, 1,756 bytes seen, zero missing | PASS |
| V-16 | HTTP object recovery | Extracted `%2f`, exactly 1,756 bytes, readable HTML; SHA-256 `4343976d…39cf3e8` | PASS |
| V-17 | Separate missing-packet capture | Removed original frame 7 from copy; 11 → 10 packets; SHA-256 `2f8723a1…44dd242` | PASS |
| V-18 | Zeek missing-data response | Modified: `S2`, `missed_bytes:1757`, no `files.log`; baseline: `SF`, zero missing, file recorded | PASS |
| V-19 | Temporary HTTP service removal | No TCP/8080 listening process after server shutdown | PASS |
| V-20 | Suricata rollback integrity | Original local-rule SHA-256 restored; only test SID had differed | PASS |
| V-21 | Suricata reload and health | `suricata -T` PASS; reload OK; service active; 53,145 loaded, zero failed/skipped | PASS |
| V-22 | Evidence inventory | Five PCAPs and extracted file rehashed; 15 screenshot filenames inventoried on operator network share | PASS — inventory |
| V-23 | Continuous Zeek / bridge-wide capture | No safe persistent capture or traffic mirror established | OPEN / BLOCKED |
| V-24 | GitHub screenshot upload | All 15 filenames verified as present in this module's `evidence/` directory; visual review remains pending | PASS — file presence |

## Key Evidence Values

| Artifact | SHA-256 |
|---|---|
| `02a-first-tcp-session.pcap` | `b8e805f886c3ea30ff0e76a94d22d5b30f34613e2e0db6cba9df0efcf333fed7` |
| `07a-reconnaissance.pcap` | `c0009c2c83ed78a80c53995af4759d3449b08ca3d37b7366f82ebb9e15f8fff5` |
| `07c-rule-validation.pcap` | `425c010ccd5b50b441e2ed55f9f4d0b00d182526b216e5df506b16fddbffd2c6` |
| `07d-http-session.pcap` | `d6ba0fca99e286eb61ed474c521e99564ff2d04f0462c865f37abd054099e094` |
| `07e-http-missing-packet.pcap` | `2f8723a1140f0f7937dfd39bf281cb684c1142b80f468c1ff5634dafc44dd242` |
| Recovered HTTP `%2f` | `4343976dc5643d803e539f91ec782586147ce0fdf2861b01c08911e8c39cf3e8` |

## Forensic Comparison — Complete vs. Missing Packet

| Field | Complete HTTP PCAP | Modified HTTP PCAP |
|---|---|---|
| Captured packets | 11 | 10 |
| HTTP request | GET `/` | GET `/` |
| Response status in Zeek | 200 OK | 200 OK |
| HTTP body observed by Zeek | 1,756 bytes | 0 |
| Zeek file metadata | `files.log` present | `files.log` absent |
| TCP state | `SF` | `S2` |
| `missed_bytes` | 0 | 1,757 |
| Responder packets | 5 | 4 |
| `resp_ip_bytes` | 2,180 | 372 |

**Interpretation:** This is a controlled capture-evidence gap. The complete recording established that the server sent the body. The modified copy does **not** establish failure, an empty real response, or attack activity.

## Scope and Outstanding Work

- **PASS:** Endpoint-captured traffic, offline protocol analysis, scoped Suricata alerting, Wazuh correlation, object recovery, evidence integrity, and test cleanup.
- **OPEN:** Persistent Zeek capture service and safe, authorized bridge-wide observation path. The existing TrueNAS network was not altered.
- **PENDING:** Upload of the 15 inventoried screenshots and selection of any PCAPs suitable for public release. Review PCAP payloads and IP/host details before publishing.
