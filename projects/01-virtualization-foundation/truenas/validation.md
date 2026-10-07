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
