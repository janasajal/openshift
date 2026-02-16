# OpenShift Virtualization - LVM (VG/LV) Configuration Guide for RHEL VMs
## Author: Sajal Jana

## Overview
By default, cloud images and automated VM provisioning create filesystems directly on disk partitions. This guide shows how to configure RHEL VMs in OpenShift Virtualization to use **LVM (Logical Volume Manager)** with Volume Groups (VG) and Logical Volumes (LV) for enterprise flexibility.

**Why Use LVM?**
- Dynamic volume resizing without downtime
- Snapshots at LV level (inside VM)
- Flexible storage management
- Mirroring and striping capabilities
- Industry standard for RHEL production systems

---

## Method 1: Create VM with Raw Disks and Configure LVM Manually (Recommended for Learning)

### Scenario
Database team needs a RHEL 9 VM with separate LVM logical volumes for OS, application data, and logs with ability to grow independently.

### Objective
Create VM with multiple raw disks, manually partition and configure LVM inside the VM using standard RHEL practices.

### Steps

#### 1.1 Create VM with Multiple Raw Disks

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create** → **From template**
3. Select **RHEL 9 Server** template
4. Click **Customize VirtualMachine**
5. Set basic details:
   - **Name**: `rhel9-lvm-vm`
   - **Project**: Your namespace
6. In **Overview** tab:
   - **CPU**: `4 cores`
   - **Memory**: `8 GiB`

#### 1.2 Configure Boot Disk (Minimal OS)

1. Go to **Disks** tab
2. Click on the **rootdisk** entry
3. Click **Edit**
4. Configure:
   - **Size**: `30 GiB` (minimal for OS only)
   - **StorageClass**: Select your storage class
   - **Interface**: `virtio`
5. Click **Save**

#### 1.3 Add Additional Raw Disks for LVM

**Add Data Disk:**
1. Still in **Disks** tab, click **Add disk**
2. Configure:
   - **Source**: `Blank (creates PVC)`
   - **Name**: `data-disk`
   - **Size**: `100 GiB`
   - **Type**: `Disk`
   - **Interface**: `virtio`
   - **StorageClass**: Same as rootdisk
3. Click **Add**

**Add Logs Disk:**
1. Click **Add disk** again
2. Configure:
   - **Source**: `Blank (creates PVC)`
   - **Name**: `logs-disk`
   - **Size**: `50 GiB`
   - **Type**: `Disk`
   - **Interface**: `virtio`
   - **StorageClass**: Same as rootdisk
3. Click **Add**

**Add Application Disk:**
1. Click **Add disk** again
2. Configure:
   - **Source**: `Blank (creates PVC)`
   - **Name**: `app-disk`
   - **Size**: `80 GiB`
   - **Type**: `Disk`
   - **Interface**: `virtio`
3. Click **Add**

#### 1.4 Configure Cloud-init for Initial Access

1. Go to **Scripts** tab
2. Enable **Cloud-init**
3. Click **Edit**
4. Configure basic access:
   ```yaml
   #cloud-config
   user: admin
   password: redhat123
   chpasswd: { expire: False }
   ssh_pwauth: True
   ```
5. Click **Save**

#### 1.5 Create and Start VM

1. Click **Create VirtualMachine**
2. Wait for creation to complete
3. Click **Actions** → **Start**
4. Wait for status: `Running`

#### 1.6 Verify Disks in VM

1. Go to **Console** tab
2. Login with `admin` / `redhat123`
3. List all block devices:
   ```bash
   lsblk
   ```

**Expected Output:**
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    252:0    0   30G  0 disk 
├─vda1 252:1    0    1G  0 part /boot
└─vda2 252:2    0   29G  0 part /
vdb    252:16   0  100G  0 disk 
vdc    252:32   0   50G  0 disk 
vdd    252:48   0   80G  0 disk 
```

---

### 1.7 Configure LVM on Data Disk (VG + LV Setup)

#### Step 1: Create Physical Volume (PV)

```bash
# Switch to root
sudo -i

# Create PV on data disk
pvcreate /dev/vdb

