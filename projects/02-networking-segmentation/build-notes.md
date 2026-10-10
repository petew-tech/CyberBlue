# Module 03 — Build Notes

## Environment

- **Hypervisor:** TrueNAS SCALE 24.10.1
- **Virtualization:** KVM/QEMU with libvirt
- **Physical NIC:** `enp4s0`
- **TrueNAS Management IP:** `192.168.1.90/24`
- **Management Gateway:** `192.168.1.1`

### Virtual Machines

| VM | Purpose | Management Network |
|---|---|---|
| Ubuntu Server 26.04.1 | CyberBlue / SOC management | `192.168.1.243/24` |
| Linux Mint 22.3 | Lab endpoint | `192.168.1.92/24` |
| Kali Linux 2026.2 | Security testing workstation | `192.168.1.91/24` |

## Phase 1 — Baseline

The existing TrueNAS networking configuration was documented before making changes.

The host used a single physical interface:

```text
enp4s0  192.168.1.90/24
```

The default route was:

```text
default via 192.168.1.1 dev enp4s0
```

No TrueNAS-managed VLAN or bridge configuration existed on the physical interface.

Because the system had only one physical NIC, the production/management interface was intentionally left unchanged during the lab.

## Phase 2 — Existing Libvirt Network

The existing libvirt `default` network was inspected.

It used:

```text
Network: 192.168.122.0/24
Gateway: 192.168.122.1
Bridge: virbr0
Forwarding: NAT
```

The network was temporarily started for validation and then returned to its previous inactive state.

## Phase 3 — CyberBlue Lab Network

A dedicated virtual network was created:

```text
Name: cyberblue-lab
Subnet: 10.10.30.0/24
Gateway: 10.10.30.1
Bridge: virbr30
DHCP: 10.10.30.100 - 10.10.30.200
Forwarding: NAT
```

The network was defined persistently and started successfully.

TrueNAS then contained the connected route:

```text
10.10.30.0/24 dev virbr30 src 10.10.30.1
```

## Phase 4 — VM Attachment Troubleshooting

An attempt was made to connect Linux Mint to `virbr30` through the TrueNAS VM interface.

Although the new NIC appeared in the VM configuration, DHCP did not succeed.

Inspection showed that TrueNAS had created a direct/macvtap interface rather than a native libvirt network interface.

The unsuccessful test NIC was removed.

A native libvirt network interface was then attached using:

```bash
sudo virsh -c 'qemu+unix:///system?socket=/run/truenas_libvirt/libvirt-sock' \
  attach-interface \
  --domain 7_Mint \
  --type network \
  --source cyberblue-lab \
  --model virtio \
  --live
```

Linux Mint received:

```text
10.10.30.195/24
```

DHCP lease information on TrueNAS confirmed the assignment.

Ubuntu was subsequently attached to `cyberblue-lab` and received:

```text
10.10.30.158/24
```

Bidirectional ICMP testing between Ubuntu and Mint succeeded.

## Phase 5 — Isolated Network

A second network was created:

```text
Name: cyberblue-isolated
Subnet: 10.10.40.0/24
Gateway: 10.10.40.1
Bridge: virbr40
DHCP: 10.10.40.100 - 10.10.40.200
Forwarding: None
```

The network definition intentionally omitted a `<forward>` element.

Linux Mint was moved from `cyberblue-lab` to `cyberblue-isolated`.

The guest successfully obtained an isolated-network address and reached:

```text
10.10.40.1
```

## Phase 6 — Segmentation Testing

Ubuntu remained on:

```text
cyberblue-lab
10.10.30.158/24
```

Linux Mint was placed on:

```text
cyberblue-isolated
10.10.40.0/24
```

Because Mint retained its management NIC, its normal routing table was inspected before interpreting connectivity results.

A cross-zone test was then performed using Mint's isolated-network source address toward Ubuntu's `10.10.30.158` address.

No ICMP replies were received.

The result was evaluated together with routing information and the libvirt network definitions rather than treating failed ICMP alone as proof of isolation.

## Phase 7 — Controlled Failure

A deliberate fault was introduced by removing Mint's live `cyberblue-isolated` interface.

After the fault:

