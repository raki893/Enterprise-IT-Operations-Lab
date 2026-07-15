# Phase 0 — Disaster Recovery Runbook

**Document Version:** 1.0  
**Last Updated:** 2026-07-14  
**Maintained By:** Md Rakibul Islam  
**Scope:** Ryzen 3700X Proxmox VE 9.1 Single-Node Hypervisor  
**Environment:** Home Lab (non-production reference architecture)  

---

## 1. Executive Summary

This runbook defines disaster recovery (DR) procedures for the Phase 0 home lab infrastructure. The lab runs on a single physical host (Proxmox VE 9.1 on Ryzen 3700X) with dual-tier storage: NVMe hot tier (OS, snapshots, active VMs) and HDD cold tier (backup archive, retention policy).

### Recovery Objectives

| Metric | Target | Notes |
|--------|--------|-------|
| **RTO** (Recovery Time Objective) | 4 hours | VM restoration from daily snapshots; Proxmox host rebuild from backup |
| **RPO** (Recovery Point Objective) | 24 hours | Daily incremental snapshots at 03:00 UTC; last snapshot used for recovery |
| **MTPD** (Mean Time to Detect) | 30 minutes | Manual health checks; Wazuh alerts (Phase 6+) |
| **MTTR** (Mean Time to Repair) | 2 hours | Typical VM snapshot restore or NVMe rebuild |

### Failure Categories & Frequencies

| Category | Likelihood | RTO | Impact | Priority |
|----------|------------|-----|--------|----------|
| VM corruption/data loss | High | 1 hr | Single workload down | P1 |
| NVMe failure (hot tier) | Medium | 2 hrs | All VMs offline; restore from HDD | P1 |
| HDD failure (cold tier) | Low | 24 hrs | Backup archive lost; RTO limited to available snapshots on NVMe | P2 |
| PSU failure | Low | 30 min | Complete outage; hardware replacement | P1 |
| Network issue (NIC/cable) | Medium | 15 min | Connectivity loss; failover to secondary NIC or manual reset | P2 |
| Thermal failure (cooling) | Medium | 20 min | Host shutdown; manual intervention required | P1 |

---

## 2. Disaster Types & Detection

### 2.1 VM Data Corruption

**Symptoms:**
- VM fails to boot or exhibits filesystem errors
- Application logs show I/O errors or journal replays
- Proxmox console shows guest OS panic or kernel errors
- VM disk becomes read-only

**Detection (manual):**
```bash
# SSH to Proxmox host
ssh root@proxmox-host

# List all VMs and their status
qm list

# Check disk usage and I/O errors
zpool status tank
zfs list -o name,used,available tank

# For specific VM:
qm status <vmid>
qm monitor <vmid>
```

**Detection (automated, Phase 5+):**
- Wazuh agent running on each VM; SIEM alerts on kernel panics
- Prometheus monitoring HDD I/O latency spikes
- Disk space trending alerts (>80% capacity on NVMe)

---

### 2.2 NVMe Storage Failure (Hot Tier)

**Symptoms:**
- Proxmox console shows "zpool degraded" warnings
- Error logs: `SMART errors detected on /dev/nvme0n1`
- VM read/write latency spikes to >500ms
- Proxmox Web UI: pool shows DEGRADED state
- System becomes unresponsive (all VMs stall)

**Detection (manual):**
```bash
# Check ZFS pool status
zpool status tank

# Check NVMe SMART health
smartctl -a /dev/nvme0n1

# Monitor I/O latency
iostat -x 1 5

# Tail kernel logs for hardware errors
dmesg | grep -i nvme
```

**Detection (automated):**
- Prometheus scrape of `zfs_pool_health` metric (0=healthy, 1=degraded, 2=offline)
- smartd daemon alerts on SMART threshold exceeded
- Kernel oops messages in system journal

**RTO Impact:** **2 hours** (restore all VMs from HDD cold tier snapshots)

---

### 2.3 HDD Storage Failure (Cold Tier)

**Symptoms:**
- Proxmox shows "zpool degraded" on archive pool
- Error logs: `Disk I/O error` on /dev/sda (HDD)
- SMART: reallocated sector count increasing, offline uncorrectable sectors
- Backup jobs fail with "I/O error: disk unavailable"