# Verify PV creation
pvdisplay
pvs
```

**Expected Output:**
```
  "/dev/vdb" is a new physical volume of "100.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/vdb
  VG Name               
  PV Size               100.00 GiB
```

#### Step 2: Create Volume Group (VG)

```bash
# Create VG named 'datavg' using /dev/vdb
vgcreate datavg /dev/vdb

# Verify VG creation
vgdisplay datavg
vgs
```

**Expected Output:**
```
  --- Volume group ---
  VG Name               datavg
  System ID             
  Format                lvm2
  VG Size               <100.00 GiB
  PE Size               4.00 MiB
  Total PE              25599
  Free  PE              25599
```

#### Step 3: Create Logical Volumes (LV)

**Create LV for database data (60GB):**
```bash
lvcreate -L 60G -n db_data_lv datavg

# Verify
lvdisplay /dev/datavg/db_data_lv
lvs
```

**Create LV for database temp (30GB):**
```bash
lvcreate -L 30G -n db_temp_lv datavg

# Verify remaining space
vgs datavg
```

**Expected Output:**
```
  VG     #PV #LV #SN Attr   VSize    VFree 
  datavg   1   2   0 wz--n- <100.00g <10.00g
```

#### Step 4: Create Filesystems on LVs

**Format db_data_lv with XFS:**
```bash
mkfs.xfs /dev/datavg/db_data_lv
```

**Format db_temp_lv with XFS:**
```bash
mkfs.xfs /dev/datavg/db_temp_lv
```

#### Step 5: Create Mount Points and Mount

```bash
# Create mount points
mkdir -p /data/db
mkdir -p /data/temp

# Mount filesystems
mount /dev/datavg/db_data_lv /data/db
mount /dev/datavg/db_temp_lv /data/temp

# Verify mounts
df -h | grep data
```

**Expected Output:**
```
/dev/mapper/datavg-db_data_lv   60G  512M   60G   1% /data/db
/dev/mapper/datavg-db_temp_lv   30G  256M   30G   1% /data/temp
```

#### Step 6: Make Mounts Persistent (Add to /etc/fstab)

```bash
# Get UUID of logical volumes
blkid /dev/datavg/db_data_lv
blkid /dev/datavg/db_temp_lv

# Or use device mapper paths directly (recommended for LVM)
echo "/dev/mapper/datavg-db_data_lv /data/db xfs defaults 0 0" >> /etc/fstab
echo "/dev/mapper/datavg-db_temp_lv /data/temp xfs defaults 0 0" >> /etc/fstab

# Verify fstab
cat /etc/fstab

# Test fstab (unmount and remount all)
umount /data/db
umount /data/temp
mount -a

# Verify
df -h | grep data
```

---

### 1.8 Configure LVM on Logs Disk (Separate VG)

```bash
# Create PV
pvcreate /dev/vdc

# Create logs VG
vgcreate logsvg /dev/vdc

# Create LV for application logs
lvcreate -L 40G -n app_logs_lv logsvg

# Create LV for system logs
lvcreate -L 10G -n sys_logs_lv logsvg

# Format filesystems
mkfs.xfs /dev/logsvg/app_logs_lv
mkfs.xfs /dev/logsvg/sys_logs_lv

# Create mount points
mkdir -p /logs/application
mkdir -p /logs/system

# Mount
mount /dev/logsvg/app_logs_lv /logs/application
mount /dev/logsvg/sys_logs_lv /logs/system

# Add to fstab
echo "/dev/mapper/logsvg-app_logs_lv /logs/application xfs defaults 0 0" >> /etc/fstab
echo "/dev/mapper/logsvg-sys_logs_lv /logs/system xfs defaults 0 0" >> /etc/fstab

# Verify
df -h | grep logs
vgs
lvs
```

---

### 1.9 Configure LVM on Application Disk

```bash
# Create PV
pvcreate /dev/vdd

# Create application VG
vgcreate appvg /dev/vdd

# Create LV for application binaries
lvcreate -L 30G -n app_bin_lv appvg

# Create LV for application config
lvcreate -L 10G -n app_config_lv appvg

# Create LV for application cache
lvcreate -L 20G -n app_cache_lv appvg

