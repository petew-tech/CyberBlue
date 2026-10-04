# CyberBlue Module 03 — Virtual Networking & Segmentation

## Project Overview

This project builds a segmented virtual networking environment on TrueNAS SCALE 24.10.1 using KVM/QEMU and libvirt.

The lab demonstrates how virtual machines connect through virtual network interfaces, Linux bridges, DHCP, NAT, routing, and isolated network segments. It also demonstrates how to validate permitted and prohibited communication paths and troubleshoot a deliberately introduced network failure.

## Lab Environment

| Component | Role |
|---|---|
| TrueNAS SCALE 24.10.1 | Virtualization host |
| Ubuntu Server 26.04.1 | CyberBlue / SOC management server |
| Linux Mint 22.3 | Lab endpoint |
| Kali Linux 2026.2 | Security testing workstation |
| `enp4s0` | TrueNAS physical network interface |
| `cyberblue-lab` | NAT-enabled CyberBlue network |
| `cyberblue-isolated` | Isolated CyberBlue network |

## Network Architecture

| Network | Subnet | Gateway | Bridge | Forwarding |
|---|---|---|---|---|
| Home / Management LAN | 192.168.1.0/24 | 192.168.1.1 | Physical LAN | External LAN |
| cyberblue-lab | 10.10.30.0/24 | 10.10.30.1 | virbr30 | NAT |
| cyberblue-isolated | 10.10.40.0/24 | 10.10.40.1 | virbr40 | None |

### Current Lab Placement

- **Ubuntu Server**
  - Management NIC: `192.168.1.243/24`
  - CyberBlue NIC: `10.10.30.158/24`
  - Connected to `cyberblue-lab`

- **Linux Mint**
  - Management NIC: `192.168.1.92/24`
  - CyberBlue NIC: `10.10.40.183/24`
  - Connected to `cyberblue-isolated`

> **Implementation note:** The `cyberblue-lab` and `cyberblue-isolated` libvirt network definitions are persistent. The additional CyberBlue VM NICs used during this lab were attached with `virsh --live` and are therefore temporary across VM restarts.
> 
## Architecture Diagram

```text
                         Home / Management LAN
                              192.168.1.0/24
                                     |
                              TrueNAS SCALE
                             192.168.1.90
                                     |
                 +-------------------+-------------------+
                 |                                       |
          cyberblue-lab                         cyberblue-isolated
          10.10.30.0/24                           10.10.40.0/24
          Gateway: 10.10.30.1                     Gateway: 10.10.40.1
          Bridge: virbr30                         Bridge: virbr40
          NAT forwarding                          No forwarding
                 |                                       |
                 |                                       |
        Ubuntu Server 26.04.1                    Linux Mint 22.3
           10.10.30.158                            10.10.40.183
```

The lab uses two separate virtual security zones. `cyberblue-lab` provides a NAT-enabled network for CyberBlue systems, while `cyberblue-isolated` provides a network with no forwarding configured.

During validation, systems attached to the same CyberBlue network successfully communicated with each other. Traffic explicitly sourced from the isolated `10.10.40.0/24` network toward the `10.10.30.0/24` lab network did not receive a response.

Both virtual machines retained their original management/LAN interfaces during testing so the lab networks could be modified and troubleshot without disrupting normal administrative access.

## TrueNAS VM Networking Discovery

During the initial build, a second NIC was added to Linux Mint through the TrueNAS VM interface and attached to `virbr30`. The guest detected the new NIC, but DHCP failed and no `10.10.30.0/24` address was assigned.

Inspection of the VM's libvirt configuration revealed that TrueNAS created the interface as a **direct/macvtap** attachment:

```xml
<interface type='direct'>
  <source dev='virbr30' mode='bridge'/>
  <model type='virtio'/>
</interface>
```

The `cyberblue-lab` DHCP service was running, but the guest did not obtain a lease. Additional inspection showed `virbr30` remained without an active conventional bridge port.

### Root Cause

Selecting `virbr30` as **NIC to Attach** in TrueNAS did not create a native libvirt network attachment. TrueNAS generated a `direct` macvtap interface using `virbr30` as its source.

For this lab, the required connection was a native libvirt network interface associated with the `cyberblue-lab` network.

### Working Configuration

A temporary interface was attached directly through libvirt:

```bash
sudo virsh -c 'qemu+unix:///system?socket=/run/truenas_libvirt/libvirt-sock' \
  attach-interface \
  --domain 7_Mint \
  --type network \
  --source cyberblue-lab \
  --model virtio \
  --live
```

The resulting interface was:

```text
Type:    network
Source:  cyberblue-lab
Device:  vnet0
```

Linux Mint then successfully received a DHCP lease from the CyberBlue network:

```text
10.10.30.195/24
```

The guest successfully reached the virtual gateway at `10.10.30.1`.

### Validation

Server-side DHCP validation confirmed that the address was issued by `cyberblue-lab`:

```text
MAC                  IP Address       Hostname
52:54:00:93:1f:8e    10.10.30.195/24 pete-mint
```

This troubleshooting exercise demonstrated the difference between:

- a **direct/macvtap VM interface**
- a **libvirt network interface**
- a Linux virtual bridge
- a libvirt virtual network
- guest DHCP configuration

It also reinforced the importance of validating networking at multiple layers rather than assuming that an interface being visible inside a VM means the complete Layer 2 and Layer 3 path is operational.

> **Persistence note:** The working CyberBlue VM interfaces were attached using `--live`. This avoided modifying the persistent VM XML managed by TrueNAS, but the additional interfaces must be recreated after a VM restart.

## Segmentation Validation

After validating the NAT-enabled `cyberblue-lab` network, a second virtual network was created to demonstrate segmentation.

### Isolated Network

The isolated network was configured as:

| Setting | Value |
|---|---|
| Network | `cyberblue-isolated` |
| Subnet | `10.10.40.0/24` |
| Gateway | `10.10.40.1` |
| Bridge | `virbr40` |
| DHCP Range | `10.10.40.100-10.10.40.200` |
| Forwarding | None |

Unlike `cyberblue-lab`, the isolated network definition contains no `<forward>` element.

Linux Mint was moved onto the isolated network and received an address in the `10.10.40.0/24` subnet. The guest successfully reached its local virtual gateway:

```text
Mint 10.10.40.x  --->  10.10.40.1
                       SUCCESS
```

This confirmed that the guest NIC, virtual bridge, DHCP configuration, and local network path were operational.

### Denied Cross-Zone Path

Ubuntu remained connected to `cyberblue-lab` at:

```text
10.10.30.158/24
```

A test from Mint explicitly sourced from its isolated-network address toward Ubuntu's CyberBlue address received no replies.

```text
cyberblue-isolated                    cyberblue-lab

Mint                                  Ubuntu
10.10.40.x       --- X --->           10.10.30.158
```

The failed traffic test was evaluated together with the virtual-network definitions. `cyberblue-lab` was configured with NAT forwarding, while `cyberblue-isolated` had no forwarding configuration.

This distinction is important because a failed ping by itself does not prove network isolation. Routing, source-interface selection, host firewalls, and other factors can also cause ICMP failure.

### Routing Validation

The TrueNAS host contained connected routes for both CyberBlue networks:

```text
10.10.30.0/24 dev virbr30 src 10.10.30.1
10.10.40.0/24 dev virbr40 src 10.10.40.1
```

The host's normal default route remained on the physical management network through `enp4s0`.

## Deliberate Network Failure and Recovery

A controlled failure was introduced to demonstrate troubleshooting and recovery.

The live `cyberblue-isolated` interface was deliberately detached from the Linux Mint VM. After removal, the guest no longer displayed its isolated-network interface or `10.10.40.x` address.

Inspection from TrueNAS confirmed that the VM no longer had a libvirt `network` interface connected to `cyberblue-isolated`.

### Root Cause

The failure was caused by removal of the VM's virtual NIC connecting Linux Mint to the isolated CyberBlue network.

### Recovery

The interface was reattached using a native libvirt network connection to:

```text
cyberblue-isolated
```

During recovery, two isolated interfaces were temporarily present. The duplicate interface was identified by its MAC address and removed.

The final Mint configuration contained one isolated interface:

```text
ens8  UP  10.10.40.183/24
```

TrueNAS showed the corresponding attachment as:

```text
vnet4  network  cyberblue-isolated  virtio
```

Connectivity to the isolated gateway was then retested:

```text
10.10.40.183  --->  10.10.40.1
                  SUCCESS
```

The successful gateway test confirmed restoration of the intended network path.

## Lessons Learned

This lab demonstrated that successful network troubleshooting requires validating each layer of the path rather than relying on a single connectivity test.

Key lessons included:

- Distinguishing physical, macvtap, bridge, and native libvirt network interfaces.
- Verifying DHCP leases rather than assuming an interface received an address.
- Checking guest addressing and routing before interpreting connectivity failures.
- Comparing virtual-network definitions when validating segmentation.
- Using source-specific testing when working with multihomed systems.
- Introducing a controlled fault, identifying the root cause, restoring the configuration, and validating recovery.
- Documenting implementation limitations instead of hiding them.

Because the lab VMs retained their original management-LAN interfaces, `cyberblue-isolated` should be understood as an isolated **virtual network segment**, not as complete isolation of the entire VM from the home/management LAN.