**Detection (manual):**
```bash
# Check archive pool status
zpool status archive

# Check HDD SMART
smartctl -a /dev/sda

# List failed backup tasks
ls -lh /var/log/backup/
grep ERROR /var/log/backup/*.log
```

**Detection (automated):**
- Wazuh alert on failed backup job
- Prometheus: `zfs_pool_health{pool="archive"}` = degraded
- smartd alerts via syslog

**RTO Impact:** **24+ hours** (VMs can run on NVMe snapshots; no new backups possible; RTO limited to oldest available NVMe snapshot)

---

### 2.4 PSU Failure

**Symptoms:**
- Host powers off unexpectedly
- No beep on boot; fans not spinning
- Voltmeter on PSU shows 0V output
- Host boots but randomly reboots under load

**Detection:**
- Manual: Physical inspection (no power LED, fan not spinning)
- Hardware monitoring (Phase 5+): `ipmi-sensors` or `lm-sensors` shows voltage rails critical

**RTO Impact:** **30 minutes** (hardware swap; Proxmox auto-starts VMs on reboot)

---

### 2.5 NIC/Network Failure

**Symptoms:**
- Proxmox lost SSH/Web UI access
- No response to ping
- Physical inspection: no link lights on NIC ports
- `ip link show` reports DOWN status
- Ethtool: no link detected

**Detection (manual):**
```bash
# Check NIC status
ip link show
ethtool eth0
ethtool eth1

# Check vmbr0/vmbr1 status
brctl show

# Check routes and gateway
ip route show
ping 192.168.1.1
```

**RTO Impact:** **15 minutes** (swap cable or use secondary NIC; no VM data loss)

---

### 2.6 Thermal/Cooling Failure

**Symptoms:**
- Host shuts down automatically (BIOS thermal protection)
- Proxmox logs show thermal warnings before shutdown
- Fan not running (physical inspection)
- Sensors report >90°C CPU temp

**Detection:**
```bash
# Check temperature sensors
sensors
cat /proc/cpuinfo | grep "temp"

# Check fan status
pwmconfig
```

**RTO Impact:** **20 minutes** (manual: unplug, wait for cool-down, inspect/replace fan)

---

## 3. Recovery Procedures by Failure Type

### 3.1 VM Data Corruption Recovery

#### Scenario: Single VM is corrupted (e.g., DC01)

**Step 1: Isolate the VM**
```bash
# Stop the affected VM immediately
qm stop 101  # Example: DC01 is VMID 101

# Prevent auto-start during troubleshooting
qm set 101 --onboot 0
```

**Step 2: Identify the most recent good snapshot**
```bash
# List all snapshots for the VM
qm listsnapshot 101

# Output format:
# ├─P0-baseline (2025-01-15 09:00)
# ├─2025-01-30-daily (2025-01-30 03:00)
# └─2025-01-31-daily (2025-01-31 03:00) ← use this one

# Check snapshot date/time metadata
zfs list -t snapshot -S creation tank/vm-101-disk-1
```

**Step 3: Restore from the most recent good snapshot**
```bash
# Revert to the snapshot (this will overwrite current disk state)
qm rollback 101 2025-01-31-daily

# Confirm the rollback
echo "Snapshot restore complete. VM will boot from 2025-01-31-daily state."
```

**Step 4: Verify and restart**
```bash
# Set auto-start back to enabled
qm set 101 --onboot 1

# Boot the VM
qm start 101

# Monitor boot logs
qm monitor 101

# Verify guest OS is responsive
ssh admin@dc01-ip "uname -a"
```

**Step 5: Validate application state**
- Check application logs for consistency
- Run integrity checks (Windows: `chkdsk /F`, Linux: `fsck -n`)
- Verify network connectivity and DNS resolution

**Impact Window:** 1 hour (detection + rollback + verification)

**Backup Plan:** If snapshot also corrupted, proceed to **Section 3.5: Full Host Recovery**

---

### 3.2 NVMe Failure (Hot Tier) Recovery

#### Scenario: NVMe develops SMART errors or fails completely

**Step 1: Confirm pool degradation**
```bash
# Check zpool status
zpool status tank

# Expected output:
#   pool: tank
#  state: DEGRADED
# status: One or more devices are faulted in response to persistent errors.

# View failing device
zpool status tank -v | grep -i faulted
```