# Format
mkfs.xfs /dev/appvg/app_bin_lv
mkfs.xfs /dev/appvg/app_config_lv
mkfs.xfs /dev/appvg/app_cache_lv

# Create mount points
mkdir -p /opt/app/bin
mkdir -p /opt/app/config
mkdir -p /opt/app/cache

# Mount
mount /dev/appvg/app_bin_lv /opt/app/bin
mount /dev/appvg/app_config_lv /opt/app/config
mount /dev/appvg/app_cache_lv /opt/app/cache

# Add to fstab
echo "/dev/mapper/appvg-app_bin_lv /opt/app/bin xfs defaults 0 0" >> /etc/fstab
echo "/dev/mapper/appvg-app_config_lv /opt/app/config xfs defaults 0 0" >> /etc/fstab
echo "/dev/mapper/appvg-app_cache_lv /opt/app/cache xfs defaults 0 0" >> /etc/fstab

# Verify complete setup
lsblk
vgs
lvs
df -h
```

---

### 1.10 Final Verification

```bash
# Complete LVM overview
pvs
vgs
lvs

# Check all mounts
df -h

# Verify fstab
cat /etc/fstab

# Test reboot persistence
reboot
```

After reboot:
1. Login via Console
2. Run `df -h` - all LVM mounts should be present
3. Run `lvs` - all logical volumes should be active

---

## Method 2: Extend LVM Volumes (Common Production Task)

### Scenario
Database grows and needs more space in `/data/db`. You need to expand the logical volume from 60GB to 100GB.

### Steps

#### 2.1 Expand Disk in OpenShift Console

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on `rhel9-lvm-vm`
3. Go to **Disks** tab
4. Click on `data-disk` (the 100GB disk)
5. Click **Edit**
6. Change **Size** from `100 GiB` to `150 GiB`
7. Click **Save**
8. Wait for expansion to complete

#### 2.2 Extend Physical Volume in VM

```bash
# Login to VM console
sudo -i

# Check current PV size
pvs

# Rescan and resize PV to use new space
pvresize /dev/vdb

# Verify new size
pvs
vgs datavg
```

**Expected Output:**
```
  PV         VG     Fmt  Attr PSize    PFree 
  /dev/vdb   datavg lvm2 a--  <150.00g <60.00g
```

#### 2.3 Extend Logical Volume

```bash
# Extend db_data_lv by 40GB (from 60GB to 100GB)
lvextend -L +40G /dev/datavg/db_data_lv

# Or extend to use all free space
# lvextend -l +100%FREE /dev/datavg/db_data_lv

# Verify
lvs
```

#### 2.4 Resize Filesystem

**For XFS (cannot shrink, only grow):**
```bash
# Resize XFS filesystem to use new LV size
xfs_growfs /data/db

# Verify
df -h /data/db
```

**For ext4 (if you used ext4):**
```bash
resize2fs /dev/datavg/db_data_lv
df -h /data/db
```

**Expected Result:**
```
/dev/mapper/datavg-db_data_lv  100G  512M  100G   1% /data/db
```

---

## Method 3: Use Cloud-init to Automate LVM Setup During VM Creation

### Scenario
Need to deploy multiple VMs with consistent LVM layout automatically without manual intervention.

### Steps

#### 3.1 Create VM with Cloud-init LVM Configuration

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create** → **From template**
3. Select RHEL 9 template
4. Click **Customize VirtualMachine**
5. Configure disks as in Method 1 (rootdisk + 3 additional disks)
6. Go to **Scripts** tab
7. Enable **Cloud-init**
8. Click **Edit**
9. Use advanced cloud-init configuration:

```yaml
#cloud-config
user: admin
password: redhat123
chpasswd: { expire: False }
ssh_pwauth: True

# Run commands during first boot to setup LVM
bootcmd:
  - [ cloud-init-per, once, pvcreatevdb, pvcreate, /dev/vdb ]
  - [ cloud-init-per, once, vgcreatedata, vgcreate, datavg, /dev/vdb ]
  - [ cloud-init-per, once, lvcreatdb, lvcreate, -L, 60G, -n, db_data_lv, datavg ]
  - [ cloud-init-per, once, mkfsdb, mkfs.xfs, /dev/datavg/db_data_lv ]
  
