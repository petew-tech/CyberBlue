# Module 07 — Screenshot Evidence Inventory

The operator confirmed these **15 filenames** in the network-share folder `\\192.168.1.70\h\BLUE SOC\Mod 07` on October 26, 2026 (US Central). **All 15 PNG files are now uploaded to this GitHub review branch, and their filenames and file presence have been independently verified.** Individual screenshot readability and pixel contents have not yet been fully reviewed.

| Evidence filename | Purpose |
|---|---|
| [03a-module07-tshark-session-reconstruction.png](03a-module07-tshark-session-reconstruction.png) | TCP/SSH session |
| [04b-module07-ssh-stream-reconstruction.png](04b-module07-ssh-stream-reconstruction.png) | SSH stream |
| [04c-module07-pcap-integrity-validation.png](04c-module07-pcap-integrity-validation.png) | PCAP integrity |
| [05b-module07-zeek-checksum-correction.png](05b-module07-zeek-checksum-correction.png) | Zeek checksum handling |
| [06a-module07-zeek-ssh-protocol-analysis.png](06a-module07-zeek-ssh-protocol-analysis.png) | Zeek SSH metadata |
| [07a-module07-reconnaissance-session-analysis.png](07a-module07-reconnaissance-session-analysis.png) | Nmap/Zeek reconnaissance |
| [08a-module07-suricata-reconnaissance-detection.png](08a-module07-suricata-reconnaissance-detection.png) | Suricata SID 1000003 |
| [09a-module07-wazuh-event-network-fields.png](09a-module07-wazuh-event-network-fields.png) | Wazuh network fields |
| [09b-module07-wazuh-rule-correlation.png](09b-module07-wazuh-rule-correlation.png) | Wazuh rule correlation |
| [10a-module07-http-request-response.png](10a-module07-http-request-response.png) | HTTP transaction |
| [10b-module07-zeek-http-file-analysis.png](10b-module07-zeek-http-file-analysis.png) | Zeek HTTP file metadata |
| [10c-module07-http-object-recovery.png](10c-module07-http-object-recovery.png) | HTTP object recovery |
| [11a-module07-zeek-packet-loss-comparison.png](11a-module07-zeek-packet-loss-comparison.png) | Complete/missing TCP comparison |
| [11b-module07-missing-file-record.png](11b-module07-missing-file-record.png) | Missing file record |
| [12a-module07-suricata-rollback-validation.png](12a-module07-suricata-rollback-validation.png) | Suricata rollback |

## Publication Checklist

- [ ] Visually review all screenshots for readability and credentials.
- [x] Upload all 15 screenshots to this `evidence/` directory.
- [x] Confirm filenames/case match the inventory.
- [x] Link uploaded filenames from this evidence index.
- [ ] Review PCAP payloads and host/IP information before deciding whether to publish captures.
- [ ] Add a Module 07 banner matching the CyberBlue/Cyber Forge visual style; do not add a broken banner link before upload.
