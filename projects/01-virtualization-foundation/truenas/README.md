# CyberBlue — TrueNAS virtualization foundation

## Objective
Build, validate, and recover one Ubuntu VM using TrueNAS-managed virtualization.

## Principle and architecture
[Explain host, guest, virtual resources, and the laptop’s management role in my own words.]

## Environment
- TrueNAS version:ElectricEel-24.10.1
- Host CPU / installed RAM:i5-3330 / 32G
- Guest OS and version: Ubuntu Server 26.04.1 LTS (Resolute Raccoon)
- Guest allocation: 2 vCPUs / 4 GiB RAM / 50 GiB disk
- Network attachment and why I chose it:
- Current network limitation: foundation VM on the LAN; range isolation not completed.

## Results
- Build: PASS
- Network and SSH: PASS
- Shutdown and restart: PASS
- Snapshot recovery: PASS
- NAS performance check: PASS

