# OpenShift Virtualization - Daily Production Operations Lab Guide

## Overview
This guide covers the **TOP 5 daily operational tasks** performed by OpenShift Virtualization administrators in production environments. All tasks focus on the **Web Console (GUI)** and include real-world scenarios.

**Prerequisites:**
- Access to OpenShift Web Console with appropriate RBAC permissions
- Existing OpenShift Virtualization operator installed
- At least one project/namespace available

---

## Task 1: VM Creation and Lifecycle Management

### Scenario
Development team requests a new RHEL 9 VM for application testing. They need 4 vCPUs, 8GB RAM, and 50GB disk. The VM must be created from a template and should be accessible within 15 minutes.

### Objective
Create a production-ready VM using best practices, start it, verify operation, and understand lifecycle states (stop, restart, migrate).

### Steps

#### 1.1 Create VM from Template

1. Navigate to **Virtualization** → **VirtualMachines** in the left sidebar
2. Click **Create** → **From template**
3. Select **Template**: `rhel9-server-small` (or closest available RHEL 9 template)
4. Configure basic details:
   - **Name**: `dev-app-vm-01`
   - **Project**: Select your project (e.g., `virtualization-lab`)
5. Click **Customize VirtualMachine**
6. In **Overview** tab:
   - Set **CPU cores**: `4`
   - Set **Memory**: `8 GiB`
7. Go to **Disks** tab:
   - Click the rootdisk entry
   - Click **Edit**
   - Change **Size** to `50 GiB`
   - Verify **StorageClass** is appropriate (e.g., `ocs-storagecluster-ceph-rbd`)
   - Click **Save**
8. Go to **Network Interfaces** tab:
   - Verify default interface exists (usually `masquerade` or `bridge`)
   - Note the network configuration for later access
9. Click **Create VirtualMachine**

#### 1.2 Start and Verify VM

1. On the VM details page, click **Actions** → **Start**
2. Wait for status to change from `Starting` to `Running` (watch the status badge)
3. Click the **Console** tab to verify boot progress
4. Wait for login prompt (confirms successful boot)

#### 1.3 Lifecycle Operations

**Stop VM:**
1. Click **Actions** → **Stop**
2. Confirm the action
3. Wait for status: `Stopped`

**Restart VM:**
1. Click **Actions** → **Restart**
2. Confirm the action
3. Verify status returns to `Running`

**Live Migration (if multi-node cluster):**
1. Note the current **Node** in the **Details** tab
2. Click **Actions** → **Migrate**
3. Confirm migration
4. Watch the **Node** field change (VM moves without downtime)

### Expected Result
- VM appears in the VirtualMachines list with `Running` status
- Console shows RHEL 9 login prompt
- VM can be stopped, started, and migrated without data loss
- Resource allocation matches requirements (verify in **Overview** tab: 4 CPUs, 8 GB RAM, 50 GB disk)

### Validation Checklist
- [ ] VM created successfully
- [ ] Status shows `Running`
- [ ] Console accessible and shows login prompt
- [ ] CPU/Memory matches requested specs
- [ ] Disk size is 50 GiB
- [ ] Lifecycle operations (stop/start/restart) work correctly

---

## Task 2: Storage Management - Adding and Expanding Disks

### Scenario
Database team needs to add a 100GB data disk to an existing VM for PostgreSQL storage. Later, they request expansion to 150GB due to growth.

### Objective
Add additional storage to running VM (hot-plug), expand existing disk, and verify changes without VM downtime.

### Steps

#### 2.1 Add New Disk to Running VM

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on your VM (e.g., `dev-app-vm-01`)
3. Go to **Disks** tab
4. Click **Add disk**
5. Configure new disk:
   - **Source**: `Blank (creates PVC)`
   - **Name**: `data-disk-01`
   - **Size**: `100 GiB`
   - **Type**: `Disk` (not CD-ROM)
   - **Interface**: `virtio` (best performance)
   - **StorageClass**: Select appropriate class (e.g., `ocs-storagecluster-ceph-rbd`)
