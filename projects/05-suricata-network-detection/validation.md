# Project 05 — Validation Record

**Validation date:** 2026-10-16  
**Sensor:** Linux Mint 22.3, `cyberblue-suricata`  
**SIEM:** Wazuh manager `siem-01` (`192.168.1.213`)

## Test Results

| ID | Test | Evidence / observed result | Status |
|---|---|---|---|
| V-01 | Suricata enabled at boot | `systemctl is-enabled suricata` returned `enabled` | PASS |
| V-02 | Suricata running | `systemctl is-active suricata` returned `active` | PASS |
| V-03 | Automatic restart policy | `Restart=on-failure` | PASS |
| V-04 | Rule load after recovery | PID 13422 logged 53,144 rules loaded and 0 failed | PASS |
| V-05 | Isolated-network connectivity | Ping to `10.10.40.1`: 4 transmitted, 4 received, 0% loss | PASS |
| V-06 | Custom Suricata rule | SID `1000001` logged `CYBERBLUE TEST - ICMP Echo Request` | PASS |
| V-07 | Wazuh agent enrollment/connectivity | Agent ID `003`; log confirmed connection to `192.168.1.213:1514/tcp` | PASS |
| V-08 | EVE JSON collection | Wazuh logcollector reported analyzing `/var/log/suricata/eve.json` | PASS |
| V-09 | Centralized Wazuh detection | Wazuh rule `86601` recorded the custom Suricata signature from `/var/log/suricata/eve.json` | PASS |
| V-10 | Dashboard visibility | Operator confirmed rule `86601` visible in Threat Hunting | PASS |
| V-11 | Controlled process failure | Suricata main process PID 3502 terminated using SIGKILL | PASS — failure injected |
| V-12 | Automatic recovery | Main PID changed to 13422; `NRestarts=1`; service active/running | PASS |
| V-13 | Detection after recovery | New SID `1000001` alert observed at 09:29:00 CDT and confirmed in Wazuh | PASS |
| V-14 | Packet capture statistics | 278 kernel packets, 0 kernel drops, 0 capture errors in sampled stats event | PASS |
| V-15 | Bridge-wide visibility | No mirroring configuration was demonstrated; visibility between other VMs not verified | OPEN |

## Representative Centralized Alert

Wazuh recorded the custom detection with:

- Agent: `cyberblue-suricata` (ID `003`, address `192.168.1.92`)
- Manager: `siem-01`
- Wazuh rule: `86601`, level 3
- Suricata signature ID: `1000001`
- Source: `10.10.40.10`
- Destination: `10.10.40.1`
- Protocol: ICMP, echo request
- Location: `/var/log/suricata/eve.json`

## Conclusion

**PASS:** The tested detection path from ICMP traffic delivered to the Linux Mint sensor through Suricata EVE JSON to Wazuh alerts and the dashboard is validated. Automatic Suricata recovery was demonstrated, and detection resumed afterward.

**OPEN:** Monitoring of traffic exchanged between other virtual machines across TrueNAS bridge `br40` has not been demonstrated. This will be handled as a separate follow-up.