mounts:
  - [ /dev/datavg/db_data_lv, /data/db, xfs, "defaults", "0", "0" ]

runcmd:
  - mkdir -p /data/db
  - mount -a
  - echo "LVM setup completed" > /tmp/lvm-setup-done.txt
```

10. Click **Save**
11. Click **Create VirtualMachine**
12. Start the VM

#### 3.2 Verify Automated Setup

1. Wait 3-5 minutes for cloud-init to complete
2. Open **Console** tab
3. Login and verify:
   ```bash
   sudo -i
   lvs
   df -h /data/db
   cat /tmp/lvm-setup-done.txt
   ```

**Limitation:** Cloud-init bootcmd has limitations for complex LVM setups. For production, consider Method 4.

---

## Method 4: Create Custom RHEL Image with Pre-configured LVM (Advanced)

### Scenario
Need standard VM image with LVM already configured for rapid deployment across multiple projects.

### Overview
This method involves:
1. Create a base VM with LVM manually configured (Method 1)
2. Sysprep/generalize the VM
3. Create a custom template from it
4. Deploy new VMs from the template

### Steps

#### 4.1 Prepare Source VM

1. Follow **Method 1** completely to create VM with LVM layout
2. Install all required software packages
3. Configure any standard settings (timezone, locale, etc.)

#### 4.2 Clean VM for Template Creation

```bash
# Login to VM
sudo -i

# Remove SSH host keys (regenerated on first boot)
rm -f /etc/ssh/ssh_host_*

# Remove machine-id
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id

# Clean cloud-init
cloud-init clean --logs --seed

# Clear bash history
history -c
cat /dev/null > ~/.bash_history

# Clear logs
find /var/log -type f -delete

# Shutdown VM
shutdown -h now
```

#### 4.3 Create Template from VM (Via Console)

1. Navigate to **Virtualization** → **VirtualMachines**
2. Find your prepared VM (ensure it's **Stopped**)
3. Click **Actions** → **Create template**
4. Configure template:
   - **Name**: `rhel9-lvm-custom-template`
   - **Description**: `RHEL 9 with pre-configured LVM (datavg, logsvg, appvg)`
   - **Provider**: `Custom`
5. Click **Create**

#### 4.4 Deploy VMs from Custom Template

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create** → **From template**
3. Select your custom template: `rhel9-lvm-custom-template`
4. Click **Customize VirtualMachine** (if needed)
5. Provide unique name and adjust resources
6. Create and start VM
7. **Result**: New VM boots with LVM already configured exactly like source

---

## Common LVM Operations in OpenShift Virtualization VMs

### Add Disk to Existing Volume Group

**Scenario:** Need to expand datavg by adding another disk.

**Steps:**

1. **In OpenShift Console:**
   - Add new disk to VM (e.g., 50GB)
   - Start or hot-plug the disk

2. **In VM Console:**
   ```bash
   sudo -i
   
   # Identify new disk
   lsblk
   # Assume new disk is /dev/vde
   
   # Create PV
   pvcreate /dev/vde
   
   # Extend existing VG
   vgextend datavg /dev/vde
   
   # Verify
   vgs datavg
   pvs
   
   # Now you can create new LVs or extend existing ones
   lvextend -L +50G /dev/datavg/db_data_lv
   xfs_growfs /data/db
   ```

### Create LVM Snapshot (Inside VM)

**Scenario:** Need snapshot before database upgrade.

```bash
sudo -i

# Create snapshot of db_data_lv (10GB snapshot space)
lvcreate -L 10G -s -n db_data_snap /dev/datavg/db_data_lv

# Verify
lvs

# Mount snapshot for backup
mkdir /mnt/db_snapshot
mount -o ro /dev/datavg/db_data_snap /mnt/db_snapshot

# Backup data
tar czf /backup/db-backup.tar.gz /mnt/db_snapshot

# Unmount and remove snapshot when done
umount /mnt/db_snapshot
lvremove /dev/datavg/db_data_snap
```

### Remove Logical Volume

```bash
sudo -i

# Unmount filesystem
umount /data/temp

