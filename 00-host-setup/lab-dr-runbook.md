# Phase 0 — Disaster Recovery Runbook

**Document Version:** 1.2  
**Last Updated:** 2026-07-22  
**Maintained By:** Infrastructure Architect  
**Scope:** Proxmox VE 9.1 single-node lab host  
**Environment:** Home lab (non-production reference architecture)

---

## 1. Executive Summary

This runbook defines the recovery steps for the lab host and its virtual machines. The current layout uses a single Proxmox node with rpool as the primary ZFS pool and tank as the secondary pool.

### Recovery Objectives

| Metric | Target | Notes |
|--------|--------|-------|
| RTO | 4 hours | Restore VMs and rebuild the host from available backups or snapshots |
| RPO | 24 hours | Use fresh snapshots and backup copies when available |
| MTTR | 2 hours | Typical VM rollback or host rebuild |

---

## 2. Failure Scenarios

### 2.1 VM Data Corruption

**Symptoms**
- VM fails to boot or shows filesystem errors
- Guest OS reports journal or I/O issues
- Proxmox shows the VM in an error state

**Steps**
```bash
qm stop <vmid>
qm listsnapshot <vmid>
qm rollback <vmid> <snapshot>
qm start <vmid>
```

**Note**
- Use the most recent healthy snapshot available on the relevant VM disk dataset.
- If the snapshot itself appears corrupt, proceed to host recovery.

### 2.2 rpool Integrity Issue

**Symptoms**
- zpool status reports errors or degraded state
- Data corruption warnings appear for rpool

**Steps**
```bash
zpool status -v rpool
zpool scrub rpool
zpool status rpool
```

**If errors persist**
- Stop affected VMs and document the issue.
- Restore from backup or rebuild the affected dataset from a known-good snapshot.
- Avoid adding new workloads until the pool is healthy.

### 2.3 tank Pool Issue

**Symptoms**
- zpool status reports tank as degraded or offline
- Backup or template operations fail

**Steps**
```bash
zpool status tank
zpool scrub tank
```

**Impact**
- The lab can usually continue running, but template, backup, and extra storage operations may be unavailable until tank is repaired.

### 2.4 Power Failure

**Symptoms**
- Host powers off suddenly
- VMs stop without a graceful shutdown

**Steps**
```bash
# Power on the host and confirm it boots normally
qm list
qm monitor <vmid>
```

**Follow-up**
- Check the guest OS for filesystem inconsistencies and run the VM rollback procedure if needed.

### 2.5 Network Failure

**Symptoms**
- Proxmox is unreachable over the network
- VMs lose connectivity

**Steps**
```bash
ip link show
ip route show
ping <gateway>
```

**Follow-up**
- Fix cable or switch issues, then verify the host and VMs have network access again.

### 2.6 Thermal or Cooling Failure

**Symptoms**
- Host powers off due to overheating
- CPU temperature rises sharply

**Steps**
```bash
sensors
cat /proc/cpuinfo | grep temp
```

**Follow-up**
- Cool the host, inspect the cooler and fans, and verify the VMs after the system is stable.

---

## 3. Recovery Procedures

### 3.1 Restore a Single VM

```bash
qm stop <vmid>
qm rollback <vmid> <snapshot>
qm start <vmid>
```

### 3.2 Recover from rpool Problems

```bash
zpool status -v rpool
zpool scrub rpool
zfs list -t snapshot rpool
```

If the pool still reports corruption after a scrub, treat the pool as non-trusted until restored from backup or rebuilt from known-good data.

### 3.3 Recover from tank Problems

```bash
zpool status tank
zpool scrub tank
```

Rebuild or restore tank only after confirming the underlying device is healthy and the pool is not reporting I/O errors.

---

## 4. Maintenance Checklist

- Confirm the health of rpool before adding new workload data.
- Monitor SMART health for the NVMe and HDD devices.
- Keep a current external backup of any critical VM configuration files.
- Review snapshot retention on a regular basis so it remains appropriate for the lab storage size.

---

## 5. Quick Reference Commands

```bash
qm list
zpool status
zpool list
zfs list
smartctl -a /dev/nvme0n1
smartctl -a /dev/sda
```

