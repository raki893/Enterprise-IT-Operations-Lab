# Actual Host Configuration Reference

## Scope

This file captures the current Phase 0 host configuration based on the live lab state and the supporting notes in the repository.

## Host Summary

- Hypervisor: Proxmox VE 9.1.9
- OS: Debian GNU/Linux 13 (Trixie)
- Hostname: pve
- Motherboard: Gigabyte B450M S2H
- CPU: Ryzen-class 8-core CPU
- RAM: 64 GB

## Storage Overview

- Primary pool: rpool
- Secondary pool: tank
- NVMe device: /dev/nvme0n1
- HDD device: /dev/sda

## Pool Notes

- rpool is the primary pool for the host OS, VM disks, and snapshots.
- tank is the secondary pool for extra storage, templates, and lab data.
- The storage documentation should be updated to reflect this layout before Phase 1 proceeds.

## Validation Commands

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
zpool status
zpool list
zfs list
pvesm status
qm list
```
