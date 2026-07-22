# Enterprise IT Operations Lab

A portfolio-ready infrastructure lab demonstrating enterprise-style operations using Proxmox VE, ZFS, pfSense, Windows Server, and Windows 10.

## Current Status

- Platform: Proxmox VE 9.1 on Debian GNU/Linux 13 (Trixie)
- Host: Gigabyte B450M S2H, Ryzen CPU, 64 GB RAM
- Storage: ZFS pools rpool (primary) and tank (secondary)
- Virtual machines: pfSense (100), DC01 (101), and Windows10 (102)

## Hardware and Storage

| Component | Detail |
|-----------|--------|
| Motherboard | Gigabyte B450M S2H |
| RAM | 64 GB |
| NVMe | SK hynix BC711, 476.9 GB |
| HDD | Seagate ST1000VM002, 931.5 GB |

| Pool | Purpose |
|------|---------|
| rpool | Primary ZFS pool for Proxmox OS and VM storage |
| tank | Secondary pool for extra storage, ISO images, and templates |

> Note: rpool should be validated before proceeding to Phase 1. Use zpool status -v rpool and zpool scrub rpool to confirm health.

## Repository Structure

- [00-host-setup](00-host-setup) — host configuration, storage notes, and Phase 0 documentation
- [01-network](01-network) — network phase materials and planning
- [00-host-setup/storage-layout.md](00-host-setup/storage-layout.md) — storage architecture
- [00-host-setup/lab-dr-runbook.md](00-host-setup/lab-dr-runbook.md) — disaster recovery procedures

## Validation Commands

```bash
zpool status -v rpool
zpool scrub rpool
qm list
pveversion
```