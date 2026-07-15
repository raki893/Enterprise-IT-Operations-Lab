# Enterprise IT Operations Lab

A hands-on enterprise infrastructure lab built with **Proxmox VE**, **OpenZFS**, **pfSense**, **Windows Server**, and **Windows 10**. This project simulates a small enterprise IT environment to develop practical skills in virtualization, networking, system administration, storage management, automation, and cybersecurity.

---

## Project Overview

This lab is designed as a multi-phase project where each phase focuses on a specific aspect of enterprise IT operations. The goal is to build a realistic, production-inspired environment while documenting the design, implementation, and operational procedures.

Current platform:

- **Hypervisor:** Proxmox VE 9.1.9
- **Host OS:** Debian GNU/Linux 13 (Trixie)
- **Filesystem:** OpenZFS
- **Primary Storage:** Mirrored ZFS `rpool`
- **Secondary Storage:** `tank`
- **Firewall:** pfSense
- **Directory Services:** Windows Server (Active Directory)
- **Client:** Windows 10

---

## Project Structure

```text
**Phases (Sequential):**

| Phase | Domain | Status | Target |
|-------|--------|--------|--------|
| **Phase 0** | Host Foundation | ✅ Complete 
| **Phase 1** | Network Architecture | ⏳ Planned 
| **Phase 2** | Windows Server & Active Directory | ⏳ Planned 
| **Phase 3** | Linux & Docker | ⏳ Planned | Apr 30, 2025 |
| **Phase 4** | Hybrid Cloud & Intune | ⏳ Planned 
| **Phase 5** | Monitoring & Backup | ⏳ Planned 
| **Phase 6** | Security Operations (Wazuh SIEM) | ⏳ Planned 
| **Phase 7** | ITSM Helpdesk (osTicket) | ⏳ Planned 
| **Phase 8** | Infrastructure as Code & Automation | ⏳ Planned |
```

---

## Lab Objectives

- Build and manage a virtualized enterprise environment
- Deploy and configure Active Directory
- Design secure network architecture using pfSense
- Implement ZFS-based storage management
- Configure backup and disaster recovery
- Automate administrative tasks with PowerShell and Bash
- Monitor infrastructure health and performance
- Apply enterprise IT best practices

---

## Technologies

### Virtualization

- Proxmox VE
- KVM
- QEMU
- Linux Containers (future)

### Storage

- OpenZFS
- ZFS Snapshots
- ZFS Send/Receive
- SMART Monitoring

### Networking

- pfSense
- VLANs
- DHCP
- DNS
- NAT
- Firewall Rules

### Windows Infrastructure

- Windows Server
- Active Directory Domain Services
- DNS
- Group Policy
- Windows 10 Client

### Automation

- Bash
- PowerShell

---

## Current Lab Environment

| VM ID | System | Purpose |
|--------|---------|---------|
| 100 | pfSense | Firewall and Router |
| 101 | Windows Server | Domain Controller |
| 102 | Windows 10 | Domain Client |

---

## Documentation

Each project phase contains:

- Objectives
- Architecture
- Configuration steps
- Commands executed
- Validation procedures
- Screenshots
- Lessons learned

Documentation is located in the `docs/` directory.

---

## Project Status

| Phase | Status |
|--------|--------|
| Phase 00 – Storage & ZFS | ✅ Complete |
| Phase 01 – Networking | 🚧 In Progress |
| Phase 02 – Virtualization | ⏳ Planned |
| Phase 03 – Active Directory | ⏳ Planned |
| Phase 04 – Security | ⏳ Planned |
| Phase 05 – Monitoring | ⏳ Planned |
| Phase 06 – Backup & Recovery | ⏳ Planned |

---

## Learning Goals

This lab focuses on developing practical experience in:

- Enterprise system administration
- Infrastructure design
- Network management
- Storage administration
- Virtualization
- Windows Server administration
- Cybersecurity fundamentals
- Infrastructure documentation
- Troubleshooting and operational support

---

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.