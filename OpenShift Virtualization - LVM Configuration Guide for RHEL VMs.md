# OpenShift Virtualization - LVM Configuration Guide for RHEL VMs
## Author: Sajal Jana
*Making storage flexible since... well, since LVM was invented* 🎯

---

## Why LVM? (Or: Why Your Future Self Will Thank You)

Remember that time you ran out of disk space and had to explain to management why the database is down? Yeah, LVM prevents that awkward conversation.

**LVM Superpowers:**
- 🔧 Resize volumes without rebooting (like magic, but real)
- 📸 Snapshots before that "totally safe" upgrade
- 🎪 Mix and match disks like storage Tetris
- 🏢 Industry standard (translation: your resume looks better)

> **Pro tip**: Using raw partitions without LVM in 2025 is like using a flip phone because "it just makes calls."

---

## Method 1: Manual LVM Setup (The Learning Experience™)

### Scenario
Database team: "We need separate volumes for data, logs, and... can they grow independently?"  
You: "Hold my coffee ☕"

### The Plan
Create a VM with multiple disks, then channel your inner Linux wizard to configure LVM manually.

---

### Step 1: Create VM with Multiple Disks

1. **OpenShift Console** → **Virtualization** → **VirtualMachines** → **Create**
2. Select **RHEL 9 Server** template → **Customize VirtualMachine**
3. Basic settings:
   - **Name**: `rhel9-lvm-vm`
   - **CPU**: `4 cores` (because we're not monsters)
   - **Memory**: `8 GiB`

### Step 2: Configure the Disks

**Root Disk** (30 GiB - OS only):
- Edit the existing `rootdisk`
- Size: `30 GiB`
- Interface: `virtio` (always virtio, unless you enjoy slow I/O)

**Add Three More Disks** (click "Add disk" for each):

| Disk Name | Size | Purpose | Your Boss Thinks |
|-----------|------|---------|------------------|
| data-disk | 100 GiB | Database data | "That's a lot!" |
| logs-disk | 50 GiB | Application logs | "Logs need space?" |
| app-disk | 80 GiB | Application files | "Why so complicated?" |

- **Source**: `Blank (creates PVC)` for all
- **Type**: `Disk`
- **Interface**: `virtio` (seriously, always virtio)

### Step 3: Cloud-init (Because Typing Passwords is So 2010)

**Scripts** tab → Enable **Cloud-init** → **Edit**:

```yaml
#cloud-config
user: admin
password: redhat123  # Change this in production, please
chpasswd: { expire: False }
ssh_pwauth: True
```

### Step 4: Launch! 🚀

**Create VirtualMachine** → **Start** → Grab coffee while it boots

---

## The Fun Part: Configuring LVM

### Verify Your Disks

Login via **Console** tab:

```bash
lsblk
```

**Expected output** (disk names may vary, but you should see 4 disks):
```
NAME   SIZE  TYPE MOUNTPOINTS
vda    30G   disk 
├─vda1  1G   part /boot
└─vda2 29G   part /
vdb   100G   disk  # ← Data disk (like an empty warehouse)
vdc    50G   disk  # ← Logs disk (filing cabinet)
vdd    80G   disk  # ← App disk (toolbox)
```

> **Note**: If you don't see vdb/vdc/vdd, they're probably hiding. Try `ls -la /dev/vd*` to find them.

---

### Configure Data Disk (The "I Know What I'm Doing" Section)

```bash
sudo -i  # Become root (with great power...)

# Step 1: Create Physical Volume (PV)
# Think of this as putting a "LVM Compatible™" sticker on the disk
pvcreate /dev/vdb
pvs  # Verify (paranoia is a virtue in IT)

# Step 2: Create Volume Group (VG)
# VG = Storage pool. Like a swimming pool, but for data
vgcreate datavg /dev/vdb
vgs  # Check your work

# Step 3: Create Logical Volumes (LV)
# The actual usable volumes (finally!)
lvcreate -L 60G -n db_data_lv datavg      # Database data
lvcreate -L 30G -n db_temp_lv datavg      # Temp files

lvs  # Admire your handiwork
vgs datavg  # Should show ~10G free (for future "oh crap" moments)

# Step 4: Format with XFS
# XFS = Fast. Can't shrink, but who shrinks storage anyway?
mkfs.xfs /dev/datavg/db_data_lv
mkfs.xfs /dev/datavg/db_temp_lv

# Step 5: Mount Everything
mkdir -p /data/{db,temp}
mount /dev/datavg/db_data_lv /data/db
mount /dev/datavg/db_temp_lv /data/temp

df -h | grep data  # Does it work? Celebrate! 🎉
```

### Make It Permanent (Or: Surviving Reboots)

```bash
# Add to fstab (the "please remember this after reboot" file)
cat >> /etc/fstab << EOF
/dev/mapper/datavg-db_data_lv  /data/db    xfs  defaults  0  0
/dev/mapper/datavg-db_temp_lv  /data/temp  xfs  defaults  0  0
EOF

# Test it (trust, but verify)
umount /data/db /data/temp
mount -a
df -h | grep data  # Still there? Good!
```

> **Pro tip**: Always test fstab changes before rebooting. Future you will be grateful.

---

### Configure Logs Disk (Rinse and Repeat)

```bash
# Same dance, different VG
pvcreate /dev/vdc
vgcreate logsvg /dev/vdc
lvcreate -L 40G -n app_logs_lv logsvg
lvcreate -L 10G -n sys_logs_lv logsvg

# Format and mount
mkfs.xfs /dev/logsvg/app_logs_lv
mkfs.xfs /dev/logsvg/sys_logs_lv

mkdir -p /logs/{application,system}
mount /dev/logsvg/app_logs_lv /logs/application
mount /dev/logsvg/sys_logs_lv /logs/system

# Update fstab
cat >> /etc/fstab << EOF
/dev/mapper/logsvg-app_logs_lv  /logs/application  xfs  defaults  0  0
/dev/mapper/logsvg-sys_logs_lv   /logs/system       xfs  defaults  0  0
EOF
```

### Configure Application Disk (You're Getting Good at This!)

```bash
pvcreate /dev/vdd
vgcreate appvg /dev/vdd
lvcreate -L 30G -n app_bin_lv appvg
lvcreate -L 10G -n app_config_lv appvg
lvcreate -L 20G -n app_cache_lv appvg

# Format (I'll spare you the repetition)
mkfs.xfs /dev/appvg/app_bin_lv
mkfs.xfs /dev/appvg/app_config_lv
mkfs.xfs /dev/appvg/app_cache_lv

# Mount
mkdir -p /opt/app/{bin,config,cache}
mount /dev/appvg/app_bin_lv /opt/app/bin
mount /dev/appvg/app_config_lv /opt/app/config
mount /dev/appvg/app_cache_lv /opt/app/cache

# Fstab (last time, I promise)
cat >> /etc/fstab << EOF
/dev/mapper/appvg-app_bin_lv     /opt/app/bin     xfs  defaults  0  0
/dev/mapper/appvg-app_config_lv  /opt/app/config  xfs  defaults  0  0
/dev/mapper/appvg-app_cache_lv   /opt/app/cache   xfs  defaults  0  0
EOF
```

### Victory Lap 🏆

```bash
# Admire your complete setup
pvs   # Physical volumes
vgs   # Volume groups
lvs   # Logical volumes
df -h # The proof

# Test reboot persistence (optional but recommended)
reboot

# After reboot, check everything came back:
df -h  # All mounts should be there
lvs    # All LVs active
```

---

## Method 2: Extending Volumes (When Murphy's Law Strikes)

### Scenario
Database filled up at 2 AM. Management wants answers. You want more disk space.

### The Emergency Growth Procedure

**Step 1: Expand the Disk in OpenShift**

**Console** → **Virtualization** → **VirtualMachines** → `rhel9-lvm-vm`  
**Disks** tab → Click `data-disk` → **Edit**  
Change size: `100 GiB` → `150 GiB` → **Save**

> **Coffee break**: Wait for expansion to complete (usually quick)

**Step 2: Tell the VM About the New Space**

```bash
sudo -i

# Check current size
pvs  # Shows old size

# Rescan and resize
pvresize /dev/vdb

# Verify the magic worked
pvs    # Should show 150G now
vgs datavg  # ~60G free space available
```

**Step 3: Extend the Logical Volume**

```bash
# Grow db_data_lv by 40GB (60GB → 100GB)
lvextend -L +40G /dev/datavg/db_data_lv

# Or go crazy and use all free space
# lvextend -l +100%FREE /dev/datavg/db_data_lv

lvs  # Verify
```

**Step 4: Resize the Filesystem**

```bash
# For XFS (can only grow, never shrink - like my waistline)
xfs_growfs /data/db

# For ext4 (if you went that route)
# resize2fs /dev/datavg/db_data_lv

# Victory!
df -h /data/db  # Should show 100G now
```

> **Crisis averted**: Time to email management with the good news! ✅

---

## Method 3: Cloud-init Automation (For the Lazy... I Mean Efficient)

### When You Need to Deploy 50 VMs

**The Problem**: Repeating the same setup 50 times sounds like torture.  
**The Solution**: Automate it!

### Cloud-init Configuration

During VM creation, in **Scripts** → **Cloud-init**:

```yaml
#cloud-config
user: admin
password: redhat123
chpasswd: { expire: False }
ssh_pwauth: True

# Automated LVM setup
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
  - echo "LVM setup completed - I am inevitable" > /tmp/lvm-done.txt
```

**Limitations**: Cloud-init is great for simple setups. For complex scenarios, use Method 4.

---

## Common Operations (Your New Daily Routine)

### Add Disk to Existing VG

```bash
# After adding disk in OpenShift Console
sudo -i
lsblk  # Find new disk (let's say /dev/vde)

pvcreate /dev/vde
vgextend datavg /dev/vde  # Add to existing VG

vgs datavg  # More space! 🎉

# Now you can grow existing LVs or create new ones
lvextend -L +50G /dev/datavg/db_data_lv
xfs_growfs /data/db
```

### Create LVM Snapshot (Before That "Totally Safe" Upgrade)

```bash
sudo -i

# Create snapshot (10GB for changes)
lvcreate -L 10G -s -n db_backup_snap /dev/datavg/db_data_lv

# Do your risky thing...
# If it breaks: lvconvert --merge /dev/datavg/db_backup_snap
# If it works: lvremove /dev/datavg/db_backup_snap

# Mount snapshot for backup
mkdir /mnt/snapshot
mount -o ro /dev/datavg/db_backup_snap /mnt/snapshot
tar czf /backup/db.tar.gz /mnt/snapshot

# Cleanup
umount /mnt/snapshot
lvremove /dev/datavg/db_backup_snap
```

---

## Troubleshooting (When Things Go Wrong)

### "Device is already in use"

**Translation**: Disk has stuff on it.

```bash
# Nuke everything on the disk (⚠️ DANGER ZONE)
sudo wipefs -a /dev/vdb
sudo pvcreate /dev/vdb  # Try again
```

### "LV not found" After Reboot

**Translation**: LVM didn't wake up properly.

```bash
sudo vgchange -ay  # Wake up all VGs
sudo mount -a      # Mount everything
```

**Permanent fix**: Check `/etc/fstab` - use `/dev/mapper/vgname-lvname` paths

### "Insufficient free space"

**Translation**: You need more disk.

```bash
# Option 1: Expand existing disk in OpenShift → pvresize
# Option 2: Add new disk → vgextend

# Check what's available
sudo vgs datavg
```

### "XFS filesystem cannot be shrunk"

**Translation**: XFS doesn't do shrinking. It's a feature, not a bug!

**Solution**: Backup → Recreate smaller → Restore  
Or use ext4 if you need shrinking (but seriously, who shrinks storage?)

---

## Best Practices (Learn from My Mistakes)

### 1. Naming Conventions

**Good**:
- `datavg`, `logsvg`, `appvg` ✅
- `db_data_lv`, `app_logs_lv` ✅

**Bad**:
- `vg00`, `vg01` ❌ (what do these even mean?)
- `lv_1`, `volume_thingy` ❌ (future you will hate you)

### 2. Always Leave Free Space

**Rule of thumb**: Don't allocate 100% of VG space

**Why?**
- Snapshots need space
- Emergencies happen
- Your 2 AM self will thank you

**How much?** 10-20% free is reasonable

### 3. Use XFS Unless...

**Use XFS when**:
- Large files ✅
- Databases ✅
- High performance needed ✅
- Storage will only grow ✅

**Use ext4 when**:
- You might need to shrink (rare)
- General purpose use ✅

> **Hot take**: If you're still using ext3, we need to talk.

### 4. Separate VGs for Different Workloads

**Why separate VGs?**
- Independent expansion
- Better I/O isolation
- Easier troubleshooting
- Looks professional in meetings 😎

**Example**:
- `datavg` → Database files
- `logsvg` → Logs (logs are chatty)
- `appvg` → Application binaries

### 5. Backup Strategy

```bash
# Before major changes:
lvcreate -L 10G -s -n pre_upgrade_snap /dev/datavg/db_data_lv

# After confirming it works:
lvremove /dev/datavg/pre_upgrade_snap

# Document your LVM layout somewhere
# (Not on a sticky note that falls off)
```

### 6. Monitoring

**Set up alerts for**:
- VG free space < 20% ⚠️
- LV utilization > 80% ⚠️

**Regular checks**:
```bash
vgs  # Weekly
lvs  # Weekly
df -h  # Daily (or when paranoid)
```

---

## Quick Reference (Print This Out)

### LVM Command Cheat Sheet

```bash
# PV (Physical Volume) - The Raw Disk
pvcreate /dev/vdX      # Initialize disk for LVM
pvs                    # Quick view
pvdisplay              # Detailed view
pvresize /dev/vdX      # After expanding disk in OpenShift

# VG (Volume Group) - The Storage Pool
vgcreate vgname /dev/vdX    # Create VG
vgs                         # Quick view
vgdisplay                   # Detailed view
vgextend vgname /dev/vdX    # Add disk to VG

# LV (Logical Volume) - The Actual Usable Volume
lvcreate -L 10G -n lvname vgname     # Create LV
lvs                                   # Quick view
lvdisplay                             # Detailed view
lvextend -L +10G /dev/vg/lv          # Grow LV
lvremove /dev/vg/lv                  # Delete LV

# Filesystem Operations
mkfs.xfs /dev/vg/lv                  # Format XFS
mkfs.ext4 /dev/vg/lv                 # Format ext4
xfs_growfs /mountpoint               # Resize XFS
resize2fs /dev/vg/lv                 # Resize ext4

# Snapshots (Your "Undo" Button)
lvcreate -L 10G -s -n snapname /dev/vg/lv    # Create
lvremove /dev/vg/snapname                     # Remove
lvconvert --merge /dev/vg/snapname            # Rollback
```

---

## Production Example: PostgreSQL Database Server

### Requirements
- RHEL 9 VM for PostgreSQL
- Separate volumes for everything
- LVM for flexibility
- Must survive DBA's "creative" queries

### VM Configuration

**Disks**:
- Root: 30GB (OS)
- Data: 200GB (database files)
- WAL: 100GB (write-ahead logs, use fast storage)
- Backup: 300GB (because backups are important)
- Temp: 50GB (temp files and shame)

### LVM Layout

```bash
# Data VG (keep 20GB free for snapshots)
pvcreate /dev/vdb
vgcreate pgdata_vg /dev/vdb
lvcreate -L 150G -n pg_data_lv pgdata_vg
lvcreate -L 30G -n pg_index_lv pgdata_vg

# WAL VG (fast storage, keep 20GB free)
pvcreate /dev/vdc
vgcreate pgwal_vg /dev/vdc
lvcreate -L 80G -n pg_wal_lv pgwal_vg

# Backup VG (keep 50GB free)
pvcreate /dev/vdd
vgcreate pgbackup_vg /dev/vdd
lvcreate -L 250G -n pg_backup_lv pgbackup_vg

# Temp VG (keep 10GB free)
pvcreate /dev/vde
vgcreate pgtemp_vg /dev/vde
lvcreate -L 40G -n pg_temp_lv pgtemp_vg

# Format everything with XFS (PostgreSQL loves XFS)
mkfs.xfs /dev/pgdata_vg/pg_data_lv
mkfs.xfs /dev/pgdata_vg/pg_index_lv
mkfs.xfs /dev/pgwal_vg/pg_wal_lv
mkfs.xfs /dev/pgbackup_vg/pg_backup_lv
mkfs.xfs /dev/pgtemp_vg/pg_temp_lv

# Create mount points
mkdir -p /var/lib/pgsql/{data,index,wal}
mkdir -p /backup/postgresql
mkdir -p /tmp/pgsql

# Mount everything
mount /dev/pgdata_vg/pg_data_lv /var/lib/pgsql/data
mount /dev/pgdata_vg/pg_index_lv /var/lib/pgsql/index
mount /dev/pgwal_vg/pg_wal_lv /var/lib/pgsql/wal
mount /dev/pgbackup_vg/pg_backup_lv /backup/postgresql
mount /dev/pgtemp_vg/pg_temp_lv /tmp/pgsql

# Update fstab
cat >> /etc/fstab << EOF
/dev/pgdata_vg/pg_data_lv     /var/lib/pgsql/data   xfs  defaults  0  0
/dev/pgdata_vg/pg_index_lv    /var/lib/pgsql/index  xfs  defaults  0  0
/dev/pgwal_vg/pg_wal_lv       /var/lib/pgsql/wal    xfs  defaults  0  0
/dev/pgbackup_vg/pg_backup_lv /backup/postgresql    xfs  defaults  0  0
/dev/pgtemp_vg/pg_temp_lv     /tmp/pgsql            xfs  defaults  0  0
EOF

# Set permissions (PostgreSQL is picky)
chown -R postgres:postgres /var/lib/pgsql /tmp/pgsql
chmod 700 /var/lib/pgsql/data
```

### Backup Strategy (Before DBA Does Something Interesting)

```bash
# Create snapshot
lvcreate -L 10G -s -n pg_backup_snap /dev/pgdata_vg/pg_data_lv

# Mount and backup
mkdir /mnt/pg_snapshot
mount -o ro /dev/pgdata_vg/pg_backup_snap /mnt/pg_snapshot
tar czf /backup/postgresql/pgdata-$(date +%Y%m%d).tar.gz /mnt/pg_snapshot

# Cleanup
umount /mnt/pg_snapshot
lvremove -y /dev/pgdata_vg/pg_backup_snap
```

---

## Summary Checklist (Don't Skip This!)

### Initial Setup ✅
- [ ] Create VM with multiple blank disks
- [ ] Verify disks visible (`lsblk`)
- [ ] Create PVs on each disk
- [ ] Create VGs (separate for different purposes)
- [ ] Create LVs with appropriate sizes
- [ ] Format with XFS (or ext4 if you must)
- [ ] Create mount points
- [ ] Mount filesystems
- [ ] Update `/etc/fstab`
- [ ] Test reboot (seriously, test it)

### Expansion Procedure ✅
- [ ] Expand PVC in OpenShift Console
- [ ] Resize PV: `pvresize /dev/vdX`
- [ ] Extend LV: `lvextend`
- [ ] Resize filesystem: `xfs_growfs` or `resize2fs`
- [ ] Verify: `df -h`
- [ ] Update documentation
- [ ] Tell your manager you're a hero

### Monitoring ✅
- [ ] Check VG free space weekly
- [ ] Monitor LV usage
- [ ] Review logs for LVM errors
- [ ] Test snapshots quarterly
- [ ] Document everything
- [ ] Keep coffee stocked ☕

---

## Final Thoughts

**What You've Learned**:
- LVM makes storage flexible (unlike your morning routine)
- Always leave free space in VGs
- XFS is your friend for databases
- Cloud-init can automate simple setups
- Snapshots are your "undo" button
- Documentation saves careers

**What You Should Do Next**:
1. Practice on a test VM (break things safely)
2. Document your LVM layouts
3. Set up monitoring alerts
4. Test your backup/restore procedures
5. Teach someone else (best way to learn)

**Remember**: The best time to configure LVM was during initial deployment. The second best time is now.

---

## Additional Resources

**Official Documentation** (for when you want the boring version):
- Red Hat LVM Guide: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/
- OpenShift Virtualization: https://docs.openshift.com/container-platform/latest/virt/storage/

**Pro Tips**:
- Join Red Hat forums
- Practice in lab environments
- Keep a personal runbook
- Learn from failures (you'll have some)

---

**Last Updated**: February 2025  
**Target Audience**: OpenShift Admins who like their storage flexible  
**Difficulty**: Intermediate (you got this!)

*"In LVM we trust, in snapshots we prepare, in documentation we survive."* 📚

---

**End of Guide** 🎉

*Questions? Found a typo? Want to share your LVM horror story? Open an issue!*
