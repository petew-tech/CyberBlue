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
