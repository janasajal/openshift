================================================================================
        OPENSHIFT VIRTUALIZATION -- DAILY PRODUCTION OPERATIONS
        Updated : February 2025
================================================================================

OVERVIEW
--------

  Top 5 daily operational tasks for OpenShift Virtualization admins.
  All tasks use the Web Console (GUI).

  Prerequisites:
    - OpenShift Web Console access with RBAC permissions
    - OpenShift Virtualization operator installed
    - At least one project/namespace available
    - OpenShift 4.13+ recommended


================================================================================
TASK 1 -- VM CREATION AND LIFECYCLE MANAGEMENT
================================================================================

Scenario: Dev team needs a new RHEL 9 VM. 4 vCPUs, 8GB RAM, 50GB disk.
          Must be ready within 15 minutes.

-- Create VM from Template --

  1. Virtualization --> VirtualMachines --> Create --> From template
  2. Select: rhel9-server-small (or closest RHEL 9 template)
  3. Name: dev-app-vm-01
     Project: virtualization-lab
  4. Click: Customize VirtualMachine
  5. Overview tab:
       CPU cores : 4
       Memory    : 8 GiB
  6. Disks tab:
       Click rootdisk --> Edit
       Size          : 50 GiB
       StorageClass  : ocs-storagecluster-ceph-rbd (or your class)
       Save
  7. Network Interfaces tab:
       Verify default interface exists (masquerade or bridge)
  8. Click: Create VirtualMachine


-- Start and Verify --

  Actions --> Start
  Wait for: Starting --> Running
  Console tab --> verify RHEL 9 login prompt appears


-- Lifecycle Operations --

  Stop:
    Actions --> Stop --> Confirm
    Wait for: Stopped

  Restart:
    Actions --> Restart --> Confirm
    Wait for: Running

  Live Migration (multi-node clusters only):
    Details tab --> note current Node
    Actions --> Migrate --> Confirm
    Watch Node field change (zero downtime)


VALIDATION
----------

  [ ] VM shows Running status
  [ ] Console shows login prompt
  [ ] Overview confirms: 4 CPU, 8GB RAM, 50GB disk
  [ ] Stop / Start / Restart all work without data loss


================================================================================
TASK 2 -- STORAGE MANAGEMENT (ADD & EXPAND DISKS)
================================================================================

Scenario: DB team needs a 100GB data disk for PostgreSQL.
          Later they need it expanded to 150GB. No downtime allowed.

-- Add New Disk to Running VM (hot-plug) --

  1. Virtualization --> VirtualMachines --> dev-app-vm-01
  2. Disks tab --> Add disk
  3. Configure:
       Source      : Blank (creates PVC)
       Name        : data-disk-01
       Size        : 100 GiB
       Type        : Disk (not CD-ROM)
       Interface   : virtio (best performance)
       StorageClass: ocs-storagecluster-ceph-rbd
  4. Click Add
  5. Wait for disk to show Ready status


-- Verify in VM Console --

  Console tab --> login --> run:
    $ lsblk
  New 100GB disk appears as /dev/vdb (unmounted, unformatted)


-- Expand Existing Disk to 150GB --

  Disks tab --> click data-disk-01 --> Edit
  Change Size: 100 GiB --> 150 GiB
  Save
  Wait for status: Ready

  If disk was already formatted/mounted, extend filesystem inside VM:
    $ sudo resize2fs /dev/vdb1


-- Verify Expansion --

  Console --> run:
    $ lsblk    (disk now shows 150GB)

  Also check: Storage --> PersistentVolumeClaims --> confirm PVC size


VALIDATION
----------

  [ ] New disk added, shows Ready
  [ ] lsblk shows new device in VM OS
  [ ] Disk expanded 100GB --> 150GB
  [ ] VM stayed Running during both operations
  [ ] PVC size matches disk size


================================================================================
TASK 3 -- NETWORKING (STATIC IP + EXTERNAL ACCESS)
================================================================================

Scenario: App team needs VM with static IP accessible from outside the cluster.

-- Configure Static IP with Cloud-init --

  1. Virtualization --> VirtualMachines --> Create --> From template
     (or edit existing VM -- must be Stopped)
  2. Customize VirtualMachine --> Scripts tab
  3. Enable Cloud-init --> Edit
  4. Paste cloud-init config (adjust IP/gateway/subnet for your network):

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

  5. Save --> Create / Restart VM for cloud-init to apply