**Step 2: Put host in maintenance mode (pause all operations)**
```bash
# Stop all running VMs (gracefully)
for vm in $(qm list | awk 'NR>1 {print $1}'); do
  echo "Stopping VM $vm..."
  qm shutdown $vm --timeout 120
done

# Wait for all VMs to stop
sleep 300
qm list  # verify all stopped

# Disable auto-start
echo "Setting maintenance mode..."
# (Later: disable scheduled backups, alerts, auto-failover)
```

**Step 3: Rebuild NVMe from HDD backup**

*Option A: If NVMe is recoverable (controller OK, only data corrupted):*
```bash
# Wipe the device
zpool offline tank nvme0n1
zpool remove tank nvme0n1

# Clear partition table and re-initialize
dd if=/dev/zero of=/dev/nvme0n1 bs=1M count=10
sgdisk -Z /dev/nvme0n1

# Recreate zpool on fresh NVMe
zpool create tank /dev/nvme0n1

# Restore all VM disks from HDD archive snapshots
# For each VM:
zfs send archive/vm-backup/dc01@2025-01-31 | zfs recv tank/vm-dc01-disk-1

# (Repeat for all VMs: CLIENT01, UBU01, pfSense, etc.)
```

*Option B: If NVMe is completely failed (requires hardware replacement):*
```bash
# Power down the host
shutdown -h now

# [MANUAL STEP] Replace failed NVMe with new drive of same capacity
# [Boot back into Proxmox BIOS; ensure new NVMe is detected]

# Once back in OS:
# Initialize new NVMe and restore
zpool create tank /dev/nvme0n1
zfs create tank/vm-disks

# Restore from HDD archive (same as Option A above)
zfs send archive/vm-backup/dc01@2025-01-31 | zfs recv tank/vm-dc01-disk-1
zfs send archive/vm-backup/client01@2025-01-31 | zfs recv tank/vm-client01-disk-1
zfs send archive/vm-backup/ubu01@2025-01-31 | zfs recv tank/vm-ubu01-disk-1
zfs send archive/vm-backup/pfsense@2025-01-31 | zfs recv tank/vm-pfsense-disk-1
```

**Step 4: Verify restored data integrity**
```bash
# Check pool status
zpool status tank

# Verify all VM disk datasets
zfs list tank/vm-*

# Check available snapshots (should show latest)
zfs list -t snapshot tank
```

**Step 5: Restart all VMs**
```bash
# Boot VMs in priority order:
# 1. pfSense (network foundation)
qm start 100  # pfSense

# Wait for pfSense network to come up
sleep 60

# 2. DC01 (Active Directory, all clients depend on this)
qm start 101  # DC01
sleep 60

# 3. Other VMs
qm start 102  # CLIENT01
qm start 103  # UBU01
qm start 104  # Wazuh

# Monitor boot and verify connectivity
qm monitor 100
```

**Step 6: Validate all services**
- SSH into each VM and verify OS is responsive
- Ping all VMs and test inter-VM connectivity
- Verify Active Directory is reachable from clients
- Check all backup jobs resume normally

**Total RTO:** ~2 hours

**Data Loss Window:** Up to 24 hours (last backup snapshot may be 24h old)

---

### 3.3 HDD Failure (Cold Tier) Recovery

#### Scenario: HDD archive pool fails; no new backups possible

**Impact:** Cannot create new backups; VMs can only rely on NVMe snapshots (24–48 hour window).

**Step 1: Confirm HDD pool failure**
```bash
zpool status archive

# Expected: DEGRADED or OFFLINE
```

**Step 2: Options**

*Option A: Rebuild archive pool (if HDD recoverable):*
```bash
# Follow same procedure as Section 3.2, but for archive pool
# 1. Offline and remove failed device
# 2. Re-initialize HDD
# 3. Recreate zpool
# 4. Restore backup datasets from secondary backup (if exists)
#    → If no secondary backup, archive pool will be empty
#    → NVMe snapshots become only RTO source
```

*Option B: Replace HDD (if failed completely):*
```bash
# Power down
shutdown -h now

# [MANUAL] Replace HDD with new drive of same/larger capacity
# [Reboot into Proxmox]

# Re-create archive pool
zpool create archive /dev/sda

# Restore backup manifests if available
# (Requires external backup of backup metadata)
```

