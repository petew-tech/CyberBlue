# CyberBlue — Troubleshooting and Recovery Journal

## Purpose

This shared journal indexes **documented** CyberBlue faults, diagnoses, corrective actions, and validation outcomes. It does not replace the detailed evidence in each project. Entries distinguish historical configurations from the current TrueNAS-managed lab.

## Troubleshooting Method

1. Define the expected behavior and record the observed symptom.
2. Identify the relevant layer: host, virtual NIC, bridge, guest IP configuration, routing, firewall, or telemetry.
3. Inspect configuration and gather evidence before making changes.
4. Change one variable or introduce a controlled fault where appropriate.
5. Re-test the original symptom and confirm recovery.
6. Document limitations and follow-up validation.

## Case 01 — VM NIC Attached to the Wrong Virtual Network Type

**Project:** [02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)  
**Context:** Historical initial libvirt network implementation; not the current persistent bridge design.

**Symptom:** A second Linux Mint NIC appeared inside the guest, but it did not receive an address from the intended `cyberblue-lab` DHCP service.

**Finding:** Selecting `virbr30` in TrueNAS produced a libvirt **direct/macvtap** interface, rather than the native libvirt `network` interface required for that experiment.

**Corrective test:** A temporary native libvirt network interface was attached to `cyberblue-lab` using a live attachment. Mint then obtained a DHCP lease and reached the virtual gateway.

**Limitation:** The live attachment did not survive VM restarts. The lab was subsequently migrated to persistent TrueNAS-managed `br30` and `br40` bridges and static lab IPs.

**Lesson:** An interface visible in the guest does not prove the intended Layer 2 or DHCP path is functioning.

## Case 02 — Deliberate Linux Mint NIC Removal and Recovery

**Project:** [02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)  
**Context:** Historical `cyberblue-isolated` libvirt test.

**Symptom:** Linux Mint lost its connection to the isolated lab after the live virtual interface was deliberately detached.

**Diagnosis:** TrueNAS/libvirt inspection showed the expected `cyberblue-isolated` network interface was absent.

**Recovery:** The interface was reattached; an extra duplicate interface encountered during recovery was identified by MAC address and removed.

**Validation:** The recovered guest interface was up with `10.10.40.183/24`, and Mint again reached `10.10.40.1`.

**Lesson:** Verify both the host-side attachment and guest-side IP configuration after restoring a VM NIC.

## Case 03 — Linux Mint Duplicate Management IPv4 Address

**Project:** [02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)

**Symptom:** Mint had two IPv4 addresses on its management NIC `ens3`: manually configured `192.168.1.92/24` and DHCP-assigned `192.168.1.212/24`.

**Correction:** Its NetworkManager management profile was switched to manual IPv4 configuration:

- Address: `192.168.1.92/24`
- Gateway: `192.168.1.1`
- DNS: `1.1.1.1`

**Validation:** The duplicate DHCP address was removed. Gateway connectivity, DNS resolution, and PuTTY SSH access were verified.

**Lesson:** Check for simultaneous DHCP and static configuration when a multihomed guest behaves unexpectedly.

## Case 04 — Cross-Bridge IPv4 Segmentation and Docker Firewall Interaction

**Project:** [02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)

**Goal:** Confirm routed IPv4 traffic between `br30` (`10.10.30.0/24`) and `br40` (`10.10.40.0/24`) is blocked while legitimate same-bridge traffic remains available.

**Implementation:** A TrueNAS POSTINIT task runs `/mnt/Pool1/CyberBlue/cyberblue-firewall.sh`, which maintains two directional iptables FORWARD DROP rules.

**Validation:** After a TrueNAS reboot, both bridges were restored, Ubuntu-to-Kali lab connectivity passed, routed traffic between Ubuntu and Mint was blocked in both directions, and the corresponding DROP counters increased by four packets in each direction.

**Docker review:** The documented Docker forwarding chains did not bypass the CyberBlue IPv4 rules in the tested configuration.

**Scope limitation:** This proves the tested **routed IPv4 lab-bridge boundary**, not IPv6 isolation or full VM isolation. The Linux VMs retain management-LAN connections.

**Follow-up:** Recheck firewall ordering after Docker restarts or TrueNAS upgrades.

## Case 05 — TrueNAS Booted the Unintended Boot Environment

**Project:** [02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)  
**Date recorded:** October 10, 2026.

**Symptom:** A TrueNAS restart selected version `24.10.2.4` rather than the intended `24.10.1` environment.

**Diagnosis:** GRUB's default entry pointed to `24.10.2.4`. Middleware showed `24.10.1` running but the other environment selected for reboot.

**Correction:** The documented supported middleware command was:

```bash
sudo midclt call bootenv.activate "24.10.1"
```

**Validation:** Middleware reported `24.10.1` as activated for running and reboot selection, and inspection showed it first in the GRUB menu.

**Outstanding:** A subsequent automatic reboot into `24.10.1` had **not** yet been performed after this correction. Do not represent that final reboot validation as complete.

**Lesson:** Prefer supported TrueNAS middleware controls to manual edits of generated GRUB files.

## Case 06 — Controlled Linux Audit Visibility Gap

**Project:** [03 — Endpoint Telemetry](../modules/module-04-endpoint-telemetry-logging/README.md)

**Test:** `auditd` was deliberately stopped, creating a period when expected audit telemetry was unavailable.

**Recovery:** The service was restarted and verified active. A new audit rule and controlled `chmod` operation produced audit telemetry.

**Lesson:** Confirm recovery by generating fresh activity, not solely by checking service status.

## Case 07 — Controlled Windows Sysmon File-Event Blind Spot

**Project:** [03 — Endpoint Telemetry](../modules/module-04-endpoint-telemetry-logging/README.md)

**Test:** Sysmon remained running, but a temporary configuration suppressed File Create telemetry. Creating `C:\CyberBlue\sysmon-blind-test.txt` produced no matching Sysmon Event ID 11 in the targeted search.

**Recovery:** The known-good configuration was restored. A second controlled file, `C:\CyberBlue\sysmon-restored-test.txt`, produced Event ID 11.

**Evidence limitation:** Project 03 records 14 PASS evidence entries and one Partial historical evidence item. The original October 5, 2026 Event ID 11 is no longer retained in the current Windows event log; the limitation is documented rather than treated as a fully retained artifact.

**Lesson:** A running telemetry agent does not guarantee that a particular event type is being collected.

## Cross-Project Lessons

- Separate the **current configuration** from historical experiments.
- Validate the whole path: virtual hardware, bridge, IP, route, firewall, and endpoint.
- Treat negative tests carefully: a failed ping alone does not prove segmentation.
- Test persistence after reboots when a change is intended to survive them.
- Restore the original monitoring configuration and generate fresh evidence.
- Preserve incomplete or pending validation explicitly.

## Source Documents

- [Project 01 — Virtualization Foundation](../projects/01-virtualization-foundation/truenas/README.md)
- [Project 02 — Networking & Segmentation](../projects/02-networking-segmentation/README.md)
- [Project 02 Build Notes](../projects/02-networking-segmentation/build-notes.md)
- [Project 02 Validation](../projects/02-networking-segmentation/validation.md)
- [Project 03 — Endpoint Telemetry](../modules/module-04-endpoint-telemetry-logging/README.md)
- [Project 03 Validation](../modules/module-04-endpoint-telemetry-logging/validation.md)

**Journal status:** Summary of previously documented work. No new host, VM, firewall, or telemetry changes were performed to create this file.