-- Verify Static IP --

  Console tab --> login --> run:
    $ ip addr show
    $ nmcli device show
  Confirm configured IP is assigned.


-- Expose VM via NodePort Service --

  1. Networking --> Services --> Create Service --> YAML view
  2. Paste and adjust:

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
           nodePort: 30022    # 30000-32767 range

  3. Click Create


-- Find External Access Details --

  Networking --> Services --> vm-external-access --> Details tab
    Note: NodePort (e.g., 30022) and Cluster IP

  Get a node IP:
    oc get nodes -o wide   (or Compute --> Nodes in Web Console)

  SSH from outside:
    $ ssh admin@<node-ip> -p 30022


-- Alternative: LoadBalancer (if supported) --

  Edit service YAML:
    type: NodePort  -->  type: LoadBalancer
  Save, wait for external IP, connect directly via LoadBalancer IP.


VALIDATION
----------

  [ ] Static IP set and visible in ip addr
  [ ] IP persists after VM reboot
  [ ] NodePort service created
  [ ] External SSH works: ssh admin@<node-ip> -p 30022
  [ ] No IP conflicts


================================================================================
TASK 4 -- SNAPSHOTS, BACKUP, AND RESTORE
================================================================================

Scenario: Before a major app upgrade, team needs a snapshot for quick rollback.

-- Create Snapshot --

  Prerequisite: VM stopped, OR storage class supports live snapshots (CSI).

  1. Virtualization --> VirtualMachines --> your VM
  2. Snapshots tab --> Take snapshot
  3. Configure:
       Name       : pre-upgrade-snapshot
       Description: Snapshot before application v2.0 upgrade
       Check box  : Ensure snapshots are stopped (if no live snapshot support)
  4. Save
  5. Wait for status: Ready (1-5 minutes depending on disk size)


-- Make Changes Post-Snapshot --

  Start VM if stopped --> Console --> login:
    $ sudo touch /tmp/after-snapshot.txt
    $ echo "Changes after snapshot" | sudo tee /tmp/after-snapshot.txt

  Stop VM: Actions --> Stop


-- Restore from Snapshot --

  Snapshots tab --> pre-upgrade-snapshot
  Actions --> Restore VirtualMachine
  Read warning --> click Restore
  Wait for: Ready
  Start VM: Actions --> Start


-- Verify Restore --

  Console --> login:
    $ ls -l /tmp/after-snapshot.txt
    File should NOT exist. Changes reverted. Restore confirmed.


-- Export VM for Backup --

  Option 1 -- Export YAML definition:
    VM --> YAML tab --> Download
    Store in version control or backup system.

  Option 2 -- Snapshot for long-term backup:
    Snapshots tab --> Take snapshot --> name: backup-2025-02-16
    View underlying PVC snapshots: Storage --> VolumeSnapshots


VALIDATION
----------

  [ ] Snapshot created, shows Ready
  [ ] Snapshot listed with correct timestamp
  [ ] Restore completed, post-snapshot changes gone
  [ ] VM boots normally after restore
  [ ] VM YAML exported
  [ ] VolumeSnapshots visible in Storage section


================================================================================
TASK 5 -- MONITORING, TROUBLESHOOTING, AND RESOURCE TUNING
================================================================================

Scenario: VM is slow. Users report poor response times.
          Identify the bottleneck and tune resources.

-- Monitor VM Metrics --

  Virtualization --> VirtualMachines --> your VM --> Metrics tab

  Review:
    CPU Usage       --> 100% = CPU bottleneck
    Memory Usage    --> near limit + swap = Memory bottleneck
    Filesystem      --> check disk space
    Network I/O     --> saturation?

  Adjust time range: Last 1 hour / 6 hours / 24 hours


-- Check Events and Logs --

  Events tab --> look for:
    FailedScheduling
    OOMKilled
    EvictionThresholdMet

  Console tab --> Serial console (kernel/boot logs)
  Console tab --> VNC console (graphical troubleshooting)


-- Increase CPU (Hot-plug, no restart needed) --

  Overview or Details tab --> CPU | Memory --> Edit (pencil icon)
  CPU cores: 4 --> 6
  Save
  Change takes effect immediately. Verify in VM console if needed.


-- Increase Memory (Requires Restart) --

  Actions --> Stop
  Details tab --> CPU | Memory --> Edit
  Memory: 8 GiB --> 16 GiB
  Save
  Actions --> Start
  Verify: Overview tab shows 16 GiB


