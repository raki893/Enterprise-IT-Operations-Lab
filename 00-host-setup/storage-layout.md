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

The HDD is divided into two ZFS partitions.

The first ZFS partition participates in the mirrored **rpool**, while the second partition forms the standalone **tank** pool.

This layout allows the operating system and VM storage to benefit from mirrored redundancy while providing a separate storage pool for additional data.

---

## 2.3 Storage Hierarchy

```
                    Physical Storage
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
  NVMe SSD                              Seagate HDD
  476.9 GB                               931.5 GB
        │                                     │
        │                                     │
        ├──────────────┐               ┌──────┘
        │              │               │
        │         sda3 (475 GB)        │
        │              │               │
        └────── Mirror ────────────────┘
                       │
                    rpool
                       │
        ├─────────────────────────────────┐
        │                                 │
   Proxmox OS                       VM Storage
   Root Filesystem                  local-zfs
   ZFS Datasets

                       │

                 sda4 (455 GB)
                       │
                     tank
                       │
      ISO Images
      Backups
      Templates
      Additional Lab Storage
```

---

# 3. ZFS Pool Architecture

The current Proxmox installation contains two independent ZFS pools.

---

## 3.1 rpool

The **rpool** serves as the primary storage pool for the Proxmox host.

### Pool Information

| Property | Value |
|----------|-------|
| Pool Name | rpool |
| Size | 944 GB |
| Allocated | 24.6 GB |
| Free | 919 GB |
| Capacity | 2% |
| Fragmentation | 1% |
| Deduplication | 1.00x |
| Health | ONLINE |

The pool stores:

- Proxmox operating system
- Root filesystem
- VM virtual disks
- VM snapshots
- ZFS datasets
- System metadata

Current datasets include:

```
rpool
├── ROOT
│   └── pve-1
├── data
│   ├── vm-100-disk-0
│   ├── vm-100-state-P0-Baseline-Configured
│   ├── vm-101-disk-0
│   ├── vm-101-disk-1
│   ├── vm-101-disk-4
│   ├── vm-102-disk-0
│   └── vm-102-disk-1
└── var-lib-vz
```

---

### Current Pool Status

```
NAME    SIZE  ALLOC  FREE  FRAG CAP HEALTH
rpool   944G 24.6G 919G   1% 2% ONLINE
```

During validation the pool reported checksum/data errors.

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

The design follows a two-tier storage model:

- **Tank (Hot Tier):** High-performance NVMe storage used for active VM workloads.
- **Archive (Cold Tier):** Large-capacity HDD storage used for backups and long-term retention.

Design goals:

- Separate active workloads from backup data.
- Allow independent VM restoration.
- Support incremental ZFS send/receive operations.
- Maintain predictable storage growth.
- Enable future migration to RAID-1 and HA storage.

---

## 4.1 Tank (Hot Tier) Dataset Organization

The `tank` pool contains all active Proxmox VM disks and system metadata.

Storage characteristics:

| Property | Value |
|---|---|
| Pool | `tank` |
| Device | 1TB NVMe |
| Compression | LZ4 |
| Workload | Active VM disks |
| Priority | Performance |
| Snapshot frequency | Daily |
| Recordsize | 16K (VM optimized) |

Dataset layout:

```text
tank/                                      # Root dataset (LZ4 compression)
│
├── vm-disks/                              # VM disk storage container
│   │
│   ├── vm-100-disk-1/                     # pfSense VM
│   │   ├── disk.qcow2                     # 40GB virtual disk
│   │   ├── @P0-baseline                   # Initial reference snapshot
│   │   └── @YYYY-MM-DD-daily              # Daily snapshots
│   │
│   ├── vm-101-disk-1/                     # Domain Controller (DC01)
│   │   ├── disk.qcow2                     # 50GB virtual disk
│   │   ├── @P0-baseline
│   │   └── @YYYY-MM-DD-daily
│   │
│   ├── vm-102-disk-1/                     # Windows Client (CLIENT01)
│   │   ├── disk.qcow2                     # 60GB virtual disk
│   │   ├── @P0-baseline
│   │   └── @YYYY-MM-DD-daily
│   │
│   ├── vm-103-disk-1/                     # Ubuntu Server (UBU01)
│   │   ├── disk.qcow2                     # 80GB virtual disk
│   │   ├── @P0-baseline
│   │   └── @YYYY-MM-DD-daily
│   │
│   └── vm-104-disk-1/                     # Wazuh SIEM LXC
│       ├── disk.qcow2                     # 30GB virtual disk
│       ├── @P0-baseline
│       └── @YYYY-MM-DD-daily
│
├── system-meta/                           # Proxmox metadata
│   ├── cluster-config
│   └── vm-configs
│
└── reserved/                              # Future expansion area
```
## 4.2 Archive (Cold Tier) Dataset Organization

```
archive/                                   # Root dataset
│
├── vm-backup/                             # VM backup repository
│   │
│   ├── dc01/                              # Domain Controller backups
│   │   ├── @YYYY-MM-DD-daily
│   │   ├── @YYYY-MM-DD-weekly
│   │   └── retention-managed snapshots
│   │
│   ├── client01/                          # Client VM backups
│   │   └── daily + weekly snapshots
│   │
│   ├── ubu01/                             # Ubuntu server backups
│   │   └── daily + weekly snapshots
│   │
│   ├── pfsense/                           # Firewall backups
│   │   └── daily + weekly snapshots
│   │
│   └── wazuh/                             # SIEM backups
│       └── daily + weekly snapshots
│
├── metadata-backup/
│   ├── pve-nodes/
│   └── cluster-config/
│
└── reserved/
    ├── archive-vault/
    └── future-offsite-replication/
```



**END OF STORAGE LAYOUT DOCUMENT**
