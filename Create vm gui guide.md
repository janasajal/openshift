# Creating VirtualMachine in OpenShift Virtualization - Step-by-Step GUI Guide

---

## Question

### Prepare the lab for this question

```bash
lab start accessing-guicreate
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
ssh-keygen -t rsa -q -f /home/student/.ssh/id_rsa  -N ""
```

### Create a VirtualMachine in the `banana` project with below requirements

- User `ram` should create a VirtualMachine named `testvm` from template "Red Hat Enterprise Linux 9 VM"
- Use PVC URL, `http://utility.lab.example.com:8080/openshift4/images/rhel9-helloworld.qcow2`
- The StorageClassName is `ocs-external-storagecluster-ceph-rbd-virtualization`
- The PVC size should be 30GiB
- The Volume mode should be Block
- The workload type is the VirtualMachine is `server`
- The flavor type of the VirtualMachine is `small`
- The network interface name is `default`
- The user `ram` with password `ram123` exists in the cloud-init definition
- The ssh Key "/home/student/.ssh/lab_rsa.pub" from user devops at workstation has been added as an authorized ssh key via the cloud-init definition

### Task: Configure Network interface

**The first Network Interface configuration:**
- The first Network interface name is `default`
- The First Network interface is attached to the `pod networking` (default) network
- The first network interface type is `masquerade`
- The model for the first network interface is `virtio`

**The Second Network Interface Configuration:**
- The second network interface name is `nic-0`
- The second network interface is attached to the `banana/database-network` network
- The second network interface type is `bridge`
- The IP address of the second network interface is provided by OpenShift
- The model for the second network interface is `virtio`

### Task: Create a Readiness probe with below configuration

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 2
  successThreshold: 1
```

---

## Prerequisites - Lab Preparation

```bash
# Start the lab environment
lab start accessing-guicreate

# Login as admin
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443

# Generate SSH key
ssh-keygen -t rsa -q -f /home/student/.ssh/id_rsa -N ""
```

---

## Solution: Creating VirtualMachine `testvm`

### Step 1: Login to OpenShift Web Console as User `ram`

1. Open browser and navigate to OpenShift Console: `https://console-openshift-console.apps.ocp4.example.com`
2. Click **htpasswd_provider** (or your configured identity provider)
3. Enter credentials:
   - **Username:** `ram`
   - **Password:** `ram123`
4. Click **Log in**

---

### Step 2: Navigate to the `banana` Project

1. In the top navigation bar, click the **Project** dropdown
2. Select or search for **banana** project
3. If project doesn't exist, create it:
   - Click **Create Project**
   - **Name:** `banana`
   - Click **Create**

---

### Step 3: Access OpenShift Virtualization

1. In the left sidebar, click **Virtualization**
2. This expands the Virtualization menu
3. Click **VirtualMachines** under Virtualization section

---

### Step 4: Create New VirtualMachine from Template

1. Click **Create** button (top right)
2. Select **From template**
3. In the template catalog, find and click **Red Hat Enterprise Linux 9 VM**
4. Click **Quick create VirtualMachine** or **Customize VirtualMachine**
5. Select **Customize VirtualMachine** for full configuration options

---

### Step 5: Configure Basic VM Details

**In the General section:**

1. **Name:** `testvm`
2. **Project:** Verify it shows `banana`
3. **Template:** Red Hat Enterprise Linux 9 VM (already selected)
4. **Workload type:** Select **Server**
5. **Flavor:** Select **Small**

---

### Step 6: Configure Boot Source (PVC from URL)

**In the Boot source section:**

1. **Boot source type:** Select **URL (creates PVC)**
2. **Image URL:** `http://utility.lab.example.com:8080/openshift4/images/rhel9-helloworld.qcow2`
3. **PVC name:** `testvm-boot` (auto-generated or specify)
4. **Storage class:** Select `ocs-external-storagecluster-ceph-rbd-virtualization`
5. **PVC size:** `30 GiB`
6. **Access mode:** Will be set by volume mode
7. **Volume mode:** Select **Block**

---

### Step 7: Configure Cloud-init

**In the Scripts section or Advanced → Cloud-init:**

1. Click **Scripts** tab (or **Advanced** → **Cloud-init**)
2. Enable **Configure via cloud-init**
3. Select **Edit** to customize cloud-init

**Add the following cloud-init configuration:**

```yaml
#cloud-config
user: ram
password: ram
chpasswd:
  expire: false
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2E... (paste content from /home/student/.ssh/lab_rsa.pub)
```

**To get the SSH public key content:**
```bash
cat /home/student/.ssh/lab_rsa.pub
```

**Alternatively, use the UI fields:**
- **User:** `ram`
- **Password:** `ram123`
- **Authorized SSH key:** Click **Add SSH key** and paste the content of `/home/student/.ssh/lab_rsa.pub`

---

### Step 8: Configure Network Interfaces

**Navigate to the Networking section:**

#### First Network Interface (Default - Already configured):

1. **Interface name:** `default`
2. **Network:** `Pod networking` (default)
3. **Type:** `masquerade`
4. **Model:** `virtio`

This interface is typically configured by default. Verify settings:
- Click on the **default** interface
- Ensure **Type** is `masquerade`
- Ensure **Model** is `virtio`

#### Add Second Network Interface:

