# CyberBlue — TrueNAS virtualization foundation

## Objective
Build, validate, and recover one Ubuntu VM using TrueNAS-managed virtualization.

## Principle and Architecture

The CyberBlue virtualization foundation uses TrueNAS SCALE 24.10.1
as the virtualization host. TrueNAS provides the compute, memory,
storage, and virtual networking resources required by the lab virtual
machines.

The foundation guest is Ubuntu Server 26.04.1 LTS. The VM is allocated
2 vCPUs, 4 GiB of RAM, and a 50 GiB virtual disk stored on the TrueNAS
Pool1 storage pool.

The Ubuntu VM uses a VirtIO network adapter connected through the
TrueNAS physical interface `enp4s0`. Inside Ubuntu, the adapter appears
as `ens3` and uses the lab LAN for management connectivity.

Verified network path:

TrueNAS `enp4s0`
→ QEMU TAP interface
→ VirtIO virtual NIC
→ Ubuntu `ens3`
→ Lab LAN

The administrative workstation is used to access the TrueNAS web
interface and remotely manage the Ubuntu Server VM using SSH.

## Environment
- TrueNAS version:ElectricEel-24.10.1
- Host CPU / installed RAM:i5-3330 / 32G
- Guest OS and version: Ubuntu Server 26.04.1 LTS (Resolute Raccoon)
- Guest allocation: 2 vCPUs / 4 GiB RAM / 50 GiB disk
- Network attachment: VirtIO NIC attached through TrueNAS `enp4s0`
- Guest interface: `ens3`
- Guest IPv4 address: `192.168.1.243/24`
- Default gateway: `192.168.1.1`
- Management method: SSH from the administrative workstation
- Design choice: Direct LAN connectivity was used for the virtualization
  foundation so VM deployment, networking, SSH access, restart behavior,
  and recovery could be validated before implementing network segmentation.

## Results
- Build: PASS
- Network and SSH: PASS
- Shutdown and restart: PASS
- Snapshot recovery: PASS
- NAS performance check: PASS