# Remove from fstab
vi /etc/fstab  # Delete the line for /data/temp

# Remove LV
lvremove /dev/datavg/db_temp_lv

# Verify
lvs
vgs datavg  # Shows reclaimed space
```

### Monitor LVM Performance

```bash
# Check I/O stats for LVs
sudo lvs -o +devices

# Check VG free space
sudo vgs

# Detailed VG information
sudo vgdisplay -v datavg

# Check which LVs are using space
sudo lvs -o +lv_size,lv_free

# Monitor LVM activity
sudo dmsetup status
```

---

## Troubleshooting Guide

### Issue 1: PV Creation Fails - "Device is already in use"

**Cause:** Disk has existing partitions or filesystem.

**Solution:**
```bash
# View partition table
sudo fdisk -l /dev/vdb

# Wipe existing signatures
sudo wipefs -a /dev/vdb

# Try pvcreate again
sudo pvcreate /dev/vdb
```

### Issue 2: Mount Fails After Reboot - LV Not Found

**Cause:** LVs not activated during boot.

**Solution:**
```bash
# Check if VGs are active
sudo vgs

# Activate all VGs
sudo vgchange -ay

# Or activate specific VG
sudo vgchange -ay datavg

# Retry mount
sudo mount -a
```

**Permanent Fix:**
Ensure `/etc/fstab` uses correct device paths (`/dev/mapper/vgname-lvname`).

### Issue 3: Cannot Extend LV - "Insufficient free space"

**Cause:** VG is full.

**Solution:**
```bash
# Check VG free space
sudo vgs datavg

# Option 1: Expand underlying disk in OpenShift Console
# Then resize PV:
sudo pvresize /dev/vdb

# Option 2: Add another disk to VG
# Add disk in OpenShift Console first
sudo pvcreate /dev/vde
sudo vgextend datavg /dev/vde

# Now extend LV
sudo lvextend -L +40G /dev/datavg/db_data_lv
sudo xfs_growfs /data/db
```

### Issue 4: XFS Filesystem Cannot Be Shrunk

**Cause:** XFS does not support shrinking.

**Solution:**
```bash
# Option 1: Backup data, recreate smaller LV, restore
sudo tar czf /backup/data.tar.gz /data/db
sudo umount /data/db
sudo lvremove /dev/datavg/db_data_lv
sudo lvcreate -L 40G -n db_data_lv datavg
sudo mkfs.xfs /dev/datavg/db_data_lv
sudo mount /dev/datavg/db_data_lv /data/db
sudo tar xzf /backup/data.tar.gz -C /

# Option 2: Use ext4 instead of XFS if shrinking is required
```

### Issue 5: LVM Metadata Corruption

**Symptoms:** `vgs` or `lvs` shows errors.

**Solution:**
```bash
# Backup LVM metadata
sudo vgcfgbackup datavg

# Restore metadata
sudo vgcfgrestore datavg

# Activate VG
sudo vgchange -ay datavg

