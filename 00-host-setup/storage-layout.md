# Phase 0 — Storage Layout & ZFS Architecture

**Document Version:** 1.2  
**Last Updated:** 2026-07-22  
**Maintained By:** Infrastructure Architect  
**Scope:** Proxmox VE 9.1 single-node lab storage  
**Target Audience:** Lab operator, future project phases, and portfolio reviewers

---

## 1. Executive Summary

Phase 0 establishes the storage foundation of the lab using Proxmox VE 9.1 and OpenZFS. The environment uses two ZFS pools:

- rpool — primary mirrored pool for the Proxmox OS, VM disks, and core datasets
- tank — secondary pool for extra storage, templates, ISO images, and future lab resources

The design keeps the operating system and active VM workloads on the primary pool while preserving a separate storage tier for auxiliary lab data.

---

## 2. Hardware and Partition Layout

### Primary device

- Device: SK hynix BC711 NVMe SSD
- Capacity: 476.9 GB
- Path: /dev/nvme0n1
- Role: primary storage for the Proxmox host and rpool member

```text
/dev/nvme0n1
├── nvme0n1p1    BIOS boot
├── nvme0n1p2    EFI system partition
└── nvme0n1p3    ZFS member for rpool
```

### Secondary device

- Device: Seagate ST1000VM002-1ET162
- Capacity: 931.5 GB
- Path: /dev/sda
- Role: second rpool mirror member and tank member

```text
/dev/sda
├── sda1    BIOS boot
├── sda2    EFI system partition
├── sda3    ZFS member for rpool mirror
└── sda4    ZFS member for tank
```

---

## 3. Pool Summary

| Pool | Type | Backing Devices | Purpose | Current State |
|------|------|-----------------|---------|---------------|
| rpool | Mirrored primary pool | /dev/nvme0n1p3 and /dev/sda3 | Proxmox OS, VM disks, snapshots | Needs integrity verification |
| tank | Secondary pool | /dev/sda4 | Templates, ISO images, overflow storage | Healthy |

### rpool

The rpool pool is the primary storage tier for the Proxmox host and the VM workload area. The current layout includes datasets such as:

```text
rpool
├── ROOT
│   └── pve-1
├── data
└── var-lib-vz
```

### tank

The tank pool is a secondary storage tier mounted at /tank. It is intended for ISO images, templates, additional datasets, and future backup storage use.

---

## 4. Proxmox Storage Backends

| Storage | Type | Purpose |
|----------|------|---------|
| local | Directory | ISO images, templates, backups |
| local-zfs | ZFS pool | VM disks |
| tank | Directory | Additional storage |

---

## 5. VM Inventory

| VM ID | Name | Status | Memory | Notes |
|------|------|--------|--------|-------|
| 100 | pfSense | Running | 2 GB | Firewall and router |
| 101 | DC01 | Stopped | 6 GB | Windows Server domain controller |
| 102 | Windows10 | Stopped | 6 GB | Windows client |

---

## 6. Integrity and Operational Notes

The current documentation should be treated as a live record of the host. Before proceeding to the next phase, validate the health of rpool by running:

```bash
zpool status -v rpool
zpool scrub rpool
zfs list -t snapshot rpool
```

If errors appear, document the result and resolve them before relying on the pool for additional workloads.

---

## 7. Quick Validation Commands

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
zpool status
zpool list
zfs list
pvesm status
qm list
```

