# Module 07 — Build Notes

**Execution:** October 26–27, 2026 (US Central time)  
**Scope:** Controlled CyberBlue lab traffic; offline packet analysis; temporary IDS validation rule.  
**Safety decision:** No TrueNAS host networking or persistent packet-mirroring changes.

## 1. Initial SSH Session

- Kali source: `192.168.1.91:47848`; Ubuntu destination: `192.168.1.243:22`.
- Capture: `02a-first-tcp-session.pcap`, 188 packets, zero kernel drops.
- SHA-256: `b8e805f886c3ea30ff0e76a94d22d5b30f34613e2e0db6cba9df0efcf333fed7`.
- TShark showed the TCP setup/termination and SSH version strings `OpenSSH_10.3p1 Debian-4` and `OpenSSH_10.2p1 Ubuntu-2ubuntu3.8`.
- Copied PCAP to Mint and `siem-01`; hashes matched.
- Installed Zeek 8.0.10 core-only on `siem-01` (Ubuntu 24.04.5). A full-package simulation would have installed Postfix, so the smaller core package was selected.
- An initial Zeek run reported invalid TCP checksums (consistent with NIC checksum offloading), and its `conn.log` yielded `OTH` entries. In an isolated analysis directory, reran:

```bash
/opt/zeek/bin/zeek -C -r ../pcaps/02a-first-tcp-session.pcap LogAscii::use_json=T
```

- Result: `conn_state:SF`; 95 originator and 93 responder packets, `missed_bytes:0`; `ssh.log` recorded `auth_success:true` (Zeek inference), `mlkem768x25519-sha256` key exchange and `chacha20-poly1305@openssh.com` cipher.

## 2. Controlled TCP Reconnaissance

On Kali:

```bash
nmap -sT -Pn -p 1-50 --max-retries 0 10.10.40.10
```

- 49 closed TCP ports; TCP/22 open.
- Capture `07a-reconnaissance.pcap`: 102 packets, zero kernel drops.
- SHA-256: `c0009c2c83ed78a80c53995af4759d3449b08ca3d37b7366f82ebb9e15f8fff5`.
- TShark detected 50 distinct SYN destination ports.
- Zeek offline output in `~/cyberblue/module07/zeek-scan`: 49 `REJ` and one `RSTO` (TCP/22), `conn.log` generated.

## 3. Suricata Temporary Detection and Wazuh Correlation

Before modification, backed up the original `/etc/suricata/rules/local.rules` to `local.rules.module07-backup`. Both initially matched SHA-256:

```text
79dd12fa1336aa97b950096b42d5cef924dea4c1226affc84671c159618c48bf
```

Temporary scoped test rule, added only after verifying SID `1000003` was unused:

```text
alert tcp 10.10.40.20 any -> 10.10.40.10 any (msg:"CYBERBLUE MODULE07 - Repeated TCP SYN Test"; flags:S; flow:stateless; threshold:type threshold, track by_src, count 10, seconds 60; classtype:attempted-recon; sid:1000003; rev:1;)
```

- `sudo suricata -T -c /etc/suricata/suricata.yaml`: configuration successfully loaded.
- `sudo suricatasc -c reload-rules`: `{"message":"done","return":"OK"}`.
- Repeated identical 50-port Nmap scan, capturing `07c-rule-validation.pcap`: 102 packets, zero kernel drops.
- Capture SHA-256: `425c010ccd5b50b441e2ed55f9f4d0b00d182526b216e5df506b16fddbffd2c6`.
- Mint EVE JSON contained five SID `1000003` alerts, `in_iface:ens4`, source `10.10.40.20`, destination `10.10.40.10`. Alert action: `allowed`.
- Wazuh Threat Hunting search `data.alert.signature_id:1000003` returned five events. Expanded event showed source/destination, `manager.name:siem-01`, log location `/var/log/suricata/eve.json`, rule `86601`, level 3.

## 4. Controlled HTTP Capture and Zeek

On Mint, verified TCP/8080 unused; started temporary HTTP server:

```bash
python3 -m http.server 8080 --bind 10.10.40.10
```

On Kali, verified connectivity with `curl -I`; captured `07d-http-session.pcap` on `eth0`, then issued HTTP GET:

```bash
curl --max-time 10 -o /dev/null -w 'HTTP Status: %{http_code}\nBytes Received: %{size_download}\n' http://10.10.40.10:8080/
```

- Observed `HTTP Status: 200`, `Bytes Received: 1756`.
- Capture: 11 packets, zero kernel drops; size 2,934 bytes.
- Capture SHA-256: `d6ba0fca99e286eb61ed474c521e99564ff2d04f0462c865f37abd054099e094`.
- Transferred to `siem-01`, verified identical SHA-256.
- Zeek in `~/cyberblue/module07/zeek-http`:

```bash
/opt/zeek/bin/zeek -C -r ../pcaps/07d-http-session.pcap LogAscii::use_json=T
```

- `http.log`: `GET /`, response `200 OK`, `response_body_len:1756`, MIME `text/html`, connection UID `CwkiiJ2fj3HVmo7jNa`, response FUID `FUX8MC3IvgZdGfOsB5`.
- `files.log`: `seen_bytes:1756`, `total_bytes:1756`, `missing_bytes:0`, `is_orig:false`.

## 5. Recover HTTP Object

On Kali:

```bash
tshark -r ~/cyberblue/module07/pcaps/07d-http-session.pcap --export-objects http,/home/pete/cyberblue/module07/extracted/
wc -c ~/cyberblue/module07/extracted/%2f
sha256sum ~/cyberblue/module07/extracted/%2f
```

- TShark saved the requested root path as `%2f`.
- Exact file length: 1,756 bytes.
- Recovered HTML begins with `<!DOCTYPE HTML>` and `Directory listing for /`.
- Object SHA-256, checked again during final inventory: `4343976dc5643d803e539f91ec782586147ce0fdf2861b01c08911e8c39cf3e8`.

## 6. Deliberately Missing Response Packet

To preserve the original evidence, created a separate capture excluding **original packet 7**:

```bash
tshark -r ~/cyberblue/module07/pcaps/07d-http-session.pcap -Y 'frame.number != 7' -w ~/cyberblue/module07/pcaps/07e-http-missing-packet.pcap
```

- Complete: 11 packets; modified: 10 packets.
- Modified PCAP SHA-256: `2f8723a1140f0f7937dfd39bf281cb684c1142b80f468c1ff5634dafc44dd242`, checked on both Kali and `siem-01`.
- Original TCP payload distribution: packet 4 request 80 bytes; packet 6 response header segment 156 bytes; packet 7 response body 1,756 bytes. Numbering changes in the 10-packet copy.
- Zeek in `~/cyberblue/module07/zeek-http-missing`: `http.log` remained, but `files.log` was absent.
- Complete `conn.log`: `conn_state:SF`, `missed_bytes:0`, 5 response packets, `resp_ip_bytes:2180`.
- Modified `conn.log`: `conn_state:S2`, `missed_bytes:1757`, 4 response packets, `resp_ip_bytes:372`. Modified `http.log` retained `GET /` and `200 OK`, but `response_body_len:0` and no file UID.
- **Interpretation:** Zeek inferred a sequence gap. The experiment removed evidence, not live application data; response-body length zero means no body bytes reconstructed from this PCAP, not that the real server transmitted an empty body.

## 7. Cleanup and Restoration

- Mint's Python HTTP server stopped via Ctrl+C; `ss` showed no TCP/8080 listener.
- Confirmed backup had original local ICMP SIDs `1000001` and `1000002`, but no SID `1000003`.
- `diff -u` showed only the added SID `1000003` line.
- Restored backup to `local.rules`; SHA-256 returned original `79dd12fa...618c48bf`.
- Suricata config test succeeded; `suricatasc -c reload-rules` returned OK; `systemctl is-active suricata` returned `active`.
- `grep -n 'sid:1000003'` against restored file returned no match.
- `suricatasc -c ruleset-stats` returned `rules_loaded:53145`, `rules_failed:0`, `rules_skipped:0`. This establishes successful reload, not a per-SID live memory listing.
- Five PCAPs and the recovered HTTP object were present and hashed again during final inventory.

## Network Monitoring Constraint

The lab's TrueNAS management is remote. This module did not attach host-side `tc` mirrors or reconfigure TrueNAS bridging. Zeek was used for **offline PCAP analysis** only. Persistent monitoring of unrelated bridge traffic remains unverified and must not be claimed in portfolio documentation.