# Check filesystem
sudo xfs_repair /dev/datavg/db_data_lv
```

---

## Best Practices for LVM in OpenShift Virtualization

### 1. Volume Group Naming Convention
- Use descriptive VG names: `datavg`, `logsvg`, `appvg`
- Avoid generic names like `vg00`, `vg01`
- Document VG purpose in VM annotations

### 2. Logical Volume Naming Convention
- Include purpose in LV name: `db_data_lv`, `app_logs_lv`
- Avoid abbreviations that aren't obvious
- Use consistent suffix: `_lv` for all logical volumes

### 3. Leave Free Space in VG
- Don't allocate 100% of VG space immediately
- Keep 10-20% free for:
  - LVM snapshots
  - Emergency expansions
  - Temporary volumes

### 4. Filesystem Selection
- **XFS**: Best for large files, databases, high performance
  - Cannot shrink
  - Excellent for workloads that grow
- **ext4**: General purpose, can shrink
  - Better for filesystems that may need to shrink
  - Good all-around choice

### 5. Use Multiple Disks for Different Purposes
- **Separate VGs for different workload types:**
  - Data VG (database files)
  - Logs VG (application logs)
  - Application VG (binaries, config)
- **Benefits:**
  - Independent expansion
  - Better I/O isolation
  - Easier troubleshooting

### 6. Disk Expansion Strategy
- Expand PVC in OpenShift Console first
- Then expand PV inside VM: `pvresize /dev/vdX`
- Then extend LV: `lvextend`
- Finally resize filesystem: `xfs_growfs` or `resize2fs`

### 7. Backup Strategy
- Use LVM snapshots for consistent backups
- Export VM YAML and PVC definitions
- Document LVM layout in VM annotations
- Test restore procedures regularly

### 8. Monitoring
- Monitor VG free space (alert at 80%)
- Monitor LV utilization
- Check `/var/log/messages` for LVM errors
- Use `vgs`, `lvs`, `pvs` regularly

### 9. Performance Considerations
- Use virtio interface for all disks
- Align LVM PE size with storage backend block size
- Consider using different storage classes for different VGs
- For databases, use dedicated VG/LV (no sharing)

### 10. Documentation
- Document LVM layout in VM description
- Tag VMs with LVM structure in OpenShift labels
- Maintain runbook for expansion procedures
- Keep fstab backup in version control

---

## Complete Example: Production Database Server

### Requirements
- RHEL 9 VM for PostgreSQL database
- Separate volumes for: OS, data, WAL logs, backup, temp
- LVM for flexibility
- High performance storage for data and WAL

### Implementation

#### VM Configuration

**Disks:**
- `rootdisk`: 30GB (OS)
- `data-disk`: 200GB (database data)
- `wal-disk`: 100GB (WAL logs, high-performance storage class)
- `backup-disk`: 300GB (backup area)
- `temp-disk`: 50GB (temporary files)

#### LVM Layout

```bash
# Data VG (200GB disk)
pvcreate /dev/vdb
vgcreate pgdata_vg /dev/vdb
lvcreate -L 150G -n pg_data_lv pgdata_vg
lvcreate -L 30G -n pg_index_lv pgdata_vg
# Keep 20GB free for snapshots

# WAL VG (100GB disk, high-performance)
pvcreate /dev/vdc
vgcreate pgwal_vg /dev/vdc
lvcreate -L 80G -n pg_wal_lv pgwal_vg
# Keep 20GB free

# Backup VG (300GB disk)
pvcreate /dev/vdd
vgcreate pgbackup_vg /dev/vdd
lvcreate -L 250G -n pg_backup_lv pgbackup_vg
# Keep 50GB free

# Temp VG (50GB disk)
pvcreate /dev/vde
vgcreate pgtemp_vg /dev/vde
lvcreate -L 40G -n pg_temp_lv pgtemp_vg
# Keep 10GB free

# Format all filesystems
mkfs.xfs /dev/pgdata_vg/pg_data_lv
mkfs.xfs /dev/pgdata_vg/pg_index_lv
mkfs.xfs /dev/pgwal_vg/pg_wal_lv
mkfs.xfs /dev/pgbackup_vg/pg_backup_lv
mkfs.xfs /dev/pgtemp_vg/pg_temp_lv

# Create mount points
mkdir -p /var/lib/pgsql/data
mkdir -p /var/lib/pgsql/index
mkdir -p /var/lib/pgsql/wal
mkdir -p /backup/postgresql
mkdir -p /tmp/pgsql

# Mount
mount /dev/pgdata_vg/pg_data_lv /var/lib/pgsql/data
mount /dev/pgdata_vg/pg_index_lv /var/lib/pgsql/index
mount /dev/pgwal_vg/pg_wal_lv /var/lib/pgsql/wal
mount /dev/pgbackup_vg/pg_backup_lv /backup/postgresql
mount /dev/pgtemp_vg/pg_temp_lv /tmp/pgsql

# Update fstab
cat >> /etc/fstab << EOF
/dev/pgdata_vg/pg_data_lv    /var/lib/pgsql/data    xfs  defaults  0  0
/dev/pgdata_vg/pg_index_lv   /var/lib/pgsql/index   xfs  defaults  0  0
/dev/pgwal_vg/pg_wal_lv      /var/lib/pgsql/wal     xfs  defaults  0  0
/dev/pgbackup_vg/pg_backup_lv /backup/postgresql    xfs  defaults  0  0
/dev/pgtemp_vg/pg_temp_lv    /tmp/pgsql             xfs  defaults  0  0
EOF

