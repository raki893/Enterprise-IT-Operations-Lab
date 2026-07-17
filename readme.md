# Enterprise IT Operations Lab

A hands-on enterprise infrastructure lab built with Proxmox VE, OpenZFS, pfSense, Windows Server, and Windows 10. The project simulates a small enterprise IT environment to build practical skills in virtualization, networking, system administration, storage management, automation, and cybersecurity.

## Project At A Glance

This lab is a multi-phase enterprise-style environment built on a single Proxmox host. The current foundation includes ZFS storage, a pfSense firewall VM, a Windows Server domain controller, and a Windows 10 client.

- Goal: learn and document enterprise IT operations in a realistic home lab.
- Current state: Phase 0 is complete; networking and later phases are planned or in progress.
- Core stack: Proxmox VE, ZFS, pfSense, Windows Server, and Windows 10.

## Project Structure

The workspace is organized around the host foundation and the next lab phase.

- 00-host-setup/ - host foundation, storage notes, hardening artifacts, and the Phase 0 diagram
- 01-network/ - network phase work and related planning
- readme.md - top-level overview and lab status

Supporting files in 00-host-setup/:

- lab-dr-runbook.md
- storage-layout.md
- storage-status.txt
- bios-audit/ - host audit outputs and screenshots
- diagrams/Phase-0.drawio - Phase 0 storage and host foundation diagram
- hardening/ - saved hardening config backups and repository files
- hardening/sshd_config.bak - SSH daemon baseline
- hardening/chrony.conf.bak - time sync baseline
- hardening/sources.list.d/ - repository definitions for Debian and Proxmox
- hardening/user-list.png and hardening/user-mfa.png - Proxmox access and MFA references

## Lab Objectives

- Build and manage a virtualized enterprise environment
- Deploy and configure Active Directory
- Design secure network architecture using pfSense
- Implement ZFS-based storage management
- Configure backup and disaster recovery
- Automate administrative tasks with PowerShell and Bash
- Monitor infrastructure health and performance
- Apply enterprise IT best practices

## Technologies

### Virtualization

- Proxmox VE
- KVM
- QEMU
- Linux Containers, planned for later phases

### Storage

- OpenZFS
- ZFS snapshots
- ZFS send/receive
- SMART monitoring

### Networking

- pfSense
- VLANs
- DHCP
- DNS
- NAT
- Firewall rules

### Windows Infrastructure

- Windows Server
- Active Directory Domain Services
- Group Policy
- Windows 10 client

### Automation

- Bash
- PowerShell

## Current Lab Environment

| VM ID | Hostname | OS / Role | Status | Purpose |
|--------|----------|-----------|--------|---------|
| 100 | pfSense | Firewall appliance | Running | Firewall and router |
| 101 | DC01 | Windows Server | Stopped | Domain controller |
| 102 | Windows10 | Windows 10 | Stopped | Domain client |

Current VM count: 3.

## Architecture Summary

| Layer | Component | Current Detail |
|-------|-----------|----------------|
| Hypervisor | Proxmox VE | 9.1.9 on Debian GNU/Linux 13 (Trixie) |
| Primary storage | ZFS rpool | Mirrored pool on /dev/nvme0n1p3 and /dev/sda3 |
| Secondary storage | ZFS tank | Separate pool on /dev/sda4 for backups, ISOs, and templates |
| Firewall | pfSense | VM 100 |
| Directory services | Windows Server AD | VM 101 (DC01) |
| Client | Windows 10 | VM 102 |

For the detailed storage design, see [storage layout notes](00-host-setup/storage-layout.md) and [storage status](00-host-setup/storage-status.txt).

## Hardware Inventory

| Device | Type | Capacity | Role |
|--------|------|----------|------|
| SK hynix BC711 NVMe | SSD | 476.9 GB | Primary boot and rpool member |
| Seagate ST1000VM002-1ET162 | HDD | 931.5 GB | rpool mirror member and tank storage |

### Storage Map

| Device / Partition | Size | Filesystem | Purpose |
|--------------------|------|------------|---------|
| /dev/nvme0n1p1 | 1007K | - | Boot partition |
| /dev/nvme0n1p2 | 1G | vfat | EFI system partition |
| /dev/nvme0n1p3 | 475G | zfs_member | rpool mirror member |
| /dev/sda1 | 1007K | - | Boot partition |
| /dev/sda2 | 1G | vfat | EFI system partition |
| /dev/sda3 | 475G | zfs_member | rpool mirror member |
| /dev/sda4 | 455.5G | zfs_member | tank pool member |

Current pool sizes:

| Pool | Size | Allocated | Free | Health |
|------|------|-----------|------|--------|
| rpool | 944G (~1.01 TB) | 24.6G | 919G | ONLINE |
| tank | 452G (~485.33 GB) | 20.6G | 431G | ONLINE |

## Phase Roadmap

| Phase | Focus | Status | Reference |
|-------|-------|--------|-----------|
| Phase 0 | Host foundation, storage, ZFS | Complete | 00-host-setup/ and diagrams/Phase-0.drawio |
| Phase 1 | Network architecture | Planned / in progress | 01-network/ |
| Phase 2 | Windows Server and Active Directory | Planned | To be added |
| Phase 3 | Linux and Docker | Planned | To be added |
| Phase 4 | Hybrid cloud and Intune | Planned | To be added |
| Phase 5 | Monitoring and backup | Planned | To be added |
| Phase 6 | Security operations | Planned | To be added |
| Phase 7 | ITSM helpdesk | Planned | To be added |
| Phase 8 | Configuration management and automation | Planned | To be added |

## How To Validate The Environment

Use these commands on the Proxmox host to confirm the current state:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
zpool status
zpool list
zfs list
pvesm status
qm list
hostnamectl
```

What to look for:

- rpool and tank should both be ONLINE
- VM IDs should match the current lab inventory
- Hostname should be pve
- ZFS datasets should match the documented layout

## Documentation

Each project phase should contain:

- Objectives
- Architecture
- Configuration steps
- Commands executed
- Validation procedures
- Screenshots
- Lessons learned

Primary supporting files for the host setup are in 00-host-setup/.

## Hardening Baselines

The hardening folder keeps the host-side configuration snapshots used during Phase 0:

- SSH daemon settings in hardening/sshd_config.bak
- Chrony time-sync settings in hardening/chrony.conf.bak
- Package repository definitions in hardening/sources.list.d/
- Proxmox access references in hardening/user-list.png and hardening/user-mfa.png

## Glossary

| Term | Meaning |
|------|---------|
| Proxmox | Bare-metal virtualization platform used for the lab host |
| rpool | Main ZFS pool for the Proxmox OS and VM storage |
| tank | Secondary ZFS pool for extra lab storage |
| pfSense | Firewall and router VM |
| DC01 | Windows Server domain controller VM |
| Windows10 | Client VM used for domain and endpoint testing |
