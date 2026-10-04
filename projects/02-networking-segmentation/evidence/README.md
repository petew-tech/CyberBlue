
# Module 03 — Evidence

This directory contains selected screenshots documenting the build, validation, troubleshooting, and recovery of the CyberBlue virtual networking and segmentation lab.

## Evidence Index

| Checkpoint | Evidence | Filename |
|---|---|---|
| 01 | Existing TrueNAS network baseline | `01-existing-network-baseline.png` |
| 02 | Default libvirt network configuration | `02-default-network-detail.png` |
| 03 | Default-network guest validation | `03-default-network-guest-validation.png` |
| 04 | CyberBlue network addressing plan | `04-cyberblue-network-plan.png` |
| 05 | `cyberblue-lab` network created | `05-cyberblue-network-created.png` |
| 06 | Guest CyberBlue addressing | `06-mint-cyberblue-address.png` |
| 07 | VM-to-VM connectivity | `07-vm-to-vm-connectivity.png` |
| 08 | Isolated network created | `08-isolated-network-created.png` |
| 09 | Cross-zone denied/unavailable path | `09-denied-path.png` |
| 10 | Host routing comparison | `10-routing-comparison.png` |
| 11 | Deliberate network failure | `11-deliberate-network-failure.png` |
| 12 | Network restored and validated | `12-network-restored.png` |
| 13 | Final network architecture | `13-final-architecture.png` |
| 13B | Final VM-to-network mapping | `13-final-guest-network-mapping.png` |
| 14 | Final validation documentation | Documented in `../validation.md` |

## Evidence Handling

Screenshots are selected to demonstrate technical results without unnecessarily exposing sensitive information.

Before publication, screenshots should be reviewed for:

- Passwords or authentication information
- API keys or tokens
- SSH private keys
- Remote-access information
- Public IP addresses
- Personal email addresses
- Browser tabs or notifications containing private information
- Unrelated host or network information

Terminal screenshots should contain enough context to establish what was tested while avoiding unnecessary disclosure.

## Important Validation Note

A failed ping alone was not considered proof of segmentation.

The cross-zone test was evaluated together with:

- Source addressing
- Guest routing
- TrueNAS routing
- Libvirt network definitions
- The absence of forwarding on `cyberblue-isolated`

This provides stronger evidence of the intended network design than relying on ICMP failure alone.

### 15 — Kali CyberBlue Network Validation

**File:** `15-kali-cyberblue-network-validation.png`

Validates the final Kali Linux (`BlueSOC`) network configuration after joining the CyberBlue lab segment.

Evidence shown:

- `eth0` — `192.168.1.91/24` management network
- `eth1` — `10.10.30.127/24` CyberBlue lab network
- Default route remains `192.168.1.1` through `eth0`
- `10.10.30.0/24` is directly routed through `eth1`
- Confirms the CyberBlue interface does not replace the management/default route

Kali also successfully reached the CyberBlue gateway (`10.10.30.1`) and Ubuntu (`10.10.30.158`) with 0% packet loss.
