# CyberBlue — TrueNAS Virtualization Build Notes

## Project Overview

This project established the virtualization foundation for the CyberBlue homelab using TrueNAS SCALE 24.10.1.

The objective was to deploy an Ubuntu Server virtual machine, validate its compute, storage, and network configuration, verify remote administration, test restart and recovery procedures, and confirm the health of the underlying ZFS storage.

## TrueNAS Host

- Platform: TrueNAS SCALE 24.10.1
- Host hardware: ASRock H77 Pro4/MVP
- Processor: Intel Core i5-3330 @ 3.00 GHz
- Installed memory: 32 GiB
- Hardware virtualization: Intel VT-x
- Storage pool: `Pool1`
- Pool topology: RAIDZ1
- Primary network interface: `enp4s0`

## Ubuntu Foundation VM

The foundation Ubuntu Server VM was created directly on the TrueNAS virtualization platform.

Verified configuration:

- Guest OS: Ubuntu Server 26.04.1 LTS
- Virtualization: KVM/QEMU
- Virtual CPU allocation: 2 virtual CPU cores
- Memory allocation: 4 GiB
- Virtual disk: 60 GiB VirtIO disk
- Storage location: `Pool1`
- Primary virtual NIC: VirtIO
- TrueNAS attachment: `enp4s0`
- Ubuntu management interface: `ens3`
- Management IPv4 address: `192.168.1.243/24`
- Default gateway: `192.168.1.1`

## Network Validation

The primary management network path was verified as:

```text
TrueNAS enp4s0
    ↓
QEMU TAP interface
    ↓
VirtIO virtual NIC
    ↓
Ubuntu ens3
    ↓
Lab LAN
```

The Ubuntu VM successfully:

- Reached the default gateway
- Reached external IP addresses
- Resolved DNS names
- Reached GitHub through DNS resolution
- Accepted SSH connections from the administrative workstation

SSH was confirmed active and listening on TCP port 22.

Network and SSH validation: **PASS**

## Restart Validation

A controlled reboot of the Ubuntu Server VM was performed.

The system successfully restarted, generated a new Linux boot ID, returned to the network, and accepted a new SSH connection.

`systemctl --failed` reported zero failed units after the reboot.

Shutdown and restart validation: **PASS**

## Snapshot and Recovery Validation

A ZFS snapshot of the Ubuntu VM virtual disk was verified:

```text
Pool1/VM/cbunbubtu01-7x1wpm@cb-foundation-clean
```

ZFS history confirmed that the snapshot had previously been created and successfully used for a rollback operation.

Snapshot recovery validation: **PASS**

## Storage Health Investigation

During project validation, `Pool1` reported a `DEGRADED` state.

The affected RAIDZ1 member was identified using its persistent PARTUUID rather than relying only on Linux `/dev/sdX` names.

SMART diagnostics showed:

- No reallocated sectors
- No pending sectors
- No offline uncorrectable sectors
- SMART short self-test completed without error
- Historical UDMA CRC error count: `289487`

The CRC counter remained unchanged during monitoring.

Kernel logs did not show current disk I/O errors, failed commands, or SATA link resets.

The TrueNAS host was shut down cleanly and the SATA connection for the affected drive was serviced.

After reboot, Linux assigned the disk a different `/dev/sdX` name, demonstrating why persistent device identifiers are important when troubleshooting storage systems.

After verifying the hardware connection and disk health indicators, the stored ZFS fault state was cleared.

Final pool status:

- `Pool1`: ONLINE
- RAIDZ1: ONLINE
- All four members: ONLINE
- READ errors: 0
- WRITE errors: 0
- CHECKSUM errors: 0
- Known data errors: None

Storage recovery validation: **PASS**

## NAS I/O Validation

A temporary 1 GiB file was used to verify successful read/write operation on the `Pool1/VM` dataset.

Observed results:

- Sequential write: approximately 1.6 GB/s
- Sequential read: approximately 7.1 GB/s

These values may be influenced by ZFS and system caching and are not treated as sustained physical-disk benchmark results.

The temporary test file was removed after testing.

A final `zpool status` confirmed that `Pool1` remained ONLINE with zero read, write, or checksum errors.

NAS I/O validation: **PASS**

## Lessons Learned

This project reinforced several important system administration practices:

- Verify configuration using both the hypervisor and guest operating system.
- Use persistent disk identifiers when troubleshooting storage instead of relying exclusively on `/dev/sdX` assignments.
- SMART CRC errors can indicate a storage communication-path problem and should be investigated alongside media-health indicators and kernel logs.
- Validate system health after hardware maintenance before clearing stored fault conditions.
- Do not treat cache-influenced storage tests as sustained physical-disk benchmarks.
- Validate recovery procedures rather than assuming that snapshots alone provide recoverability.
- Document commands, observations, remediation, and final validation so troubleshooting work can be reproduced and explained.

## Final Result

**PASS — the CyberBlue TrueNAS virtualization foundation was successfully built, validated, recovered, and documented.**