**Step 3: Resume backup jobs**
```bash
# Once pool is online, re-enable daily backup jobs
# Verify first backup runs successfully
proxmox-backup-client run --all
```

**Critical Action:** Until HDD is restored, implement manual daily snapshot policy:
```bash
# Emergency: Manual snapshots to NVMe to extend retention
# (Run daily at 03:00 UTC)
for vmid in 100 101 102 103 104; do
  qm snapshot $vmid emergency-$(date +%Y-%m-%d)
done
```

**Total RTO:** 1–24 hours (HDD rebuild/replacement)

**Data Loss Risk:** High if NVMe snapshots older than 48 hours

---

### 3.4 PSU Failure Recovery

#### Scenario: Host loses power suddenly

**Step 1: Detect the failure**
- Host is unresponsive (no ping, no SSH)
- Physical inspection: PSU fan not spinning, no power LED

**Step 2: Hardware replacement**
```bash
# Power off the host (if still responding)
shutdown -h now

# [MANUAL STEP: Remove PSU, replace with identical/rated 650W+ PSU]

# Power back on
# Host will boot automatically

# Monitor boot sequence
# Proxmox should detect all VMs and auto-start them
watch qm list
```

**Step 3: Verify VM recovery**
```bash
# After ~2 min, all VMs should be starting/running
qm list

# Check each VM for corruption (unexpected shutdown may corrupt data)
# See Section 3.1 if any VMs show errors
for vmid in 100 101 102 103 104; do
  qm monitor $vmid | head -20
done
```

**Step 4: Validate services**
- Verify all VMs started cleanly
- Check filesystem integrity on each guest OS (fsck, chkdsk)
- Verify network and inter-VM communication

**Total RTO:** 30 minutes (hardware swap + boot + verification)

**Data Loss Risk:** Low (VMs auto-recover from disk; no backup needed)

**Prevention:** Consider UPS (uninterruptible power supply) for Phase 1+ to graceful shutdown and prevent data corruption.

---

### 3.5 NIC/Network Failure Recovery

#### Scenario: Both NICs down; no network access to Proxmox

**Detection:** Host offline; no response to ping; cannot SSH

**Step 1: Physical inspection**
```bash
# Reboot the host from IPMI or power cycle
# (Requires direct physical access or OOMA if installed)
reboot

# After reboot, connect via serial console or HDMI/keyboard
# (No SSH until network is up)
```

**Step 2: Diagnose NIC status (from console)**
```bash
# List network interfaces
ip link show

# Example output:
# 1: lo: <LOOPBACK,UP>
# 2: eth0: <BROADCAST,MULTICAST> # DOWN (no link)
# 3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> # UP (good)

# If both are DOWN:
# - Check cable connections (reseat cables)
# - Try swapping cables between ports
# - Reboot NIC drivers
modprobe -r <driver>  # unload
modprobe <driver>     # reload
```

**Step 3: Verify network connectivity**
```bash
# Restart networking
systemctl restart networking

# Ping gateway
ping 192.168.1.1

# Check DHCP lease (if using DHCP)
ip addr show eth0
```

**Step 4: Once network is up, verify VM connectivity**
```bash
# SSH back into Proxmox
ssh root@proxmox-ip

# Check if VMs lost network (may need restart)
qm list

# Restart any VMs that need network reset
qm shutdown <vmid>
qm start <vmid>
```

**Total RTO:** 15 minutes (cable swap/reboot)

**Data Loss Risk:** None (network-only outage)

---

### 3.6 Thermal/Cooling Failure Recovery

#### Scenario: CPU fan failed; host shutdowns due to thermal protection

**Symptoms:**
- Host powers off suddenly
- BIOS shows CPU temp >95°C before shutdown
- Kernel logs show "Thermal protect activated"

**Step 1: Cool-down phase**
```bash
# Power off the host
shutdown -h now

# Wait 30 minutes for natural cool-down
# Remove side panel for airflow if indoors (temporary)

# Do NOT power on until cool-down complete
# Risk: thermal cycling damage to capacitors
```

