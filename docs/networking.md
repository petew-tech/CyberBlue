# CyberBlue — Network Architecture and Segmentation

## Purpose

This shared networking reference combines the historical Project 02 baseline with **newer staged migration results from Project 05 Module 06 (2026-10-18)**. The earlier Project 02 addressing, commands, and diagram are retained as historical evidence, not as a live network inventory. See [Project 05 Module 06 validation](../projects/05-suricata-network-detection/validation.md#module-06--staged-network-migration-validation-2026-10-18).

## Network Inventory

| Network | Subnet | TrueNAS interface | Host bridge address | Purpose |
|---|---|---|---|---|
| Home / Management LAN | `192.168.1.0/24` | `enp4s0` | TrueNAS management: `192.168.1.90` | Host and guest administration |
| SOC bridge — Ubuntu and Mint connected | `10.10.20.0/24` | `br20` | `10.10.20.1/24` | Ubuntu `10.10.20.10/24` and Linux Mint `10.10.20.11/24`; VM-to-VM ping verified 2026-10-17 |
| Victim bridge | `10.10.30.0/24` | `br30` | `10.10.30.1/24` | Windows `10.10.30.10/24` validated; Mint passive `ens5`; Ubuntu NIC present without IPv4 |
| Attack bridge | `10.10.40.0/24` | `br40` | `10.10.40.1/24` | Kali `10.10.40.20/24` validated after reboot; Mint Suricata capture `10.10.40.10` previously validated |

The home/management LAN uses gateway `192.168.1.1`. The existing `br30` and `br40` lab bridges are documented as persistent TrueNAS-managed Linux bridges, not 802.1Q VLANs. The new `br20` has been saved and registered in TrueNAS; its post-reboot persistence remains to be tested.



### SOC Bridge Creation — 2026-10-16

- Created TrueNAS-managed interface `br20` through Network → Interfaces as type **Bridge**, without any physical bridge members. DHCP and IPv6 autoconfiguration were disabled.
- Configured IPv4 alias `10.10.20.1/24`. An initial unconfirmed Test Changes operation rolled back; the bridge was recreated, saved, and applied through TrueNAS.
- Verified on TrueNAS with `ip -4 addr show br20`: interface flags `NO-CARRIER,BROADCAST,MULTICAST,UP`, operational `state DOWN`, IPv4 `10.10.20.1/24`. No carrier is expected with no VM ports attached.
- Verified management-plane registration through `midclt call interface.query`: `name: br20`, `type: BRIDGE`, IPv4 alias `10.10.20.1`, netmask 24.
- **Status:** Bridge creation and Ubuntu ↔ Mint VM-to-VM SOC connectivity verified; reboot persistence and inter-zone security controls **not yet tested**. Existing `br30` and `br40` assignments remain in place pending staged migration. The permanent three-zone target and future VM roles are documented in [architecture.md](architecture.md#13-approved-target-architecture--soc--victim--attack-segmentation-2026-10-16).

### Ubuntu SOC NIC and Three-Network Connectivity — 2026-10-16

- Renamed TrueNAS VM ID `8` from `cbunbubtu01` to `cbubuntu01`; Ubuntu's internal hostname was corrected from `cbunbuntu01` to `cbubuntu01` after network validation.
- Preserved Ubuntu `mgmt0` (TrueNAS NIC device 24, `enp4s0`) and `lab0` (device 41, `br30`, `10.10.30.10/24`).
- Added VirtIO NIC device `52` on `br20`, MAC `00:a0:98:64:1e:3c`; Ubuntu detected this as `ens4`.
- Configured Ubuntu `ens4` with Netplan file `/etc/netplan/10-cyberblue-soc.yaml` using MAC match, address `10.10.20.10/24`, disabled DHCP, no additional default route. `netplan generate` and `netplan try --timeout 180` completed successfully. Guest `ip -4 addr show ens4` confirmed `10.10.20.10/24` and link UP.
- **Validation — all 4/4 replies with 0% loss:** `ping -I ens4 -c 4 10.10.20.1` (SOC); `ping -I mgmt0 -c 4 192.168.1.1` (management); `ping -I lab0 -c 4 10.10.30.1` (existing lab).
- **Limitation at the 2026-10-16 checkpoint:** These were guest-to-gateway checks, not proof of inter-VM communication or full network isolation. Ubuntu still had `10.10.30.10` at that time; it was released during Module 06 on 2026-10-18. TrueNAS host reboot persistence and remaining VM migrations are still open.
- **Hostname correction (2026-10-16):** TrueNAS VM display name is `cbubuntu01` (VM ID `8`). Ubuntu's system hostname originally remained `cbunbuntu01`. Ran `sudo hostnamectl set-hostname cbubuntu01`, verified `hostnamectl --static` returned `cbubuntu01`, changed `/etc/hosts` mapping from `127.0.1.1 cbunbuntu01` to `127.0.1.1 cbubuntu01` using `sudo sed -i.bak 's/cbunbuntu01/cbubuntu01/' /etc/hosts` (backup `/etc/hosts.bak`), then verified `/etc/hosts` contents. Existing session's prompt still showed old name, expected until a fresh shell/login; no restart required.


### Linux Mint SOC Attachment and Validation — 2026-10-17

- TrueNAS Mint VM ID `7` retained management NIC device `20` (`enp4s0`), Suricata monitoring NIC `43` (`br40`), and monitoring NIC `51` (`br30`). Added fourth VirtIO NIC device `53` to `br20`, MAC `00:a0:98:7e:12:f8`; guest interface `ens6`.
- Linux Mint's NetworkManager connection `Wired connection 4` for `ens6` was changed from automatic addressing to `ipv4.method manual`, `ipv4.addresses 10.10.20.11/24`, no IPv4 gateway, `ipv4.never-default yes`, and `ipv6.method disabled`. Activation succeeded; `ip -4 addr show ens6` showed `10.10.20.11/24`, link UP.
- **Connectivity (2026-10-17):** `ping -I ens6 -c 4 10.10.20.1` (TrueNAS) = 4/4, 0% loss; `ping -I ens6 -c 4 10.10.20.10` (Ubuntu) = 4/4, 0% loss. This confirms **guest-to-guest SOC network communication**.
- After restarting Mint, Suricata and Wazuh agent both returned `active`; Wazuh agent log showed connection to `192.168.1.213:1514/tcp`. Suricata 7.0.3 initialized two capture threads on both `ens4` and `ens5`, loaded 53,144 rules with no failures, and logged fresh `ens5` flow and capture telemetry. A 4-ping `ens4` test to `10.10.40.1` passed, Suricata generated four SID `1000001` EVE alerts, and `siem-01` Wazuh manager received matching level-3 rule `86601` alerts from agent `003`. Manager ingestion was validated at this stage; the Wazuh dashboard was subsequently verified on 2026-10-17 (see [Project 05 validation](../projects/05-suricata-network-detection/validation.md#threat-hunting-dashboard-confirmation--2026-10-17)).
- Removed the temporary `vnet5` `tc clsact` mirroring proof-of-concept filters before stopping Mint; TrueNAS `tc qdisc show dev vnet5` returned only `noqueue`. Earlier mirroring evidence is historical, **not a live packet-mirroring deployment**. After NIC changes, virtual port IDs may change.
- After the Mint restart, `ens3` management connectivity to gateway `192.168.1.1` and `ens4` connectivity to `10.10.40.1` were both verified with four successful pings and 0% loss. Inter-zone isolation and TrueNAS `br20` reboot persistence remain open.

## Historical Project 02 Virtual Machine Addressing (superseded)

| VM | Management IPv4 | Lab IPv4 | Lab bridge |
|---|---|---|---|
| Ubuntu Server 26.04.1 | `192.168.1.245/24` | `10.10.30.10/24` | `br30` |
| Kali Linux 2026.2 | `192.168.1.91/24` | `10.10.30.20/24` | `br30` |
| Linux Mint 22.3 | `192.168.1.92/24` | `10.10.40.10/24` | `br40` |
| Windows 11 | Management network (IP not documented in Project 02 inventory) | None assigned | None |

These values describe the documented Project 02 configuration; they are not a fresh live scan of the guests.

## Historical Project 02 Logical Topology (superseded)

```text
                     HOME / MANAGEMENT LAN
                         192.168.1.0/24
                          GW 192.168.1.1
                                |
                       TrueNAS SCALE 24.10.1
                        enp4s0 / management
                                |
                +---------------+---------------+
                |                               |
              br30                            br40
         10.10.30.1/24                   10.10.40.1/24
          SECURITY LAB                    ISOLATED LAB
                |                               |
          +-----+-----+                         |
          |           |                         |
        Ubuntu       Kali                    Linux Mint
       .30.10       .30.20                    .40.10

         br30 <-- routed IPv4 DROP --> br40
```

Ubuntu, Kali, and Mint also have **separate management-LAN NICs**, which are not shown as individual links above.

## Security Boundary and Firewall

The TrueNAS host uses a persistent firewall script:

`/mnt/Pool1/CyberBlue/cyberblue-firewall.sh`

Its two relevant IPv4 forwarding rules are:

```bash
iptables -I FORWARD 1 -i br30 -o br40 -j DROP
iptables -I FORWARD 1 -i br40 -o br30 -j DROP
```

The script removes existing copies of these rules before inserting them to avoid duplicates. A TrueNAS **POSTINIT** task runs the script at startup.

**Scope:** The rules block routed **IPv4 forwarding between the two lab bridges**. They do **not** establish full guest isolation: the Linux VMs also share the management LAN, and IPv6 isolation has not been validated.

## Validation Recorded in Project 02

The October 10, 2026 Project 02 record reports:

| Check | Recorded result |
|---|---|
| `br30` and `br40` restored after reboot | PASS |
| Ubuntu ↔ Kali security-lab connectivity | PASS |
| Ubuntu → Mint routed IPv4 blocked | PASS |
| Mint → Ubuntu routed IPv4 blocked | PASS |
| DROP counters incremented | PASS |
| Temporary diagnostic routes removed | PASS |
| Mint management address, gateway, DNS, and PuTTY SSH | PASS |

Four dropped packets in each direction were observed during the controlled cross-bridge tests.

A separate Docker compatibility review found the current Docker forwarding chains did not bypass the CyberBlue rules. The record identifies revalidation after future Docker restarts or TrueNAS updates as follow-up work.

## Historical Implementation and Troubleshooting

The initial experiment used libvirt-managed `virbr30` and `virbr40` networks, DHCP, and guest interfaces attached using live libvirt commands.

- Selecting a `virbr` interface in TrueNAS initially resulted in a **direct/macvtap** attachment, not the expected native libvirt network attachment.
- A native libvirt `--live` attachment restored connectivity, but the extra guest NIC was **not persistent** across VM restarts.
- A controlled Linux Mint NIC-detachment failure was diagnosed and recovered; a duplicate interface encountered during recovery was removed.
- The implementation was subsequently replaced by **persistent TrueNAS-managed `br30` and `br40` bridges** and static lab addresses.

These historical `virbr` addresses and DHCP leases are **not** the current network inventory.

### Linux Mint Management Address Correction

Linux Mint temporarily held both `192.168.1.92/24` (manual) and `192.168.1.212/24` (DHCP) on its management interface. Its NetworkManager profile was corrected to manual IPv4 configuration:

- Interface: `ens3`
- Address: `192.168.1.92/24`
- Gateway: `192.168.1.1`
- DNS: `1.1.1.1`

The duplicate DHCP address was removed; gateway connectivity, DNS, and SSH were subsequently verified.

## Open Follow-Up Items

- Recheck firewall ordering after Docker or application-subsystem restarts and future TrueNAS updates.
- Test IPv6 separately if introduced.
- Evaluate management-LAN restrictions if stronger VM isolation is required.
- Confirm automatic boot into the selected TrueNAS 24.10.1 boot environment during a future planned restart; activation was corrected but that later boot verification remained pending in the Project 02 record.

## Related Documentation

- [Lab Architecture](architecture.md)
- [Project 02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)
- [Project 02 Build Notes](../projects/02-networking-segmentation/build-notes.md)
- [Project 02 Validation](../projects/02-networking-segmentation/validation.md)

**Documentation status:** Project 02 implementation details are preserved as historical context. Project 05 Module 06 records the newer verified Ubuntu, Windows, and Kali guest network changes, with final security controls and host-reboot persistence still open.


### Windows 11 Victim Network Addition — 2026-10-18

- TrueNAS WIN11 VM `11`: retained management VirtIO NIC device `36` (`enp4s0`, MAC `00:a0:98:0a:82:2b`) and added second VirtIO NIC device `54` (`br30`, MAC `00:a0:98:31:78:bd`); verified both via `midclt call vm.device.query`.
- Windows adapter identification by MAC: management `Ethernet 3`, interface `19`, `192.168.1.151` with gateway/DNS `192.168.1.1`; new Victim `Ethernet`, interface `3`. New NIC's DHCP attempt yielded `169.254.133.82` APIPA; no DHCP service is configured on `br30`.
- TrueNAS ping to proposed `10.10.30.11` received no responses (not definitive absence of conflict); `arping` unavailable. On Windows Administrator PowerShell added static `10.10.30.11/24` on `Ethernet`, **without** default gateway. Follow-up `Get-NetIPAddress` reported state `Preferred`.
- **Validation PASS (each 4/4, zero packet loss):** Windows Victim `10.10.30.11` to TrueNAS `10.10.30.1` and to Ubuntu `10.10.30.10` over `br30`; Windows management `192.168.1.151` to `192.168.1.1` using `ping -S`. Management connectivity preserved.
- **Historical interim allocation:** At initial Windows onboarding, `10.10.30.11` was a temporary address while Ubuntu used `10.10.30.10`. **Superseded by Module 06 below:** Ubuntu released that IPv4 address/route and Windows migrated to `10.10.30.10` with validated guest reboot persistence. The Ubuntu `br30` NIC was not detached; shared management paths remain, and **complete inter-zone isolation is not established**.


### Module 06 — Current Verified Guest Addressing (2026-10-18)

| Guest | Management IPv4 | SOC / Victim IPv4 | Confirmed change |
|---|---|---|---|
| Ubuntu `cbubuntu01` | `mgmt0 192.168.1.243/24` | `ens4 10.10.20.10/24` on SOC `br20` | Old Victim `10.10.30.10/24` and route removed; previous `lab0` now `ens3` with no IPv4 after guest reboot |
| Windows 11 | `Ethernet 3 192.168.1.151/24` | `Ethernet 10.10.30.10/24` on Victim `br30` | Temporary `10.10.30.11` removed; new address/route survived guest reboot |
| Linux Mint | `ens3 192.168.1.92/24` (previously documented) | SOC `ens6 10.10.20.11/24`; Suricata `ens4 10.10.40.10/24`, passive `ens5` on `br30` (prior documentation) | Not re-tested during Module 06 |
| Kali | `eth1 192.168.1.91/24` | `eth0 10.10.40.20/24` on Attack `br40` | Moved from `br30`; TrueNAS NIC `42` now explicitly MAC-bound; both addresses/routes and SSH survived guest reboot |

Windows and Kali post-reboot Victim/Attack and management gateway pings each returned 4/4 replies, zero loss. Ubuntu maintained SSH after restart, with management and SOC routes present. This proves those specific guest addressing/routing behaviors—not final inter-zone firewall enforcement or full network isolation. TrueNAS `br20` **host** reboot persistence remains untested.

[Project 05 Module 06 evidence](../projects/05-suricata-network-detection/evidence/README.md#module-06--network-migration-evidence-2026-10-18) · [Validation details](../projects/05-suricata-network-detection/validation.md#module-06--staged-network-migration-validation-2026-10-18)

### Module 06 — Kali Attack NIC Migration and Reboot Validation (2026-10-18)

| TrueNAS device | Bridge | Permanent MAC | Kali guest mapping after reboot |
|---|---|---|---|
| `16` | `enp4s0` (management) | `00:a0:98:5b:5f:c8` | `eth1` / `192.168.1.91/24`, default via `192.168.1.1` |
| `42` | `br40` (Attack) | `00:a0:98:76:2d:fd` | `eth0` / `10.10.40.20/24`, no additional default gateway |

After moving device `42` from `br30` to `br40`, Kali's interface names initially changed order. NetworkManager applied the IP profiles to the wrong physical connections, resulting in loss of SSH and management gateway connectivity. The operator recovered the configuration using TrueNAS SPICE and corrected the guest profiles. TrueNAS then assigned an explicit permanent MAC to device `42`; both guest profiles were set to match their respective MACs without interface-name restrictions.

**Guest reboot verification:** Attack and management IPv4 addresses and routes persisted; PuTTY reconnected; post-reboot `10.10.40.20 → 10.10.40.1` and `192.168.1.91 → 192.168.1.1` pings each returned four replies and zero loss. No `10.10.30.0/24` connected route remained in Kali. This validates the guest migration but does not demonstrate inter-zone firewall isolation, an unrestricted traffic path, comprehensive passive IDS visibility, or TrueNAS host reboot persistence.

[Pre-reboot evidence](../projects/05-suricata-network-detection/evidence/module06-kali-attack-network-pre-reboot.png) · [Post-reboot evidence](../projects/05-suricata-network-detection/evidence/module06-kali-attack-network-post-reboot.png) · [Project 05 validation](../projects/05-suricata-network-detection/validation.md#kali-attack-zone-migration-and-recovery--2026-10-18)