- The isolated guest interface disappeared.
- The `10.10.40.x` address was lost.
- TrueNAS no longer showed the expected isolated libvirt interface.

The root cause was identified as the missing VM network attachment.

## Phase 8 — Recovery

The `cyberblue-isolated` interface was reattached.

During recovery, two isolated interfaces were temporarily present. The duplicate was identified by MAC address and removed.

The final Mint lab interface became:

```text
vnet4 → cyberblue-isolated
ens8  → 10.10.40.183/24
```

Connectivity to `10.10.40.1` was successfully restored.

Ubuntu's final lab attachment was:

```text
vnet2 → cyberblue-lab
ens8  → 10.10.30.158/24
```

## Historical Final State — Initial Libvirt Configuration

**Historical context:** This section records the final state of the original libvirt-managed `virbr30` and `virbr40` implementation. It was subsequently replaced by persistent TrueNAS-managed `br30` and `br40` bridges with static lab IP addresses. The current configuration is documented in the Network Segmentation — Implementation and Validation section below. In the original libvirt implementation, the additional CyberBlue guest NICs were live-only and had to be recreated following a VM restart. This limitation was resolved by migrating to persistent TrueNAS-managed VM network interfaces.

```text
cyberblue-lab
  Bridge: virbr30
  Gateway: 10.10.30.1/24
  State: Active
  Persistent: Yes
  Forwarding: NAT
  Ubuntu: 10.10.30.158

cyberblue-isolated
  Bridge: virbr40
  Gateway: 10.10.40.1/24
  State: Active
  Persistent: Yes
  Forwarding: None
  Mint: 10.10.40.183
```

Both CyberBlue network definitions are persistent but have autostart disabled.

The additional CyberBlue guest NICs are live-only and must be recreated following a VM restart.

## Kali BlueSOC Network Integration and Troubleshooting

Kali Linux (`BlueSOC`) required additional troubleshooting when adding a second interface to the `cyberblue-lab` network.

### Initial Issue

Kali originally used:

- `eth0` — `192.168.1.91/24`
- NetworkManager connection — `Wired connection 1`
- Default gateway — `192.168.1.1`

A live native libvirt NIC was attached to `cyberblue-lab`. During initial testing, activating the additional interface resulted in loss of Kali's management connectivity. Inspection from the local Kali TTY showed that `eth0` had become disconnected in NetworkManager.

The original management connection was restored with:

`sudo nmcli connection up "Wired connection 1"`

This restored `192.168.1.91/24` and the default route through `192.168.1.1`.

### Safer Configuration

Before retrying the CyberBlue attachment, a dedicated NetworkManager profile was created for the second interface:

`cyberblue-lab`

The profile was bound to `eth1`, configured for DHCP, and set with:

`ipv4.never-default yes`

This ensured the CyberBlue interface would not install a competing default route.

The native libvirt NIC was then attached live to the `cyberblue-lab` network.

### Final Result

Kali automatically configured:

- `eth0` — `192.168.1.91/24`
- `eth1` — `10.10.30.127/24`

Routing validation showed:

- Default route remained through `eth0`
- `10.10.30.0/24` was directly connected through `eth1`

Kali successfully reached:

- `10.10.30.1` — CyberBlue gateway
- `10.10.30.158` — Ubuntu CyberBlue host

Both tests returned 4/4 replies with 0% packet loss.

### Troubleshooting Lesson

The earlier management-connectivity failure was observed as the original `eth0` NetworkManager connection becoming disconnected. A competing default route was not established as the confirmed root cause.

For the successful configuration, the second interface was given its own NetworkManager profile with `ipv4.never-default yes`, preserving the intended management/default-route design.

Duplicate live libvirt NICs encountered during troubleshooting were also removed before final validation.

No persistent TrueNAS-owned VM XML was manually modified.

## Safety Decisions

Several design choices were intentional:

- The TrueNAS physical management interface was not reconfigured.
- The host IP was not moved to a new physical bridge.
- No VLAN configuration was introduced without VLAN-capable network infrastructure.
- The existing management NICs were retained to reduce the risk of losing administrative access.
- Persistent TrueNAS-managed VM XML was not manually edited.
- The isolated network was kept separate from the normal management LAN.