-- Improve Disk Performance --

  Existing disks cannot change storage class.
  Add a NEW disk with a higher-performance StorageClass:
    Disks tab --> Add disk
    StorageClass: ocs-storagecluster-ceph-rbd (SSD backend preferred)
  Move app data to the new disk.


-- Enable Dedicated CPU Pinning (latency-sensitive workloads) --

  Stop VM --> YAML tab --> edit spec.template.spec.domain.cpu:

    spec:
      template:
        spec:
          domain:
            cpu:
              dedicatedCpuPlacement: true
              cores: 6

  Save --> Start VM
  VM scheduled only on nodes with dedicated CPU resources.


-- Monitor Improvement --

  After 10-15 minutes: Metrics tab
  Compare to earlier baseline.
  Targets:
    CPU usage    < 80%
    Memory usage < 85%
    No new errors in Events tab
  Document changes.


VALIDATION
----------

  [ ] Metrics accessible and accurate
  [ ] Bottleneck identified (CPU / Memory / Disk)
  [ ] CPU increased, recognized by VM
  [ ] Memory increased after restart
  [ ] Performance improved, utilization below 80%
  [ ] No new errors in Events tab
  [ ] Changes documented


================================================================================
BEST PRACTICES
================================================================================

PRODUCTION SAFETY
-----------------

  Always snapshot before major changes (upgrades, config changes)
  Test resource changes in non-prod first
  Use RBAC properly. No cluster-admin unless absolutely required.
  Monitor Events tab after every operation
  Document all changes in VM annotations or CMDB


RESOURCE MANAGEMENT
-------------------

  CPU    --> Hot-plug supported. Can increase without downtime.
  Memory --> Requires restart. Plan a maintenance window.
  Disk   --> Can add new disks hot. Expansion depends on storage class.
  Network--> Plan IP addressing carefully. Avoid conflicts.


TROUBLESHOOTING QUICK REFERENCE
--------------------------------

  VM won't start         --> Events tab, verify PVC bound, check resource quotas
  Console not accessible --> Try Serial Console, check VM networking
  Performance issues     --> Metrics tab, review requests/limits
  Snapshot fails         --> Verify StorageClass supports snapshots, check CSI driver
  Live migration fails   --> Check node resources, network between nodes


DAILY CHECKLIST (PRODUCTION)
-----------------------------

  1. Review VM status dashboard -- any Error or CrashLoopBackOff states?
  2. Check storage capacity trends -- expand before 80% full
  3. Review Metrics for optimization opportunities
  4. Verify snapshots completing successfully
  5. Monitor Events for recurring warnings


================================================================================
CLI QUICK REFERENCE (for automation)
================================================================================

  $ oc get vms -n <namespace>
  $ oc get vmi -n <namespace>

  $ virtctl start <vm-name> -n <namespace>
  $ virtctl console <vm-name> -n <namespace>
  $ virtctl snapshot create <vm-name> <snapshot-name> -n <namespace>


================================================================================
LAB PROGRESSION
================================================================================

  Day 1-2  : Task 1 -- VM lifecycle (create, start, stop, migrate)
  Day 3-4  : Task 2 -- Storage (add disk, hot-plug, expand)
  Day 5-6  : Task 3 -- Networking (static IP, NodePort, external access)
  Day 7-8  : Task 4 -- Snapshots (create, restore, export)
  Day 9-10 : Task 5 -- Monitoring (metrics, tune, verify)

  Advanced practice:
    Create VM --> Add disk --> Static IP --> Snapshot --> Tune resources
    Simulate failures: fill disk, exhaust memory, break network config
    Practice rollback: restore from snapshots, recover from mistakes


ENVIRONMENT NOTES
-----------------

  UI screenshots vary by version (4.12, 4.13, 4.14, 4.15, 4.16+)
  StorageClass names depend on provider (ODF, NetApp, NFS, etc.)
  Network config varies by CNI (OVN-Kubernetes, OpenShift SDN)
  Some features need specific entitlements or operator versions

  Lab assumptions:
    OpenShift 4.13+ with Virtualization operator installed
    At least one StorageClass with RWX + snapshot support
    Multi-node cluster (for live migration testing)
    RBAC: edit or admin role in target namespace

================================================================================
  END OF GUIDE
================================================================================