1. Click **Add network interface** button
2. Configure as follows:
   - **Name:** `nic-0`
   - **Network:** Select `banana/database-network`
     - If not in dropdown, you may need to type: `banana/database-network`
   - **Type:** `bridge`
   - **Model:** `virtio`
   - **MAC address:** Leave as auto-generated (OpenShift will assign IP via IPAM)
3. Click **Add** or **Save**

**Note:** The `banana/database-network` NetworkAttachmentDefinition must exist beforehand. If it doesn't exist, create it first:

```bash
# As admin, create the network attachment definition
oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
oc project banana

cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: database-network
  namespace: banana
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br-database",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24"
      }
    }
EOF
```

---

### Step 9: Configure Readiness Probe

**Navigate to Advanced → Health checks:**

1. Click **Add health check** (or **Add readiness probe**)
2. Select **Readiness probe**
3. Configure the probe:
   - **Probe type:** `HTTP GET`
   - **Path:** `/health`
   - **Port:** `80`
   - **Initial delay (seconds):** `10`
   - **Period (seconds):** `5`
   - **Timeout (seconds):** `2`
   - **Success threshold:** `1`
   - **Failure threshold:** `2`
4. Click **Add** or **Save**

**UI Field Mapping:**
- **initialDelaySeconds** → Initial delay: `10`
- **periodSeconds** → Period: `5`
- **timeoutSeconds** → Timeout: `2`
- **failureThreshold** → Failure threshold: `2`
- **successThreshold** → Success threshold: `1`

---

### Step 10: Review and Create VM

1. Review all configurations in the summary panel (right side)
2. Verify all settings are correct:
   - Name: `testvm`
   - Project: `banana`
   - Workload: Server
   - Flavor: Small
   - Boot source: URL with 30GiB Block PVC
   - Network interfaces: 2 (default + nic-0)
   - Cloud-init configured with user `ram`
   - Readiness probe configured
3. Click **Create VirtualMachine** button

---

### Step 11: Start the VirtualMachine

After creation:

1. The VM appears in the VirtualMachines list
2. Click on **testvm** to view details
3. Click **Actions** → **Start** (or click the Start button)
4. Wait for the VM to boot (status changes to **Running**)
5. Monitor the **Events** and **Console** tabs for progress

---

## Verification Steps

### Verify VM Configuration:

```bash
# Login as ram or admin
oc login -u ram -p ram123 https://api.ocp4.example.com:6443
oc project banana

# Check VM status
oc get vm testvm

# Check VMI (VirtualMachineInstance)
oc get vmi testvm

# Describe VM to see full configuration
oc describe vm testvm

# Verify network interfaces
oc get vmi testvm -o jsonpath='{.spec.domain.devices.interfaces[*].name}' && echo

# Verify readiness probe
oc get vmi testvm -o yaml | grep -A 10 readinessProbe
```

### Verify Network Connectivity:

```bash
# Get VM IP addresses
oc get vmi testvm -o jsonpath='{.status.interfaces[*].ipAddress}' && echo

# Console access
virtctl console testvm

# SSH access (if cloud-init completed)
ssh ram@<vm-ip-address>
```

### Verify PVC:

```bash
# Check PVC created for boot disk
oc get pvc | grep testvm

# Verify PVC details
oc describe pvc testvm-boot
```

---

## Summary of Configuration

| Component | Value |
|-----------|-------|
| **VM Name** | testvm |
| **Project** | banana |
| **Template** | Red Hat Enterprise Linux 9 VM |
| **Workload Type** | Server |
| **Flavor** | Small |
| **Boot Source** | URL (PVC) |
| **Image URL** | http://utility.lab.example.com:8080/openshift4/images/rhel9-helloworld.qcow2 |
| **Storage Class** | ocs-external-storagecluster-ceph-rbd-virtualization |
| **PVC Size** | 30 GiB |
| **Volume Mode** | Block |
| **Cloud-init User** | ram |
| **Cloud-init Password** | ram123 |
| **SSH Key** | /home/student/.ssh/lab_rsa.pub |
| **NIC 1 Name** | default |
| **NIC 1 Network** | Pod networking |
| **NIC 1 Type** | masquerade |
| **NIC 1 Model** | virtio |
| **NIC 2 Name** | nic-0 |
| **NIC 2 Network** | banana/database-network |
| **NIC 2 Type** | bridge |
| **NIC 2 Model** | virtio |
| **Readiness Probe** | HTTP GET /health:80 |

---

## Troubleshooting

### If VM fails to start:

1. Check Events in the VM details page
2. Review logs: `oc logs virt-launcher-testvm-xxxxx`
3. Verify PVC is bound: `oc get pvc`
4. Check if image download succeeded

### If network interface doesn't attach:

1. Verify NetworkAttachmentDefinition exists: `oc get network-attachment-definitions -n banana`
2. Check Multus is configured properly
3. Verify bridge CNI plugin is available

### If readiness probe fails:

1. Access VM console: `virtctl console testvm`
2. Check if the service on port 80 is running
3. Verify the `/health` endpoint exists
4. Adjust probe timings if service takes longer to start

---

## Additional Notes

- The VM template may have default values that need to be adjusted
- Ensure the `banana/database-network` NetworkAttachmentDefinition is created before adding the second NIC
- The SSH key must be the public key from `/home/student/.ssh/lab_rsa.pub`
- Cloud-init may take a few minutes to complete after VM boot
- The readiness probe will only pass when the HTTP service on port 80 responds successfully to `/health` endpoint

---

**Lab Completion Command:**
```bash
lab finish accessing-guicreate
```