6. Click **Add**
7. Wait for disk to show `Ready` status in the disks list

#### 2.2 Verify Disk in VM Console

1. Click **Console** tab
2. Login to the VM
3. Run: `lsblk` to see new disk (should appear as `/dev/vdb` or similar)
4. Expected output shows new 100GB disk unmounted and unformatted

#### 2.3 Expand Existing Disk

1. Return to **Disks** tab
2. Click on `data-disk-01` (the disk you just added)
3. Click **Edit**
4. Change **Size** from `100 GiB` to `150 GiB`
5. Click **Save**
6. Wait for expansion to complete (status remains `Ready`)

**Note**: For rootdisk expansion, additional steps inside the VM OS may be required (resize partition, extend filesystem).

#### 2.4 Verify Expansion

1. In VM console, run: `lsblk` again
2. Verify disk now shows 150GB
3. If disk was formatted/mounted, you may need to expand the filesystem:
   ```bash
   # Example for ext4 (if disk was already in use)
   sudo resize2fs /dev/vdb1
   ```

### Expected Result
- New 100GB disk appears in VM without restart (hot-plug successful)
- Disk visible in `lsblk` output as unmounted block device
- Disk successfully expanded to 150GB
- No VM downtime during either operation
- PVC shows correct size in **Storage** → **PersistentVolumeClaims**

### Validation Checklist
- [ ] New disk added and shows `Ready` status
- [ ] Disk visible in VM OS (`lsblk` shows new device)
- [ ] Disk expanded from 100GB to 150GB successfully
- [ ] VM remained `Running` during both operations
- [ ] PVC size matches disk size (verify in Storage section)

---

## Task 3: Networking - Static IP Configuration and External Access

### Scenario
Application team needs their VM to have a static IP address and be accessible from external networks (outside OpenShift cluster) for client connections.

### Objective
Configure VM with static IP using cloud-init, set up external access via NodePort service or route, and verify connectivity.

### Steps

#### 3.1 Configure Static IP with Cloud-init

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create** → **From template** (or edit existing VM)
3. Select RHEL or cloud image template
4. Click **Customize VirtualMachine**
5. Go to **Scripts** tab
6. Enable **Cloud-init**
7. Click **Edit** on cloud-init
8. In the cloud-init configuration, add network configuration:
   ```yaml
   #cloud-config
   user: admin
   password: redhat
   chpasswd: { expire: False }
   ssh_pwauth: True
   network:
     version: 2
     ethernets:
       eth0:
         addresses:
           - 10.0.2.100/24
         gateway4: 10.0.2.1
         nameservers:
           addresses:
             - 8.8.8.8
             - 8.8.4.4
   ```
   **Adjust IP, gateway, and subnet to match your pod network range**

9. Click **Save**
10. Create/restart the VM for cloud-init to apply

#### 3.2 Verify Static IP

1. Go to **Console** tab
2. Login to VM
3. Run: `ip addr show` or `nmcli device show`
4. Verify the configured static IP is assigned