**Step 2: Inspect cooling system**
```bash
# Check CPU cooler fan
# - Verify fan spins freely (no debris)
# - Check thermal paste on cooler base
# - Reseat cooler if thermal paste is dry/cracked

# Clean case fans and filters
# - Compressed air: remove dust from heatsink fins
# - Verify all airflow paths are clear

# Check case airflow
# - Ensure intake fans pull cool air from front
# - Ensure exhaust fans push hot air out the rear
# - No blocked vents
```

**Step 3: Power on and verify temperatures**
```bash
# Power on the host
# Monitor boot logs for thermal warnings

# Once booted, check sensors
sensors | grep -i cpu
cat /proc/cpuinfo | grep temp

# Under load (start a VM), verify temps don't exceed 80°C
# Idle: should be 35–55°C
# Load: should be 60–80°C
```

**Step 4: If temperatures are still high**
```bash
# Likely cause: thermal paste has dried or cooler is misaligned
# Solution:
# - Power off again
# - Remove CPU cooler
# - Clean thermal paste: IPA (isopropyl alcohol) + lint-free cloth
# - Apply fresh thermal paste (thin layer)
# - Reinstall cooler and tighten evenly
# - Power on and re-check temperatures
```

**Step 5: Once stable, check VMs for inconsistencies**
```bash
# Unexpected thermal shutdown may corrupt data on VMs
# See Section 3.1 if any VMs show errors

qm list  # check status
for vmid in 100 101 102 103 104; do
  qm monitor $vmid | grep "status\|guest\|error" | head -5
done
```

**Total RTO:** 1–2 hours (cool-down + inspection + thermal paste replacement)

**Data Loss Risk:** Low (filesystem journals recover from dirty shutdown)

**Preventive Measure:** Add CPU temperature monitoring (Phase 5+) to alert before shutdown threshold.

---

## 4. Backup Validation & Restore Testing

### 4.1 Weekly Backup Integrity Check

**Schedule:** Every Sunday, 09:00 UTC

**Procedure:**
```bash
#!/bin/bash
# weekly-backup-test.sh

echo "=== Weekly Backup Validation ==="
date

# Check HDD archive pool health
echo "1. Archive pool status:"
zpool status archive

# Verify all backup snapshots exist on HDD
echo "2. Backup snapshots on archive pool:"
zfs list -t snapshot archive/vm-backup -S creation | tail -10

# Count total snapshots (should be ~30-35 for full retention: 7 daily + 4 weekly)
echo "3. Total retention snapshots:"
zfs list -t snapshot archive/vm-backup -H | wc -l

# Verify NVMe to HDD backup flow (last 24h)
echo "4. Latest NVMe snapshot:"
zfs list -t snapshot -S creation tank/vm-disks@* | head -1

echo "5. Latest HDD backup:"
zfs list -t snapshot archive/vm-backup@* -S creation | head -1

# Both timestamps should be within 24 hours

# Test restore of oldest backup (disaster recovery worst-case)
echo "6. Testing restore of 2-week-old backup (DC01):"
# Do NOT actually restore; just verify snapshot exists and can be sent
zfs send -n archive/vm-backup/dc01@oldest-backup-2w | head -5

echo "=== Backup Validation Complete ==="
```

**Expected Results:**
- ✓ Archive pool status: ONLINE
- ✓ All VM snapshots present on HDD (5 VMs × 30 snapshots = 150 snapshots)
- ✓ Timestamp delta between NVMe and HDD: < 24 hours (last backup ran successfully)
- ✓ Oldest backup (2 weeks old) is still readable

**Failure Action:** If any backup missing or HDD pool degraded → escalate to **Section 3.2 or 3.3**

---

### 4.2 Quarterly Restore Drill

**Schedule:** First week of every quarter (Jan, Apr, Jul, Oct)

**Procedure:** Restore one VM from backup to temporary location; verify functionality

