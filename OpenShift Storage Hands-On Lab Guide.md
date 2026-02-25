# OpenShift Storage Hands-On Lab Guide
### Red Hat OpenShift Virtualization Administration — v4.18

> **Environment:** Red Hat OpenShift Virtualization Administration Rapid Track v4.18  
> **Cluster:** 3-node (master01, master02, master03) — all running as control-plane + worker  
> **OCP Version:** v1.31.6

---

## Table of Contents

1. [Prerequisites & Environment Check](#1-prerequisites--environment-check)
2. [Storage Classes Overview](#2-storage-classes-overview)
3. [Persistent Volumes & Claims](#3-persistent-volumes--claims)
4. [Creating & Using PVCs with Pods](#4-creating--using-pvcs-with-pods)
5. [Storage Persistence](#5-storage-persistence)
6. [PVC Expansion](#6-pvc-expansion)
7. [Volume Snapshots & Restore](#7-volume-snapshots--restore)
8. [Object Storage with NooBaa](#8-object-storage-with-noobaa)
9. [VM Storage with OpenShift Virtualization](#9-vm-storage-with-openshift-virtualization)
10. [Storage Monitoring](#10-storage-monitoring)
11. [GUI Health Check Guide](#11-gui-health-check-guide)
12. [Key Concepts Reference](#12-key-concepts-reference)
13. [Command Cheatsheet](#13-command-cheatsheet)

---

## 1. Prerequisites & Environment Check

### Verify Login
```bash
oc whoami
```
Expected output: `admin`

### Check Cluster Nodes
```bash
oc get nodes
```
Expected output:
```
NAME       STATUS   ROLES                         AGE    VERSION
master01   Ready    control-plane,master,worker   273d   v1.31.6
master02   Ready    control-plane,master,worker   273d   v1.31.6
master03   Ready    control-plane,master,worker   273d   v1.31.6
```

### Create a Working Namespace
```bash
oc new-project storage-lab
```

---

## 2. Storage Classes Overview

### List Available Storage Classes
```bash
oc get sc
```

### Describe a Specific Storage Class
```bash
oc describe sc nfs-storage
```

### Check if Expansion is Allowed (targeted)
```bash
oc get sc nfs-storage -o yaml | grep allowVolumeExpansion
```

### Storage Classes in This Lab

| Storage Class | Provisioner | Expandable | Best For |
|---|---|---|---|
| `nfs-storage` *(default)* | nfs-subdir-external-provisioner | No | General purpose |
| `ocs-external-storagecluster-ceph-rbd` | rbd.csi.ceph.com | Yes | Databases |
| `ocs-external-storagecluster-ceph-rbd-virtualization` | rbd.csi.ceph.com | Yes | VM disks |
| `ocs-external-storagecluster-ceph-rgw` | ceph.rook.io/bucket | No | Object/Bucket storage |
| `ocs-external-storagecluster-cephfs` | cephfs.csi.ceph.com | Yes | Shared filesystem |
| `openshift-storage.noobaa.io` | noobaa.io/obc | No | Multi-cloud object storage |

> **Note:** `nfs-storage` is the default storage class. OpenShift picks it automatically if no storage class is specified in the PVC.

---

## 3. Persistent Volumes & Claims

### Key Concepts

| Term | Description |
|---|---|
| **PV (Persistent Volume)** | Actual storage chunk provisioned in the cluster |
| **PVC (Persistent Volume Claim)** | A request for storage — matched to a PV |
| **StorageClass** | Defines the type and properties of storage |
| **RWO (ReadWriteOnce)** | Only one pod can read/write at a time |
| **RWX (ReadWriteMany)** | Multiple pods can read/write simultaneously |
| **Bound** | PVC successfully matched to a PV |

### List Existing Persistent Volumes
```bash
oc get pv
```

### List PVCs in Current Namespace
```bash
oc get pvc
```

### List PVCs Across All Namespaces
```bash
oc get pvc -A
```

---

## 4. Creating & Using PVCs with Pods

### Create a PVC
```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-first-pvc
  namespace: storage-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-storage
EOF
```

### Verify PVC is Bound
```bash
oc get pvc my-first-pvc -n storage-lab
```

### Create a Pod that Mounts the PVC
```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-test-pod
  namespace: storage-lab
spec:
  containers:
  - name: storage-container
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/sh", "-c", "echo 'Hello from Storage Lab!' > /mnt/storage/hello.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /mnt/storage
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-first-pvc
EOF
```

### Verify Pod is Running
```bash
oc get pod storage-test-pod -n storage-lab
```

### Verify Data Was Written
```bash
oc exec storage-test-pod -n storage-lab -- cat /mnt/storage/hello.txt
```

> **Important:** Always use single quotes in `oc exec` shell commands to avoid bash interpreting special characters like `!`. For example, use `sh -c 'echo Hello'` not `sh -c "echo Hello!"`.

---

## 5. Storage Persistence

Storage persistence means data survives pod deletion. The PVC and its data live independently of the pod that created it.

### Test Persistence — Delete the Pod
```bash
oc delete pod storage-test-pod -n storage-lab
```

### Verify PVC Still Exists
```bash
oc get pvc -n storage-lab
```

### Create a New Pod Mounting the Same PVC
```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-verify-pod
  namespace: storage-lab
spec:
  containers:
  - name: verify-container
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /mnt/storage
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-first-pvc
EOF
```

### Verify Data Still Exists in New Pod
```bash
oc exec storage-verify-pod -n storage-lab -- cat /mnt/storage/hello.txt
```

> **Key Lesson:** Pods are **temporary**. Storage is **permanent**. Never store important data inside a container — always use a PVC.

---

## 6. PVC Expansion

Only storage classes with `ALLOWVOLUMEEXPANSION: true` support live expansion. In this lab, `ocs-external-storagecluster-cephfs` supports expansion while `nfs-storage` does not.

### Create an Expandable PVC
```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
  namespace: storage-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-external-storagecluster-cephfs
EOF
```

### Method 1 — Expand via CLI Patch
```bash
oc patch pvc expandable-pvc -n storage-lab --type='merge' -p '{"spec":{"resources":{"requests":{"storage":"3Gi"}}}}'
```

### Method 2 — Expand via YAML Editor (Recommended)
```bash
oc edit pvc expandable-pvc -n storage-lab
```

Find and update this section:
```yaml
spec:
  resources:
    requests:
      storage: 1Gi   # Change this to your desired size e.g. 3Gi
```

**Vim quick reference:**

| Key | Action |
|---|---|
| `i` | Enter INSERT mode |
| `ESC` | Exit INSERT mode |
| `:wq` + ENTER | Save and quit |
| `:q!` + ENTER | Quit without saving |

### Verify Expansion
```bash
oc get pvc expandable-pvc -n storage-lab
```

> **Key Lesson:** PVC expansion is **live** — zero downtime, no pod restarts required when using a supported storage class.

---

## 7. Volume Snapshots & Restore

### Check Available Snapshot Classes
```bash
oc get volumesnapshotclass
```

| Snapshot Class | Driver | Use For |
|---|---|---|
| `ocs-external-storagecluster-cephfsplugin-snapclass` | CephFS | CephFS PVC snapshots |
| `ocs-external-storagecluster-rbdplugin-snapclass` | RBD | Block storage and VM snapshots |

> **Important:** Always match the snapshot class to the storage class of your PVC. CephFS PVCs use the CephFS snapclass. RBD PVCs use the RBD snapclass.

### Write Data Before Snapshotting

Create a pod mounted to the PVC first, then write data:

```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: expandable-pod
  namespace: storage-lab
spec:
  containers:
  - name: expandable-container
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /mnt/storage
      name: expandable-storage
  volumes:
  - name: expandable-storage
    persistentVolumeClaim:
      claimName: expandable-pvc
EOF
```

```bash
# Write data using single quotes to avoid bash special char issues
oc exec expandable-pod -- sh -c 'echo This data will be snapshotted > /mnt/storage/snapshot-test.txt'

# Verify data exists
oc exec expandable-pod -- cat /mnt/storage/snapshot-test.txt
```

### Take a Volume Snapshot
```bash
cat << EOF | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: expandable-pvc-snapshot-v2
  namespace: storage-lab
spec:
  volumeSnapshotClassName: ocs-external-storagecluster-cephfsplugin-snapclass
  source:
    persistentVolumeClaimName: expandable-pvc
EOF
```

### Verify Snapshot is Ready
```bash
oc get volumesnapshot -n storage-lab
```
Look for `READYTOUSE: true` before proceeding.

### Restore from Snapshot (Create New PVC from Snapshot)
```bash
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc-v2
  namespace: storage-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: ocs-external-storagecluster-cephfs
  dataSource:
    name: expandable-pvc-snapshot-v2
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOF
```

### Verify Restored Data
```bash
# Create a verification pod
cat << EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restore-verify-pod-v2
  namespace: storage-lab
spec:
  containers:
  - name: verify-container
    image: registry.access.redhat.com/ubi9/ubi:latest
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /mnt/storage
      name: restored-storage
  volumes:
  - name: restored-storage
    persistentVolumeClaim:
      claimName: restored-pvc-v2
EOF
```

```bash
# Verify pod is running then check data
oc get pod restore-verify-pod-v2 && oc exec restore-verify-pod-v2 -- cat /mnt/storage/snapshot-test.txt
```

> **Key Lesson:** Always write data to the PVC **before** taking a snapshot. A snapshot of empty storage restores to empty storage.

---

## 8. Object Storage with NooBaa

Object storage provides S3-compatible storage inside OpenShift. Unlike PVCs mounted as filesystems, object storage is accessed via HTTP API using access keys — exactly like Amazon S3.

### Check NooBaa Health
```bash
oc get noobaa -n openshift-storage
```
Look for `PHASE: Ready`.

### Get the External S3 Route
```bash
oc get route -n openshift-storage
```
Note the `s3` route hostname — this is the public S3 endpoint.

> **Important:** Use the external route hostname for s3cmd. The internal `s3.openshift-storage.svc` address only resolves from inside the cluster.

### Create an Object Bucket Claim (OBC)
```bash
cat << EOF | oc apply -f -
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-first-bucket
  namespace: storage-lab
spec:
  generateBucketName: my-first-bucket
  storageClassName: openshift-storage.noobaa.io
EOF
```

### Check OBC Status
```bash
oc get obc -n storage-lab
```
Look for `PHASE: Bound`.

### Retrieve Bucket Connection Details
```bash
# ConfigMap holds endpoint and bucket name
oc get configmap my-first-bucket -n storage-lab -o yaml
```

Key fields from ConfigMap:

| Field | Description |
|---|---|
| `BUCKET_NAME` | Auto-generated unique bucket name |
| `BUCKET_HOST` | Internal S3 service hostname |
| `BUCKET_PORT` | Port (443 for HTTPS) |

### Retrieve and Decode Bucket Credentials
```bash
# Decode Access Key
oc get secret my-first-bucket -n storage-lab -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d
echo ""

# Decode Secret Key
oc get secret my-first-bucket -n storage-lab -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d
echo ""
```

> **Note:** Credentials are stored base64-encoded in a Secret. Base64 is encoding, not encryption — always protect these values.

### Configure s3cmd with External Route
```bash
s3cmd --configure \
  --access_key=<DECODED_ACCESS_KEY> \
  --secret_key=<DECODED_SECRET_KEY> \
  --host=s3-openshift-storage.apps.ocp4.example.com \
  --host-bucket=s3-openshift-storage.apps.ocp4.example.com \
  --no-ssl \
  --dump-config 2>/dev/null | tee ~/.s3cfg
```

### List All Buckets
```bash
s3cmd ls --no-ssl
```

### Upload a File to Bucket
```bash
echo "Hello from OpenShift Object Storage!" > /tmp/test-object.txt
s3cmd put /tmp/test-object.txt s3://<BUCKET_NAME>/test-object.txt --no-ssl
```

### List Objects in Bucket
```bash
s3cmd ls s3://<BUCKET_NAME> --no-ssl
```

### Download an Object
```bash
s3cmd get s3://<BUCKET_NAME>/test-object.txt /tmp/retrieved-object.txt --no-ssl
cat /tmp/retrieved-object.txt
```

---

## 9. VM Storage with OpenShift Virtualization

VMs require block storage (`ceph-rbd-virtualization`) rather than shared filesystems. Block storage appears as a raw disk device to the VM operating system. RWX access mode is required for live migration.

### Check Virtualization is Running
```bash
oc get pods -n openshift-cnv | head -10
```

### List All Virtual Machines
```bash
oc get vm -A
```

### Key VM Storage Components

| Component | Purpose |
|---|---|
| **CDI (Containerized Data Importer)** | Imports disk images and creates PVCs automatically |
| **DataVolume** | Combines disk import + PVC creation in a single resource |
| **ceph-rbd-virtualization** | Block storage optimized for VM disks |
| **RWX on VM disks** | Required for live migration between nodes |

### Create a DataVolume (Blank VM Disk)
```bash
cat << EOF | oc apply -f -
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: my-vm-disk
  namespace: storage-lab
spec:
  source:
    blank: {}
  storage:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 10Gi
    storageClassName: ocs-external-storagecluster-ceph-rbd-virtualization
EOF
```

### Check DataVolume Status
```bash
oc get datavolume my-vm-disk -n storage-lab
```
Look for `PHASE: Succeeded` and `PROGRESS: 100.0%`.

### Verify Auto-Created PVC
```bash
oc get pvc my-vm-disk -n storage-lab
```

### Create a Virtual Machine
```bash
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: my-test-vm
  namespace: storage-lab
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        kubevirt.io/vm: my-test-vm
    spec:
      domain:
        devices:
          disks:
          - name: mydisk
            disk:
              bus: virtio
        resources:
          requests:
            memory: 1Gi
      volumes:
      - name: mydisk
        dataVolume:
          name: my-vm-disk
EOF
```

> **Note:** Use `runStrategy: Always` instead of `running: true/false`. The `spec.running` field is deprecated in OpenShift Virtualization v4.18+.

### Start and Stop VM
```bash
virtctl start my-test-vm -n storage-lab
virtctl stop my-test-vm -n storage-lab
```

### Check VM and VMI Status
```bash
# VM object (definition)
oc get vm my-test-vm -n storage-lab

# VMI object (running instance — shows IP and node)
oc get vmi my-test-vm -n storage-lab
```

### Verify VM Storage and Live Migration Capability
```bash
oc describe vmi my-test-vm -n storage-lab | grep -A5 "Volumes"
```
Look for `LiveMigratable: True` — confirms RWX storage is correctly configured.

---

## 10. Storage Monitoring

### Check Ceph Cluster Health
```bash
oc get cephcluster -n openshift-storage
```
Look for `HEALTH: HEALTH_OK` and `PHASE: Connected`.

### Check Storage System
```bash
oc get storagesystem -n openshift-storage
```

### Audit All PVCs Across the Cluster
```bash
oc get pvc -A
```

### Storage Class Usage Summary
```bash
oc get pvc -A --no-headers | awk '{print $7}' | sort | uniq -c | sort -rn
```

### Check Real Disk Usage from Inside a Pod
```bash
oc exec <pod-name> -n storage-lab -- df -h /mnt/storage
```

### Check Storage Usage Across All Pods in a Namespace
```bash
for pod in $(oc get pods -n storage-lab --no-headers -o custom-columns=NAME:.metadata.name); do
  echo "=== $pod ==="
  oc exec $pod -n storage-lab -- df -h /mnt/storage 2>/dev/null || echo "No /mnt/storage mounted"
done
```

### Check Storage-Related Events
```bash
oc get events -n storage-lab --sort-by='.lastTimestamp' | tail -10
```

### Full Cluster Health Check (One Command)
```bash
echo "=== NODES ===" && oc get nodes && \
echo "=== STORAGE CLASSES ===" && oc get sc && \
echo "=== ALL PVCs ===" && oc get pvc -n storage-lab && \
echo "=== SNAPSHOTS ===" && oc get volumesnapshot -n storage-lab && \
echo "=== PODS ===" && oc get pods -n storage-lab && \
echo "=== VM ===" && oc get vm -n storage-lab && \
echo "=== BUCKET ===" && oc get obc -n storage-lab && \
echo "=== CEPH HEALTH ===" && oc get cephcluster -n openshift-storage
```

---

## 11. GUI Health Check Guide

### Access OpenShift Console
```
https://console-openshift-console.apps.ocp4.example.com
```
Login with: `admin` / your lab password

### Navigation Reference

| What to Check | Navigation Path |
|---|---|
| Overall Storage Health | Home → Overview → Storage tab |
| ODF Dashboard | Storage → Data Foundation → Overview |
| All PVCs | Storage → PersistentVolumeClaims |
| Volume Snapshots | Storage → VolumeSnapshots |
| Object Buckets | Storage → ObjectBucketClaims |
| VM Disks | Virtualization → VirtualMachines → [VM Name] → Disks tab |
| Storage Alerts | Observe → Alerting |
| Capacity Breakdown | Storage → Data Foundation → Overview → Storage Systems |

### ODF Dashboard Tabs

| Tab | Shows |
|---|---|
| Overview | Overall health, capacity, performance (IOPS, throughput, latency) |
| System | Individual storage system status |
| Storage classes | Per-class usage breakdown and trends |

### Health Color Indicators

| Color | Meaning | Recommended Action |
|---|---|---|
| Green | Healthy | No action needed |
| Yellow | Warning | Investigate within business hours |
| Red | Critical | Immediate action required |
| Grey | Unknown | Check connectivity and operator status |

---

## 12. Key Concepts Reference

### Storage Hierarchy
```
StorageClass
    └── PersistentVolume (PV)              <- provisioned by StorageClass
          └── PersistentVolumeClaim (PVC)  <- claimed by workload
                └── Pod / VM              <- consumes the PVC
```

### Access Modes

| Mode | Short | Description | Use Case |
|---|---|---|---|
| ReadWriteOnce | RWO | One node reads/writes | Databases, single-pod apps |
| ReadWriteMany | RWX | Multiple nodes read/write | Shared storage, VM live migration |
| ReadOnlyMany | ROX | Multiple nodes read only | Static content |

### Reclaim Policies

| Policy | What Happens When PVC is Deleted |
|---|---|
| Delete | PV and underlying storage are deleted |
| Retain | PV remains and must be manually reclaimed |

### Storage Type Comparison

| Type | Access Method | Best For | OpenShift Resource |
|---|---|---|---|
| Block (RBD) | Raw block device | Databases, VMs | PVC with ceph-rbd |
| File (CephFS) | POSIX filesystem | Shared app storage | PVC with cephfs |
| Object (NooBaa) | S3 HTTP API | Backups, media, logs | ObjectBucketClaim |

### DataVolume vs PVC

| Feature | PVC | DataVolume |
|---|---|---|
| Creates storage | Yes | Yes |
| Imports disk image | No | Yes |
| Tracks import progress | No | Yes |
| Used for VMs | Manual wiring | Automatic |

---

## 13. Command Cheatsheet

### Environment
```bash
oc whoami                                    # Check current user
oc get nodes                                 # List cluster nodes
oc new-project <name>                        # Create namespace
oc project                                   # Show current namespace
```

### Storage Classes
```bash
oc get sc                                    # List all storage classes
oc describe sc <name>                        # Describe a storage class
oc get sc <name> -o yaml                     # Full YAML output
oc get sc <name> -o yaml | grep allowVolume  # Check expansion support
```

### Persistent Volumes and Claims
```bash
oc get pv                                    # List all PVs
oc get pvc                                   # List PVCs in current namespace
oc get pvc -A                                # List PVCs in all namespaces
oc describe pvc <name>                       # Describe a PVC
oc edit pvc <name>                           # Edit PVC live (e.g. expand)
oc delete pvc <name>                         # Delete a PVC
```

### Pods and Storage
```bash
oc get pods                                  # List pods
oc describe pod <name>                       # Describe pod details
oc exec <pod> -- <command>                   # Run command in pod
oc exec <pod> -- df -h /mnt/storage          # Check disk usage in pod
oc logs <pod>                                # View pod logs
oc delete pod <name>                         # Delete a pod
```

### Snapshots
```bash
oc get volumesnapshotclass                   # List snapshot classes
oc get volumesnapshot                        # List snapshots in namespace
oc describe volumesnapshot <name>            # Describe a snapshot
oc delete volumesnapshot <name>              # Delete a snapshot
```

### Object Storage
```bash
oc get obc                                   # List Object Bucket Claims
oc get noobaa -n openshift-storage           # Check NooBaa health
oc get route -n openshift-storage            # Get S3 public routes
s3cmd ls --no-ssl                            # List all buckets
s3cmd put <file> s3://<bucket>/<key>         # Upload a file
s3cmd get s3://<bucket>/<key> <local-file>   # Download a file
s3cmd ls s3://<bucket> --no-ssl             # List objects in bucket
```

### Virtual Machines
```bash
oc get vm -A                                 # List all VMs across namespaces
oc get vmi                                   # List running VM instances
oc get datavolume                            # List DataVolumes
virtctl start <vm-name>                      # Start a VM
virtctl stop <vm-name>                       # Stop a VM
virtctl console <vm-name>                    # Access VM serial console
```

### Monitoring
```bash
oc get cephcluster -n openshift-storage      # Ceph health status
oc get storagesystem -n openshift-storage    # Storage system overview
oc get events --sort-by='.lastTimestamp'     # Recent events sorted by time
oc get pvc -A --no-headers | awk '{print $7}' | sort | uniq -c | sort -rn  # PVC count per storage class
```

### Cleanup
```bash
oc delete project storage-lab                # Delete namespace and ALL its resources
```

---

## Lab Accomplishments Summary

| Area | What Was Done |
|---|---|
| PVC Basics | Created PVC on NFS, mounted to pod, wrote and verified data |
| Storage Persistence | Deleted pod, confirmed data survived, re-mounted PVC to new pod |
| PVC Expansion | Expanded CephFS PVC from 1Gi to 4Gi live with zero downtime |
| Snapshots | Took snapshot of PVC with data, restored to new PVC, verified data integrity |
| Object Storage | Created OBC via NooBaa, decoded credentials, configured s3cmd, uploaded and downloaded objects |
| VM Storage | Created blank DataVolume, launched VM with ceph-rbd-virtualization, verified live migration capability |
| Monitoring | Audited cluster-wide storage consumption, checked Ceph health, reviewed events, ran per-pod disk reports |
| GUI Navigation | Explored ODF dashboard, PVC list, snapshot view, and VM disk configuration via web console |

---

*Generated from a Red Hat OpenShift Virtualization Administration Rapid Track v4.18 hands-on lab session.*