#### 3.3 Expose VM with NodePort Service

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on your VM
3. Go to **Network Interfaces** tab
4. Note the interface name and network
5. Go to **Networking** → **Services** in the main menu
6. Click **Create Service**
7. Switch to **YAML view**
8. Paste the following (adjust names):
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: vm-external-access
     namespace: virtualization-lab
   spec:
     type: NodePort
     selector:
       vm.kubevirt.io/name: dev-app-vm-01
     ports:
       - protocol: TCP
         port: 22
         targetPort: 22
         nodePort: 30022  # Optional: specify port 30000-32767
   ```
9. Click **Create**

#### 3.4 Find External Access Details

1. Go to **Networking** → **Services**
2. Click on `vm-external-access`
3. In **Details** tab, note:
   - **Node Port**: e.g., `30022`
   - **Cluster IP**: for internal access
4. To access from outside:
   - Get any OpenShift node IP: `oc get nodes -o wide` (or check in **Compute** → **Nodes**)
   - SSH access: `ssh admin@<node-ip> -p 30022`

#### 3.5 Alternative: LoadBalancer (if available)

If your cluster has LoadBalancer support:
1. Edit the service YAML
2. Change `type: NodePort` to `type: LoadBalancer`
3. Save and wait for external IP assignment
4. Access VM using the LoadBalancer IP directly

### Expected Result
- VM has static IP configured and visible in `ip addr` output
- NodePort service created successfully
- VM accessible via SSH from external network using `<node-ip>:<nodeport>`
- Connection is stable and persistent
- Service shows in Services list with correct port mappings

### Validation Checklist
- [ ] Static IP configured via cloud-init
- [ ] IP address persists after VM reboot
- [ ] NodePort service created
- [ ] External SSH access works: `ssh admin@<node-ip> -p 30022`
- [ ] Service selector correctly matches VM
- [ ] No network conflicts or IP collisions

---

## Task 4: Snapshots, Backup, and Restore Operations

### Scenario
Before a major application upgrade, operations team needs to create a VM snapshot for quick rollback capability. After testing, they either restore to snapshot or create a backup for long-term retention.

### Objective
Create VM snapshot, restore from snapshot, and export VM for backup purposes.

### Steps

#### 4.1 Create VM Snapshot

**Prerequisite**: VM must be stopped or storage class must support live snapshots (CSI with snapshot support)

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on your VM
3. Go to **Snapshots** tab
4. Click **Take snapshot**
5. Configure snapshot:
   - **Name**: `pre-upgrade-snapshot`
   - **Description**: `Snapshot before application v2.0 upgrade`
   - **Ensure that snapshots are stopped**: Check this box (stops VM temporarily if storage doesn't support live snapshots)
6. Click **Save**
7. Wait for snapshot status to show `Ready` (may take 1-5 minutes depending on disk size)

#### 4.2 Make Changes to VM

1. Start the VM if it was stopped
2. Login via console
3. Make some changes (e.g., create file, install package):
   ```bash
   sudo touch /tmp/after-snapshot.txt
   echo "Changes after snapshot" | sudo tee /tmp/after-snapshot.txt
   ```
4. Stop the VM: **Actions** → **Stop**

#### 4.3 Restore from Snapshot

1. Go to **Snapshots** tab
2. Click on `pre-upgrade-snapshot`
3. Click **Actions** → **Restore VirtualMachine**
4. Review the warning message
5. Click **Restore**
6. Wait for restore to complete (status shows `Ready`)
7. Start the VM: **Actions** → **Start**

#### 4.4 Verify Restoration

1. Open **Console** tab
2. Login to VM
3. Check if changes are reverted:
   ```bash
   ls -l /tmp/after-snapshot.txt
   # File should NOT exist (reverted to pre-snapshot state)
   ```

#### 4.5 Export VM for Backup

**Note**: Export functionality varies by OpenShift version. Alternative approach:

1. Go to **Snapshots** tab
2. Create a new snapshot for backup: `backup-2025-02-16`
3. Note the underlying PVC snapshot names in **Storage** → **VolumeSnapshots**
4. For long-term backup, document:
   - VM configuration (export YAML from **YAML** tab)
   - Snapshot names
   - PVC details

**Manual Export Method:**
1. Click VM name → **YAML** tab
2. Click **Download** to save VM definition
3. Store YAML in version control or backup system
4. PVC snapshots are stored in your storage backend automatically

### Expected Result
- Snapshot created successfully and shows `Ready` status
- Changes made after snapshot are lost after restore
- VM boots correctly after restoration
- Snapshot remains available for future restores
- VM configuration can be exported as YAML for disaster recovery

### Validation Checklist
- [ ] Snapshot created with `Ready` status
- [ ] Snapshot listed in Snapshots tab with correct timestamp
- [ ] Restore operation completed successfully
- [ ] Post-snapshot changes are gone after restore
- [ ] VM boots and operates normally after restore
- [ ] VM YAML exported for backup documentation
- [ ] Volume snapshots visible in Storage section

---

## Task 5: Monitoring, Troubleshooting, and Resource Tuning

### Scenario
A VM is experiencing performance issues. Users report slow response times. You need to monitor resource usage, identify bottlenecks, and tune resources accordingly.

### Objective
Monitor VM metrics, identify resource constraints, adjust CPU/memory allocation, and resolve performance issues using the Web Console.

### Steps

#### 5.1 Monitor VM Metrics

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on your VM
3. Go to **Metrics** tab
4. Review key metrics:
   - **CPU Usage**: Check if consistently at 100%
   - **Memory Usage**: Check if at or near limit
   - **Filesystem Usage**: Check disk space
   - **Network I/O**: Check for saturation
5. Adjust time range using the dropdown (Last 1 hour, 6 hours, 24 hours, etc.)
6. Identify bottlenecks:
   - CPU at 100% = CPU bottleneck
   - Memory near limit + swap usage = Memory bottleneck
   - High iowait = Disk bottleneck

#### 5.2 Check VM Events and Logs

1. Go to **Events** tab
2. Look for warnings or errors:
   - `FailedScheduling`
   - `OOMKilled` (out of memory)
   - `EvictionThresholdMet`
3. Note any recurring issues
4. Go to **Console** tab → **Serial console** for low-level boot/kernel logs
5. Switch to **VNC console** for graphical troubleshooting

#### 5.3 Increase CPU Resources (Hot-plug)

**For Running VM:**
1. Go to **Overview** tab or **Details** tab
2. In the **Details** section, find **CPU | Memory**
3. Click **Edit** (pencil icon)
4. Increase **CPU cores** from `4` to `6`
5. Click **Save**
6. Verify change: CPU count updates immediately (hot-plug supported)

**Note**: Some OS configurations may require guest OS recognition. Check console for new CPUs.

#### 5.4 Increase Memory (Requires Restart)

**Warning**: Memory changes typically require VM restart.

1. Stop the VM: **Actions** → **Stop**
2. Go to **Details** tab
3. Click **Edit** on CPU | Memory section
4. Increase **Memory** from `8 GiB` to `16 GiB`
5. Click **Save**
6. Start the VM: **Actions** → **Start**
7. Verify in **Overview** tab: Memory shows `16 GiB`

#### 5.5 Adjust Disk Performance (Change Storage Class)

**For new disks only** (existing disks cannot change storage class):

1. Go to **Disks** tab
2. Click **Add disk** for additional disk
3. Select a higher-performance **StorageClass**:
   - For better performance: `ocs-storagecluster-ceph-rbd` with SSD backend
   - For lower cost: `ocs-storagecluster-cephfs`
4. Move application data to the new disk if needed

#### 5.6 Enable Dedicated CPU (CPU Pinning)

**For latency-sensitive workloads:**

1. Stop the VM
2. Click **YAML** tab
3. Find `spec.template.spec.domain.cpu` section
4. Add dedicated CPU configuration:
   ```yaml
   spec:
     template:
       spec:
         domain:
           cpu:
             dedicatedCpuPlacement: true
             cores: 6
   ```
5. Click **Save**
6. Start VM
7. Verify: VM scheduled on nodes with dedicated CPU resources

#### 5.7 Monitor Improvement

1. Return to **Metrics** tab after 10-15 minutes
2. Compare current metrics to earlier baselines
3. Verify:
   - CPU usage decreased from 100% to acceptable range (<80%)
   - Memory usage within safe limits (<85%)
   - Application response improved
4. Document changes in VM annotations or external documentation

### Expected Result
- Metrics clearly show resource usage patterns
- Bottlenecks identified (CPU, memory, or disk)
- Resource adjustments applied successfully
- CPU hot-plugged without VM restart
- Memory increased after controlled restart
- Performance improved and metrics show lower resource pressure
- No errors in Events tab after changes

### Validation Checklist
- [ ] Metrics accessible and showing accurate data
- [ ] Resource bottleneck identified correctly
- [ ] CPU increased and recognized by VM
- [ ] Memory increased successfully
- [ ] VM performance improved (verify in application)
- [ ] No new errors in Events tab
- [ ] Changes documented for future reference
- [ ] Metrics show utilization below 80% after tuning

---

## Best Practices Summary

### Production Safety
- **Always create snapshots before major changes** (upgrades, configuration changes)
- **Test resource changes in non-production first** when possible
- **Use RBAC properly**: Don't grant cluster-admin unless absolutely necessary
- **Monitor events tab** after any operation for errors
- **Document changes** in VM annotations or external CMDB

### Resource Management
- **CPU**: Hot-plug supported for most VMs (increase without downtime)
- **Memory**: Requires restart in most cases (plan maintenance window)
- **Disk**: Can add new disks hot, expansion depends on storage class
- **Networking**: Plan IP addressing carefully to avoid conflicts

### Troubleshooting Quick Reference
- **VM won't start**: Check Events tab, verify PVC bound, check resource quotas
- **Console not accessible**: Try Serial Console, check VM networking
- **Performance issues**: Check Metrics tab, review resource requests/limits
- **Snapshot fails**: Verify storage class supports snapshots, check CSI driver
- **Live migration fails**: Check node resources, network connectivity between nodes

### Daily Checklist for Production
1. Review VM status dashboard for any `CrashLoopBackOff` or `Error` states
2. Check storage capacity trends (expand before 80% full)
3. Review resource utilization metrics for optimization opportunities
4. Verify snapshots are completing successfully
5. Monitor Events for recurring warnings or errors

---

## Additional Resources

### OpenShift Documentation
- **Virtualization Guide**: https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html
- **VM Management**: https://docs.openshift.com/container-platform/latest/virt/virtual_machines/virt-create-vms.html
- **Storage for VMs**: https://docs.openshift.com/container-platform/latest/virt/storage/virt-storage-defaults.html

### Common CLI Commands (for reference)
While this guide focuses on GUI, these CLI commands are useful for automation:
```bash
# List VMs
oc get vms -n <namespace>