```bash
#!/bin/bash
# quarterly-restore-drill.sh
# Rotate which VM is restored each quarter

TARGET_VM=$1  # e.g., "dc01", "client01", "ubu01"
TARGET_VMID=$2  # e.g., 101

echo "=== Quarterly Restore Drill: $TARGET_VM ==="

# Step 1: Find backup to restore (use 1-week-old snapshot, not latest)
RESTORE_SNAPSHOT=$(zfs list -t snapshot archive/vm-backup/$TARGET_VM -S creation | tail -5 | head -1 | awk '{print $1}')

echo "Restoring from snapshot: $RESTORE_SNAPSHOT"

# Step 2: Create temporary VM from backup
# (Do NOT boot it; just create disk)
TEMP_VMID=$((1000 + VMID))  # e.g., 1101 for temporary DC01
zfs send $RESTORE_SNAPSHOT | zfs recv tank/vm-$TARGET_VM-test-$RESTORE_SNAPSHOT

# Step 3: Verify restored disk is readable
zfs list tank/vm-$TARGET_VM-test*
zfs get compressratio tank/vm-$TARGET_VM-test*

# Step 4: Clean up (do not keep temporary disk)
zfs destroy tank/vm-$TARGET_VM-test*

echo "=== Restore Drill Complete: $TARGET_VM restored, verified, cleaned up ==="
```

**Rotation Schedule:**
- Q1 (Jan): Restore DC01
- Q2 (Apr): Restore CLIENT01
- Q3 (Jul): Restore UBU01
- Q4 (Oct): Restore pfSense

**Expected Results:**
- ✓ Snapshot found on HDD archive
- ✓ Snapshot can be sent without errors
- ✓ Temporary disk created successfully on NVMe
- ✓ Temporary disk is readable and consistent

**Failure Action:** If restore fails → escalate to **Section 3.2 or 3.3**

---

## 5. Escalation Matrix

| Failure Type | RTO | Detection | Owner | Escalation |
|--------------|-----|-----------|-------|------------|
| VM corruption | 1 hr | Console/alert | Lab owner | N/A (self-heal via snapshot) |
| NVMe failure | 2 hrs | SMART alert + zpool status | Lab owner | Hardware vendor (RMA if failed) |
| HDD failure | 24 hrs | Backup job failure | Lab owner | Hardware vendor (RMA if failed) |
| PSU failure | 30 min | Power loss + no boot | Lab owner | Hardware vendor (purchase replacement) |
| NIC failure | 15 min | Network loss | Lab owner | Reseat cables; swap NIC if add-in |
| Thermal failure | 1–2 hrs | Thermal alerts + shutdown | Lab owner | Clean cooler; replace if bearing failed |

**Escalation Contact:**
- **Immediate action:** Lab owner (self)
- **Hardware RMA:** Check warranty status; order replacement part (NVMe, HDD, PSU)
- **Data recovery:** If all backups lost, consult professional data recovery (cost ~$1000–$5000)

---

## 6. Prevention & Maintenance Schedule

### 6.1 Daily (automated)

- [ ] Daily VM snapshot backup at 03:00 UTC (7 daily retention)
- [ ] Daily backup validation: verify HDD archive pool is ONLINE

### 6.2 Weekly (manual)

- [ ] Sunday 09:00 UTC: Run `weekly-backup-test.sh` (Section 4.1)
- [ ] Check NVMe SMART health: `smartctl -a /dev/nvme0n1`
- [ ] Check HDD SMART health: `smartctl -a /dev/sda`
- [ ] Visual inspection: All fans spinning, no LED errors, cables secure

### 6.3 Monthly

- [ ] Review Proxmox logs for warnings/errors: `journalctl -u pve-cluster -n 100`
- [ ] Verify backup retention policy still correct: `zfs list -t snapshot | wc -l`
- [ ] Test console access: SSH into each VM and verify responsiveness

### 6.4 Quarterly

- [ ] Run `quarterly-restore-drill.sh` (Section 4.2) for one VM
- [ ] Clean dust from case and heatsink (compressed air)
- [ ] Update Proxmox if security patches available
- [ ] Full disk inventory: `zfs list -r tank; zfs list -r archive`

### 6.5 Annual (pre-Phase 1)

- [ ] Full system backup audit: verify all VMs have snapshots, no orphaned data
- [ ] Thermal paste inspection on CPU cooler
- [ ] Test emergency shutdown procedures
- [ ] Review DR runbook; update based on lessons learned

---

## 7. Testing Checklist

### 7.1 Restore Procedure Test

