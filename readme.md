# Enterprise IT Operations Lab

A portfolio-ready home lab that demonstrates enterprise-style IT operations using Proxmox VE, OpenZFS, pfSense, Windows Server, and Windows 10.

## Current Status

- Hypervisor: Proxmox VE 9.1.9 on Debian GNU/Linux 13 (Trixie)
- Host: Gigabyte B450M S2H with Ryzen CPU and 64 GB RAM
- Storage: ZFS pools rpool and tank
- Virtual machines: pfSense (100), DC01 (101), Windows10 (102)

## Hardware and Storage

| Component | Detail |
|-----------|--------|
| Motherboard | Gigabyte B450M S2H |
| CPU | Ryzen 8-core class CPU |
| RAM | 64 GB |
| NVMe | SK hynix BC711, 476.9 GB, /dev/nvme0n1 |
| HDD | Seagate ST1000VM002-1ET162, 931.5 GB, /dev/sda |

| Pool | Role | Capacity | Notes |
|------|------|----------|-------|
| rpool | Primary ZFS pool for Proxmox OS and VM storage | ~944 GB | Mirrored across NVMe and HDD; verify data integrity before Phase 1 |
| tank | Secondary pool for extra storage and templates | ~452 GB | Used for overflow storage, ISO images, and backups |

> Important: the lab documentation now reflects the actual rpool/tank layout. If rpool reports data errors, validate them with zpool status -v rpool and zpool scrub rpool before moving to the next phase.

## Lab Structure

- [00-host-setup](00-host-setup) — host foundation, storage notes, hardening backups, and Phase 0 diagrams
- [01-network](01-network) — network phase work and related planning
- [00-host-setup/storage-layout.md](00-host-setup/storage-layout.md) — storage architecture and pool layout
- [00-host-setup/lab-dr-runbook.md](00-host-setup/lab-dr-runbook.md) — disaster recovery procedures
- [00-host-setup/storage-status.txt](00-host-setup/storage-status.txt) — current storage status snapshot

## Current Virtual Machines

| VM ID | Name | Role | Status |
|------|------|------|--------|
| 100 | pfSense | Firewall and router | Running |
| 101 | DC01 | Windows Server domain controller | Stopped |
| 102 | Windows10 | Windows client | Stopped |

## Validation Commands

Run these on the Proxmox host to confirm the current environment:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
zpool status
zpool list
zfs list
pvesm status
qm list
hostnamectl
```

## Next Steps

1. Investigate rpool data integrity and document the result.
2. Keep the storage documentation aligned with the live host configuration.
3. Proceed to Phase 1 networking once the storage baseline is verified.

