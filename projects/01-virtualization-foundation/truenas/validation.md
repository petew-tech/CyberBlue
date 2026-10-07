# CyberBlue — TrueNAS Virtualization Validation

## Hardware validation

### Test

Verify that hardware-assisted virtualization is available to the host operating system.

### Commands

```bash
lscpu | grep -i virtualization
egrep -c '(vmx|svm)' /proc/cpuinfo
```

### Results

```text
Virtualization: VT-x
8
```
### Conclusion

**PASS**

## Network and SSH Validation

### Test

Validate network connectivity, DNS resolution, routing, and remote SSH administration of the Ubuntu Server foundation VM.

### Guest Network Configuration

The Ubuntu Server management interface was verified as:

```text
Interface: ens3
IPv4 address: 192.168.1.243/24
Default gateway: 192.168.1.1
```

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

### Routing Validation

The guest routing path to an external address was verified with:

```bash
ip route get 8.8.8.8
```

The result confirmed that external traffic used:

```text
Gateway: 192.168.1.1
Interface: ens3
Source: 192.168.1.243
```

### DNS Validation

DNS resolution was tested with:

```bash
getent hosts github.com
```

The hostname successfully resolved to an IP address.

### External Connectivity

External IP connectivity was tested with:

```bash
ping -c 4 8.8.8.8
```

Result:

```text
4 packets transmitted
4 packets received
0% packet loss
```

### SSH Service Validation

The SSH service was verified with:

```bash
systemctl is-active ssh
```

Result:

```text
active
```

TCP port 22 was also verified as listening:

```bash
ss -tlnp | grep ':22'
```

The SSH daemon was listening on IPv4 and IPv6.

### Remote Administration Test

A successful SSH connection was established from the administrative Windows workstation to:

```text
cyberblue@192.168.1.243
```

The Ubuntu Server shell was successfully reached and remote administration was confirmed.

### Result

**PASS — routing, DNS resolution, external connectivity, SSH service availability, and remote administration were successfully validated.**

## Shutdown and Restart Validation

### Test

Validate that the Ubuntu Server foundation VM can perform a controlled reboot, return to normal operation, and restore remote management access without failed system services.

### Pre-Reboot Validation

The current Linux boot ID was recorded before restarting the VM:

```text
0c0f8686-4bc3-4b2c-a4e2-528527c5f1f0
```

### Controlled Restart

The Ubuntu Server VM was restarted using:

```bash
sudo reboot
```

The SSH session disconnected as expected while the guest restarted.

### Post-Reboot Validation

After the VM returned to service, SSH connectivity was successfully re-established.

The new Linux boot ID was:

```text
5e1dd1bd-e9a9-4a46-bdaa-70f0da0c8463
```

The change in boot ID confirmed that a new boot cycle had occurred.

System service health was checked with:

```bash
systemctl --failed
```

Result:

```text
0 loaded units listed
```

The VM successfully returned to the network and accepted a new SSH connection from the administrative workstation.

### Result

**PASS — the Ubuntu Server VM completed a controlled restart, returned to network service, restored SSH administration, and reported no failed systemd units.**

## Storage Health and Recovery Validation

### Test

Validate the health of the TrueNAS `Pool1` RAIDZ1 storage pool and
investigate a degraded pool member.

### Initial Finding

During validation, `Pool1` reported a `DEGRADED` state because one RAIDZ1
member had been marked with `too many errors`.

ZFS reported:

- READ errors: 0
- WRITE errors: 0
- CHECKSUM errors: 0
- Known data errors: None
- Previous resilver: 22.5 GiB completed with 0 errors

### Investigation

The affected pool member was identified using its persistent PARTUUID
rather than relying on `/dev/sdX` device names.

SMART diagnostics showed:

- SMART short self-test completed without error
- No reallocated sectors
- No pending sectors
- No offline uncorrectable sectors
- Historical UDMA CRC error count: 289487

The UDMA CRC counter was monitored over time and did not increase.

Kernel logs showed the affected SATA link negotiating normally at
6.0 Gbps without current I/O errors, failed commands, or link resets.

### Remediation

The TrueNAS host was shut down cleanly and the SATA connection for the
affected drive was serviced.

After restart, the affected disk changed Linux device names from
`/dev/sdb` to `/dev/sdc`. The disk was correctly identified using its
persistent PARTUUID rather than relying on the temporary `/dev/sdX`
assignment.

The historical UDMA CRC counter remained unchanged at `289487` after
the hardware maintenance.

After validating the device and connection, the stored ZFS fault state
was cleared.

### Final Validation

`Pool1` returned to:

- Pool state: ONLINE
- RAIDZ1 state: ONLINE
- All four members: ONLINE
- READ errors: 0
- WRITE errors: 0
- CHECKSUM errors: 0
- Known data errors: None

The UDMA CRC counter remained unchanged after recovery.

### Result

**PASS — storage pool returned to ONLINE with no known data errors.**

The historical CRC count will continue to be monitored, and drive
temperature/airflow will remain part of routine health checks.

## NAS I/O Validation

### Test

Validate basic read/write operation of the TrueNAS `Pool1` storage after storage-path remediation and recovery.

A temporary 1 GiB test file was created on the `Pool1/VM` dataset.

### Write Test

The following command was used to perform a 1 GiB sequential write test:

```bash
sudo dd if=/dev/zero of=/mnt/Pool1/VM/cyberblue-perf-test.bin bs=1M count=1024 conv=fdatasync status=progress
```

Result:

```text
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.656506 s, 1.6 GB/s
```

**Write Test: PASS**

### Read Test

The temporary file was then read back using:

```bash
sudo dd if=/mnt/Pool1/VM/cyberblue-perf-test.bin of=/dev/null bs=1M status=progress
```

Result:

```text
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.151447 s, 7.1 GB/s
```

**Read Test: PASS**

> Note: The reported throughput values may be influenced by ZFS and system caching. They are not presented as sustained physical-disk benchmark results. The purpose of this test was to validate successful storage I/O and confirm pool stability.

### Cleanup

The temporary test file was removed after testing:

```bash
sudo rm /mnt/Pool1/VM/cyberblue-perf-test.bin
```

### Post-Test Pool Validation

After the read/write test, `zpool status Pool1` confirmed:

- Pool1: ONLINE
- RAIDZ1: ONLINE
- All four pool members: ONLINE
- READ errors: 0
- WRITE errors: 0
- CHECKSUM errors: 0
- Known data errors: None

### Result

**PASS — storage read/write operations completed successfully and Pool1 remained ONLINE with no ZFS errors.**