# Set ownership for PostgreSQL
chown -R postgres:postgres /var/lib/pgsql
chown -R postgres:postgres /tmp/pgsql
chmod 700 /var/lib/pgsql/data
```

#### Backup Strategy

```bash
# Create consistent snapshot before backup
lvcreate -L 10G -s -n pg_data_snap /dev/pgdata_vg/pg_data_lv

# Mount snapshot
mkdir /mnt/pg_snapshot
mount -o ro /dev/pgdata_vg/pg_data_snap /mnt/pg_snapshot

# Backup from snapshot
tar czf /backup/postgresql/pgdata-$(date +%Y%m%d).tar.gz /mnt/pg_snapshot

# Cleanup
umount /mnt/pg_snapshot
lvremove -y /dev/pgdata_vg/pg_data_snap
```

---

## Summary Checklist

### Initial Setup
- [ ] Create VM with multiple blank disks in OpenShift Console
- [ ] Boot VM and verify all disks are visible (`lsblk`)
- [ ] Create Physical Volumes on each disk
- [ ] Create Volume Groups (separate VGs for different purposes)
- [ ] Create Logical Volumes with appropriate sizes
- [ ] Format filesystems (XFS recommended)
- [ ] Create mount points
- [ ] Mount filesystems
- [ ] Update /etc/fstab for persistence
- [ ] Verify mounts survive reboot

### Expansion Procedure
- [ ] Expand PVC in OpenShift Console
- [ ] Resize PV in VM: `pvresize /dev/vdX`
- [ ] Extend LV: `lvextend -L +XXG /dev/vg/lv`
- [ ] Resize filesystem: `xfs_growfs` or `resize2fs`
- [ ] Verify new size: `df -h`

### Monitoring
- [ ] Check VG free space weekly: `vgs`
- [ ] Monitor LV usage: `lvs` and `df -h`
- [ ] Review LVM logs: `/var/log/messages`
- [ ] Test snapshot creation quarterly
- [ ] Document LVM layout

---

## Additional Resources

### Red Hat Documentation
- **LVM Administration Guide**: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/
- **OpenShift Virtualization Storage**: https://docs.openshift.com/container-platform/latest/virt/storage/virt-storage-defaults.html

### Useful Commands Reference

```bash
# PV Management
pvcreate /dev/vdX          # Create PV
pvdisplay                  # Show PV details
pvs                        # Summary of PVs
pvremove /dev/vdX          # Remove PV
pvresize /dev/vdX          # Resize PV

# VG Management
vgcreate vgname /dev/vdX   # Create VG
vgdisplay vgname           # Show VG details
vgs                        # Summary of VGs
vgextend vgname /dev/vdX   # Add PV to VG
vgreduce vgname /dev/vdX   # Remove PV from VG
vgremove vgname            # Remove VG

# LV Management
lvcreate -L 10G -n lvname vgname    # Create LV
lvdisplay                           # Show LV details
lvs                                 # Summary of LVs
lvextend -L +10G /dev/vg/lv         # Extend LV
lvreduce -L -10G /dev/vg/lv         # Reduce LV (dangerous)
lvremove /dev/vg/lv                 # Remove LV
lvrename vg oldlv newlv             # Rename LV

# Snapshot Management
lvcreate -L 10G -s -n snapname /dev/vg/lv   # Create snapshot
lvconvert --merge /dev/vg/snapname          # Merge snapshot back

# Filesystem Operations
mkfs.xfs /dev/vg/lv                # Format XFS
mkfs.ext4 /dev/vg/lv               # Format ext4
xfs_growfs /mountpoint             # Resize XFS
resize2fs /dev/vg/lv               # Resize ext4
mount /dev/vg/lv /mnt              # Mount
umount /mnt                        # Unmount
```

---

**End of Guide**

*Last Updated: February 2025*  
*Target Audience: OpenShift Virtualization Administrators*  
*Skill Level: Intermediate to Advanced*
