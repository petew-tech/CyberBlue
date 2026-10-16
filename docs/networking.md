# CyberBlue — Network Architecture and Segmentation

## Purpose

This document is the shared networking reference for the CyberBlue lab on TrueNAS SCALE 24.10.1. It summarizes the **current documented configuration** from Project 02, its validated security boundaries, and historical troubleshooting. Detailed build commands, test evidence, and change history remain in [Project 02](../projects/02-networking-segmentation/README.md).

## Network Inventory

| Network | Subnet | TrueNAS interface | Host bridge address | Purpose |
|---|---|---|---|---|
| Home / Management LAN | `192.168.1.0/24` | `enp4s0` | TrueNAS management: `192.168.1.90` | Host and guest administration |
| SOC bridge — new, no VMs yet | `10.10.20.0/24` | `br20` | `10.10.20.1/24` | Planned SOC network, bridge created and registered on 2026-10-16 |
| Security Lab | `10.10.30.0/24` | `br30` | `10.10.30.1/24` | Ubuntu Server and Kali |
| Isolated Lab | `10.10.40.0/24` | `br40` | `10.10.40.1/24` | Linux Mint |

The home/management LAN uses gateway `192.168.1.1`. The existing `br30` and `br40` lab bridges are documented as persistent TrueNAS-managed Linux bridges, not 802.1Q VLANs. The new `br20` has been saved and registered in TrueNAS; its post-reboot persistence remains to be tested.



### SOC Bridge Creation — 2026-10-16

- Created TrueNAS-managed interface `br20` through Network → Interfaces as type **Bridge**, without any physical bridge members. DHCP and IPv6 autoconfiguration were disabled.
- Configured IPv4 alias `10.10.20.1/24`. An initial unconfirmed Test Changes operation rolled back; the bridge was recreated, saved, and applied through TrueNAS.
- Verified on TrueNAS with `ip -4 addr show br20`: interface flags `NO-CARRIER,BROADCAST,MULTICAST,UP`, operational `state DOWN`, IPv4 `10.10.20.1/24`. No carrier is expected with no VM ports attached.
- Verified management-plane registration through `midclt call interface.query`: `name: br20`, `type: BRIDGE`, IPv4 alias `10.10.20.1`, netmask 24.
- **Status:** Creation and current managed configuration verified; reboot persistence and VM connectivity are **not yet tested**. Existing `br30` and `br40` assignments remain in place pending staged migration. The permanent three-zone target and future VM roles are documented in [architecture.md](architecture.md#13-approved-target-architecture--soc--victim--attack-segmentation-2026-10-16).

## Virtual Machine Addressing

| VM | Management IPv4 | Lab IPv4 | Lab bridge |
|---|---|---|---|
| Ubuntu Server 26.04.1 | `192.168.1.245/24` | `10.10.30.10/24` | `br30` |
| Kali Linux 2026.2 | `192.168.1.91/24` | `10.10.30.20/24` | `br30` |
| Linux Mint 22.3 | `192.168.1.92/24` | `10.10.40.10/24` | `br40` |
| Windows 11 | Management network (IP not documented in Project 02 inventory) | None assigned | None |

These values describe the documented Project 02 configuration; they are not a fresh live scan of the guests.

## Logical Topology

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

**Documentation status:** This shared reference summarizes recorded Project 02 findings; it does not claim new tests or configuration changes.
