# Module 03 — Network Validation

## Validation Objective

Validate the CyberBlue virtual networking environment by confirming:

- Virtual network creation and addressing
- DHCP operation
- VM-to-VM communication
- Network segmentation
- Routing behavior
- Controlled network failure
- Troubleshooting and successful recovery

## Final Network State

| Network | Bridge | Gateway | State | Forwarding |
|---|---|---|---|---|
| `cyberblue-lab` | `virbr30` | `10.10.30.1/24` | Active | NAT |
| `cyberblue-isolated` | `virbr40` | `10.10.40.1/24` | Active | None |
| `default` | `virbr0` | `192.168.122.1/24` | Inactive | NAT |

Both CyberBlue network definitions were persistent. Autostart was disabled during the lab.

## Final VM Network Placement

### Ubuntu Server

```text
Management NIC
  Source: enp4s0
  Address: 192.168.1.243/24

CyberBlue NIC
  Interface: vnet2
  Type: network
  Source: cyberblue-lab
  Guest address: 10.10.30.158/24
```

### Linux Mint

```text
Management NIC
  Source: enp4s0
  Address: 192.168.1.92/24

CyberBlue NIC
  Interface: vnet4
  Type: network
  Source: cyberblue-isolated
  Guest address: 10.10.40.183/24
```

## Kali BlueSOC Validation

Kali Linux (`BlueSOC`) was successfully integrated into the `cyberblue-lab` network while retaining its management network connection.

### Addressing and Routing

- Management interface: `eth0` — `192.168.1.91/24`
- CyberBlue interface: `eth1` — `10.10.30.127/24`
- Default gateway: `192.168.1.1` through `eth0`
- CyberBlue route: `10.10.30.0/24` through `eth1`
- NetworkManager profile: `cyberblue-lab`
- `ipv4.never-default`: `yes`

### Connectivity Tests

| Test | Result |
|---|---|
| Kali → `10.10.30.1` CyberBlue gateway | **PASS — 4/4 replies, 0% loss** |
| Kali → Ubuntu `10.10.30.158` | **PASS — 4/4 replies, 0% loss** |
| Management/default route remains on `eth0` | **PASS** |
| Libvirt DHCP lease for Kali `10.10.30.127` | **PASS** |

### DHCP Confirmation

Libvirt DHCP reported:

- `cbunbuntu01` — `10.10.30.158/24`
- `BlueSOC` — `10.10.30.127/24`

This confirms that both Ubuntu and Kali received addresses from the `cyberblue-lab` DHCP service and can communicate across the dedicated lab segment.

### Persistence Note

The Kali NetworkManager profile persists inside the guest, but the native libvirt `cyberblue-lab` NIC attachment is live-only and does not survive a VM restart.

**Kali CyberBlue integration status: PASS**

## Test Results

| Test | Expected Result | Actual Result | Status |
|---|---|---|---|
| `virbr30` gateway available | `10.10.30.1` reachable | Reachable | PASS |
| Mint DHCP on cyberblue-lab | Receive `10.10.30.x` | `10.10.30.195/24` | PASS |
| Ubuntu DHCP on cyberblue-lab | Receive `10.10.30.x` | `10.10.30.158/24` | PASS |
| Mint → Ubuntu while both on lab network | ICMP succeeds | 4/4 replies | PASS |
| Ubuntu → Mint while both on lab network | ICMP succeeds | 4/4 replies | PASS |
| Mint isolated gateway | `10.10.40.1` reachable | 4/4 replies | PASS |
| Isolated-source → Ubuntu lab address | No usable cross-zone path | 0/4 replies | PASS |
| Deliberately remove Mint isolated NIC | Isolated interface disappears | Interface removed | PASS |
| Restore Mint isolated NIC | Address/connectivity restored | `10.10.40.183/24`; gateway reachable | PASS |

## Routing Evidence

TrueNAS contained connected routes for both CyberBlue networks:

```text
10.10.30.0/24 dev virbr30 src 10.10.30.1
10.10.40.0/24 dev virbr40 src 10.10.40.1
```

The TrueNAS management/default route remained on `enp4s0`.

## Segmentation Evidence

`cyberblue-lab` was configured with:

```xml
<forward mode='nat'/>
```

`cyberblue-isolated` contained no `<forward>` element.

A connectivity failure alone was not treated as proof of segmentation. The failed cross-zone test was evaluated together with source addressing, routing information, and the libvirt network definitions.

## Failure and Recovery Validation

A controlled fault was introduced by removing Linux Mint's live `cyberblue-isolated` NIC.

The guest lost its isolated-network interface and address. TrueNAS-side inspection confirmed that the corresponding libvirt network interface was absent.

The interface was reattached and the final configuration was verified as:

```text
vnet4   network   cyberblue-isolated   virtio
```

Linux Mint recovered:

```text
ens8   UP   10.10.40.183/24
```

A final ping to `10.10.40.1` returned 4 of 4 replies.

**Recovery status: PASS**

## Implementation Limitation

The `cyberblue-lab` and `cyberblue-isolated` network definitions are persistent, but the additional guest interfaces used for the CyberBlue networks were attached using `virsh --live`.

Therefore, these additional VM interfaces do **not** survive a VM restart in the current implementation.

This approach was intentionally used to avoid directly modifying persistent VM XML managed by TrueNAS SCALE.

The VMs also retained their original management-LAN interfaces. Therefore, the isolated network demonstrates segmentation of the CyberBlue virtual network path rather than complete isolation of the VM from every network.

## Validation Result

**MODULE 03 NETWORK VALIDATION: PASS**

The lab successfully demonstrated:

- Two distinct virtual network zones
- Separate IP addressing
- DHCP address assignment
- NAT and non-forwarding network configurations
- Allowed same-zone communication
- A prohibited/unavailable cross-zone path
- Routing inspection
- Controlled fault injection
- Root-cause identification
- Network restoration
- Post-recovery validation

## IPv6 Network Segmentation Validation

**Date:** October 10, 2026  
**Platform:** TrueNAS SCALE 24.10.1  
**Result:** PASS — IPv6 forwarding disabled

### Objective

Verify that IPv6 routing does not provide an unintended path between the CyberBlue lab networks.

### Validation Results

| Configuration | Observed value |
|---|---|
| Global IPv6 forwarding | 0 — Disabled |
| Default IPv6 forwarding | 0 — Disabled |
| br30 IPv6 forwarding | 0 — Disabled |
| br40 IPv6 forwarding | 0 — Disabled |
| enp4s0 IPv6 forwarding | 0 — Disabled |
| IPv6 FORWARD policy | ACCEPT |
| Docker IPv6 DOCKER-USER chain | RETURN |

### Bridge IPv6 Addresses

- br30: `fe80::ece9:89ff:fe41:9ce/64`
- br40: `fe80::78b0:b0ff:fe81:dc/64`

Both addresses are link-local.

### Security Assessment

IPv6 forwarding is disabled on the TrueNAS host and both CyberBlue bridges. No IPv6 routing between the lab networks is currently configured.

The IPv6 FORWARD firewall policy is ACCEPT, and no explicit IPv6 DROP rules exist between br30 and br40. This should be reviewed if IPv6 forwarding is enabled in the future.

### Outcome

**PASS:** Configuration inspection found no enabled IPv6 routing path between the CyberBlue lab bridges.

No firewall or network configuration changes were required.
