# Phase 0 — Storage Layout & ZFS Architecture

**Document Version:** 1.1  
**Last Updated:** 2026-07-14  
**Maintained By:** Infrastructure Architect  
**Scope:** Proxmox VE 9.1 Single-Node Storage Architecture  
**Target Audience:** Lab operator, future project phases, and portfolio reviewers

---

# 1. Executive Summary

Phase 0 establishes the storage foundation of the enterprise home lab using **Proxmox VE 9.1** and **OpenZFS**. The objective is to provide a resilient, high-performance storage architecture for virtualization while maintaining flexibility for future expansion.

The storage subsystem is composed of two independent ZFS pools:

- **rpool** – Primary mirrored ZFS pool hosting the Proxmox operating system, VM disks, and core datasets.
- **tank** – Secondary standalone ZFS pool used for additional storage, backups, ISO images, templates, and future lab resources.

Unlike a traditional consumer installation, this design separates production workloads from auxiliary storage while taking advantage of ZFS features including:

- End-to-end data integrity
- Copy-on-write filesystem
- Native snapshots
- Compression
- Send/Receive replication
- Online storage monitoring

The architecture is intentionally designed as the baseline for future phases of the Enterprise IT Operations Lab.

---

## Storage Metrics

| Metric | Current Value |
|---------|---------------|
| Hypervisor | Proxmox VE 9.1.9 |
| Hostname | pve |
| Operating System | Debian GNU/Linux 13 (Trixie) |
| Filesystem | OpenZFS |
| Primary Pool | rpool |
| Secondary Pool | tank |
| Number of VMs | 3 |
| Pool Health | ONLINE |

---

# 2. Physical Storage Hardware

## 2.1 Primary Storage Device

**Device**

SK hynix BC711 NVMe SSD

**Capacity**

476.9 GB

**Interface**

PCIe NVMe

**Device Path**

```
/dev/nvme0n1
```

### Partition Layout

```
nvme0n1
├── nvme0n1p1    BIOS Boot
├── nvme0n1p2    EFI System Partition
└── nvme0n1p3    ZFS Member (rpool)
```

The NVMe SSD provides high-performance storage for the Proxmox operating system and virtual machine workloads.

---

## 2.2 Secondary Storage Device

**Device**

Seagate ST1000VM002-1ET162

**Capacity**

931.5 GB

**Interface**

SATA

**Device Path**

```
/dev/sda
```

### Partition Layout

```
sda
├── sda1    BIOS Boot
├── sda2    EFI System Partition
├── sda3    ZFS Member (rpool mirror)
└── sda4    ZFS Member (tank)
```

# 4. Dataset Hierarchy

This section defines the ZFS dataset organization for the Phase 0 storage architecture.

The current layout uses two pools with distinct roles:

- `rpool` is the primary mirrored pool for the Proxmox OS, VM disks, and core datasets.
- `tank` is the secondary standalone pool for ISO images, templates, backups, and archive data.

Design goals:

- Keep the operating system and active VM storage on the mirrored primary pool.
- Separate active workloads from backup data.
- Allow independent VM restoration.
- Support incremental ZFS send/receive operations.
- Maintain predictable storage growth.

---

## 4.1 rpool (Primary Tier)

The `rpool` pool contains the Proxmox OS, root filesystem, VM disks, and system metadata.

Storage characteristics:

| Property | Value |
|---|---|
| Pool | `rpool` |
| Backing devices | `/dev/nvme0n1p3` and `/dev/sda3` |
| Role | Primary host and active VM storage |
| Health | ONLINE |

Current datasets include:

```text
rpool
├── ROOT
│   └── pve-1
├── data
└── var-lib-vz
```

---

## 4.2 tank (Secondary Tier)

The `tank` pool provides secondary storage independent of the system pool.

Storage characteristics:

| Property | Value |
|---|---|
| Pool | `tank` |
| Backing device | `/dev/sda4` |
| Role | Backups, ISO images, templates, and additional lab storage |
| Mount point | `/tank` |
| Health | ONLINE |

Current Proxmox storage backends configured in the host are:

| Storage | Type | Purpose |
|----------|------|---------|
| local | Directory | ISO images, templates, backups |
| local-zfs | ZFS Pool | VM disks |
| tank | Directory | Additional storage |

---

```
errors: 122 data errors
```

Although the pool remains ONLINE, this condition should be investigated further using:

```
zpool status -v
zpool scrub rpool
smartctl
```

to determine whether the affected files require restoration.

---

## 3.2 tank

The **tank** pool provides additional storage capacity independent of the system pool.

### Pool Information

| Property | Value |
|----------|-------|
| Pool Name | tank |
| Size | 452 GB |
| Allocated | 20.6 GB |
| Free | 431 GB |
| Capacity | 4% |
| Fragmentation | 0% |
| Deduplication | 1.00x |
| Health | ONLINE |

The tank pool is mounted at:

```
/tank
```

It is configured in Proxmox as directory storage and is intended for:

- ISO images
- VM backups
- Templates
- Additional datasets
- Future project resources

---

## 3.3 Proxmox Storage Configuration

Current storage backends configured in Proxmox are:

| Storage | Type | Purpose |
|----------|------|---------|
| local | Directory | ISO images, templates, backups |
| local-zfs | ZFS Pool | VM disks |
| tank | Directory | Additional storage |

---

## 3.4 Virtual Machine Inventory

At the completion of Phase 0 the environment contains three virtual machines.

| VM ID | Name | Status | Memory | Boot Disk |
|-------|------|--------|---------|-----------|
| 100 | pfSense | Running | 2 GB | 32 GB |
| 101 | DC01 | Stopped | 6 GB | Additional ZFS disks |
| 102 | Windows10 | Stopped | 6 GB | 40 GB |

These virtual machines represent the initial infrastructure used throughout the remaining project phases.

# 4. Dataset Hierarchy

This section defines the ZFS dataset organization for the Phase 0 storage architecture.

The current layout uses two pools with distinct roles:

- `rpool` is the primary mirrored pool for the Proxmox OS, VM disks, and core datasets.
- `tank` is the secondary standalone pool for ISO images, templates, backups, and archive data.

Design goals:

- Keep the operating system and active VM storage on the mirrored primary pool.
- Separate active workloads from backup data.
- Allow independent VM restoration.
- Support incremental ZFS send/receive operations.
- Maintain predictable storage growth.

---

## 4.1 rpool (Primary Tier)

The `rpool` pool contains the Proxmox OS, root filesystem, VM disks, and system metadata.

Storage characteristics:

| Property | Value |
|---|---|
| Pool | `rpool` |
| Backing devices | `/dev/nvme0n1p3` and `/dev/sda3` |
| Role | Primary host and active VM storage |
| Health | ONLINE |

Current datasets include:

```text
rpool
├── ROOT
│   └── pve-1
├── data
└── var-lib-vz
```

---

## 4.2 tank (Secondary Tier)

The `tank` pool provides secondary storage independent of the system pool.

Storage characteristics:

| Property | Value |
|---|---|
| Pool | `tank` |
| Backing device | `/dev/sda4` |
| Role | Backups, ISO images, templates, and additional lab storage |
| Mount point | `/tank` |
| Health | ONLINE |

Current Proxmox storage backends configured in the host are:

| Storage | Type | Purpose |
|----------|------|---------|
| local | Directory | ISO images, templates, backups |
| local-zfs | ZFS Pool | VM disks |
| tank | Directory | Additional storage |

---

**END OF STORAGE LAYOUT DOCUMENT**