| Test Case | Procedure | Expected Result | Frequency |
|-----------|-----------|-----------------|-----------|
| **T1: VM snapshot rollback** | See Section 3.1 | VM boots from 24h-old snapshot without errors | Quarterly (quarterly-restore-drill.sh) |
| **T2: NVMe to HDD restore** | See Section 3.2 | All VMs restored from HDD and boot successfully | Annual (full DR drill) |
| **T3: HDD replace & rebuild** | See Section 3.3 | Archive pool re-created; backups resume | Annual (full DR drill) |
| **T4: PSU replace** | See Section 3.4 | Host boots; all VMs auto-recover | Annual (full DR drill) |
| **T5: NIC failover** | See Section 3.5 | Secondary NIC takes over; VMs regain network | Annual (full DR drill) |
| **T6: Thermal recovery** | See Section 3.6 | CPU cooler cleaned; temps normal; VMs resume | Annual (full DR drill) |

### 7.2 Full DR Drill (Annual)

Simulate total host failure; restore from backup to temporary hardware (if available).

**Procedure:**
1. Create full host backup to external USB drive
2. Boot Proxmox from live ISO on test hardware
3. Restore host configuration from backup
4. Restore all VMs from HDD archive
5. Verify all VMs boot and are operational
6. Document time taken (should be <4 hours for full recovery)

**Success Criteria:**
- ✓ All VMs boot without errors
- ✓ Network connectivity verified
- ✓ Application services running (DC01 AD, UBU01 SSH, etc.)
- ✓ Total time < 4 hours RTO

---

## 8. Known Limitations & Mitigations

| Limitation | Impact | Mitigation |
|-----------|--------|-----------|
| **Single host design** (no HA cluster) | If Proxmox host fails completely, entire lab is down | Phase 1+: Plan 2-node cluster with shared storage |
| **NVMe hot tier only ~500GB usable** | Limited space for multiple large VMs; snapshots may fill disk | Monitor disk usage daily; implement quota per VM |
| **24-hour RPO** | Up to 24h data loss if both NVMe and HDD fail | Phase 1+: Implement 3-2-1 backup rule (3 copies, 2 media types, 1 offsite) |
| **No RAID on NVMe** | Single point of failure; no redundancy | Phase 1+: Upgrade to dual-NVMe with RAID-1 (mirror) |
| **Home lab, no UPS** | Unexpected power loss corrupts data | Phase 1+: Add UPS (150W minimum) for graceful shutdown |
| **Manual testing only** | Restore procedures not automated; human error risk | Phase 5+: Integrate with osTicket ITSM; schedule automated tests |

---

## 9. Runbook Maintenance

**Document Owner:** Infrastructure Architect  
**Review Cycle:** Annually (minimum); after any major incident  
**Version History:**

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01-31 | Initial draft; all failure scenarios covered |
| 1.1 | TBD | Updates post-Phase 1 (HA cluster, 3-2-1 backup) |
| 2.0 | TBD | Full automation via Phase 8 (Ansible/Terraform) |

**Last Reviewed:** 2025-01-31  
**Next Review Due:** 2026-01-31

---

## 10. Contact & Escalation

| Role | Contact | Availability | Authority |
|------|---------|--------------|-----------|
| Lab Owner / Infrastructure Architect | (self) | Daily | All decisions |
| Emergency Vendor Support | (TBD) | 24/7 | Hardware RMA |
| Data Recovery Service | (TBD) | Business hours | Last resort (only if all backups lost) |

---

## Appendix A: Quick Reference Commands

```bash
# ==== CHECK SYSTEM HEALTH ====
qm list                           # All VMs
zpool status                      # ZFS pool status
zfs list -t snapshot -S creation  # Latest snapshots
smartctl -a /dev/nvme0n1          # NVMe SMART
smartctl -a /dev/sda              # HDD SMART
sensors                           # CPU/system temps

# ==== RESTORE A VM ====
qm rollback <vmid> <snapshot>     # Rollback VM to snapshot
qm start <vmid>                   # Start VM

# ==== RESTORE BACKUP (HDD → NVMe) ====
zfs send archive/vm-backup/dc01@snapshot | zfs recv tank/vm-dc01-disk-1

# ==== CREATE EMERGENCY SNAPSHOT ====
qm snapshot <vmid> emergency-$(date +%Y-%m-%d)

# ==== MONITOR ONGOING BACKUP ====
zfs list -t snapshot -S creation tank  # Latest snapshots
tail -f /var/log/backup/daily.log      # Backup logs
```

---

**END OF RUNBOOK**
