# Create Virtual Machine in OpenShift Virtualization - GUI Guide

**Author:** Sajal Jana

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Navigate to Virtualization](#step-1-navigate-to-virtualization)
3. [Step 2: Create Virtual Machine](#step-2-create-virtual-machine)
4. [Step 3: Configure Basic Settings](#step-3-configure-basic-settings)
5. [Step 4: Configure Storage](#step-4-configure-storage)
6. [Step 5: Configure Networking](#step-5-configure-networking)
7. [Step 6: Configure Cloud-Init](#step-6-configure-cloud-init)
8. [Step 7: Configure Readiness Probe](#step-7-configure-readiness-probe)
9. [Step 8: Review and Create](#step-8-review-and-create)
10. [Verification](#verification)

---

## Prerequisites

- OpenShift Virtualization operator installed
- Project `banana` created
- Network attachment definition `database-network` exists in `banana` namespace
- SSH public key from `/home/student/.ssh/lab_rsa.pub` available

---

## Step 1: Navigate to Virtualization

**Purpose:** Access the VM management interface in the banana project

1. Login to **OpenShift Web Console**
2. Switch to **Administrator** perspective
3. Select project: **banana** (from project dropdown)
4. Click **Virtualization** → **VirtualMachines** in left menu

---

## Step 2: Create Virtual Machine

**Purpose:** Initiate VM creation using RHEL 9 template

1. Click **Create** → **From template**
2. Select template: **Red Hat Enterprise Linux 9 VM**
3. Click **Customize VirtualMachine**

---

## Step 3: Configure Basic Settings

**Purpose:** Set VM name, workload type, and resource flavor

1. **General** section:
   - **Name:** `myvm-lan1`
   - **Workload type:** `Server`
   - **Flavor:** `Small`

2. Click **Next** or scroll to **Storage** section

---

## Step 4: Configure Storage

**Purpose:** Configure 30Gi block storage using HTTP URL as boot source

1. **Boot source** section:
   - **Source type:** Select `URL (creates PVC)`
   - **URL:** `http://utility.lab.example.com:8080/openshift4/images/rhel9-helloworld.qcow2`

2. **Disk** configuration:
   - **PVC size:** `30` (GiB)
   - **Storage class:** `ocs-external-storagecluster-ceph-rbd-virtualization`
   - **Volume mode:** `Block`
   - **Access mode:** `ReadWriteMany` (auto-selected)

3. Click **Next** or scroll to **Network interfaces**

---

## Step 5: Configure Networking

**Purpose:** Configure dual network interfaces - pod networking and database-network bridge

### Interface 1 (Default - Already exists)

- **Name:** `default`
- **Model:** `virtio`
- **Network:** `Pod networking`
- **Type:** `masquerade`
- Leave as default

### Interface 2 (Add New)

1. Click **Add network interface**
2. Configure:
   - **Name:** `nic-0`
   - **Model:** `virtio`
   - **Network:** Select `database-network` (from banana namespace)
   - **Type:** `Bridge`
3. Click **Save**

---

## Step 6: Configure Cloud-Init

**Purpose:** Set up user credentials and SSH key authentication

1. Scroll to **Scripts** section
2. Click **Cloud-init** tab or **Edit** if visible
3. Select **Configure via: Script**
4. Paste the following cloud-init configuration:

```yaml
#cloud-config
user: ram
password: ram2026
chpasswd:
  expire: false
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC... devops@workstation
```

**Note:** Replace the SSH key with actual content from `/home/student/.ssh/lab_rsa.pub`

5. Click **Apply** or **Save**

---

## Step 7: Configure Readiness Probe

**Purpose:** Configure TCP health check on SSH port to monitor VM readiness

1. Scroll to **Advanced** section
2. Click **Health checks** or **Probes**
3. Click **Add readiness probe**
4. Configure:
   - **Probe type:** `TCP Socket`
   - **Port:** `22`
   - **Initial delay (seconds):** `120`
   - **Period (seconds):** `10`
   - **Timeout (seconds):** `5`
   - **Success threshold:** `1`
   - **Failure threshold:** `3`
5. Click **Save** or **Apply**

---

## Step 8: Review and Create

**Purpose:** Finalize configuration and deploy the virtual machine

1. Review all configurations
2. Ensure **Start this VirtualMachine after creation** is checked
3. Click **Create VirtualMachine**

---

## Verification

### Check VM Status

1. Navigate to **Virtualization** → **VirtualMachines**
2. Find `myvm-lan1` in the list
3. Wait for status to change to **Running** (may take 5-10 minutes)
4. Verify readiness probe shows **Ready**

### Verify Network Configuration

1. Click on `myvm-lan1` VM name
2. Go to **Network interfaces** tab
3. Verify two interfaces:
   - `default` (masquerade, Pod networking)
   - `nic-0` (bridge, database-network)

### Verify Storage

1. Click **Disks** tab
2. Verify `myvm-lan1-rootdisk` shows:
   - Size: `30Gi`
   - Storage class: `ocs-external-storagecluster-ceph-rbd-virtualization`

### Access VM Console

1. Click **Console** tab
2. Login with credentials:
   - **User:** `ram`
   - **Password:** `ram2026`
3. Verify SSH access works (readiness probe on port 22)

---

## Summary

Virtual Machine `myvm-lan1` has been successfully created with:

- ✅ RHEL 9 template
- ✅ Server workload, Small flavor
- ✅ 30Gi Block storage from URL source
- ✅ Two network interfaces (Pod + database-network)
- ✅ Cloud-init with user ram and SSH key
- ✅ TCP readiness probe on port 22

---

*End of Document*