# Get VM status
oc get vmi -n <namespace>

# Start VM
virtctl start <vm-name> -n <namespace>

# Access console
virtctl console <vm-name> -n <namespace>

# Create snapshot
virtctl snapshot create <vm-name> <snapshot-name> -n <namespace>
```

---

## Lab Progression

**Recommended Practice Order:**
1. **Day 1-2**: Task 1 (VM Lifecycle) - Master creation, start/stop, migration
2. **Day 3-4**: Task 2 (Storage) - Practice disk operations until confident
3. **Day 5-6**: Task 3 (Networking) - Set up external access patterns
4. **Day 7-8**: Task 4 (Snapshots) - Critical for production safety
5. **Day 9-10**: Task 5 (Monitoring) - Ties everything together

**Advanced Practice:**
- Combine tasks: Create VM → Add disk → Configure static IP → Take snapshot → Tune resources
- Simulate failure scenarios: Fill disk, exhaust memory, break network config
- Practice rollback procedures: Restore from snapshots, recover from mistakes

---

## Notes
- Screenshots vary by OpenShift version (4.12, 4.13, 4.14, 4.15, 4.16+)
- Storage class names depend on your storage provider (ODF, NetApp, NFS, etc.)
- Network configuration varies by CNI plugin (OVN-Kubernetes, OpenShift SDN)
- Some features require specific entitlements or operator versions

**Lab Environment Assumptions:**
- OpenShift 4.13+ with OpenShift Virtualization operator installed
- At least one storage class with RWX and snapshot support
- Multi-node cluster (for testing live migration)
- RBAC permissions: edit or admin role in target namespace

---

**End of Lab Guide**

*Last Updated: February 2025*
*Target Audience: OpenShift Virtualization Administrators*
*Skill Level: Intermediate to Advanced*
