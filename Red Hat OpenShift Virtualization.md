# Red Hat OpenShift Virtualization 🚀

**Author:** Sajal Jana 

---

## 1. KubeVirt Architecture

**What is it?**  
KubeVirt lets you run traditional VMs inside Kubernetes. It's like teaching an old dog new tricks, except the old dog is your legacy app, and the new trick is playing nice with containers.

**Why should I care?**  
Because rewriting every legacy app would take until the heat death of the universe. KubeVirt lets you modernize gradually without your CFO having a meltdown.

**The Apartment Building Analogy:**  
Your Kubernetes cluster is like a modern apartment building originally designed for studio apartments (containers). KubeVirt is the renovation that also allows traditional family apartments (VMs) in the same building. Same lobby, same elevators, same stressed-out building manager (that's you).

### Quick Admin Checklist:

1. **Install OpenShift Virtualization Operator**
   - Go to OperatorHub → Search → Install
   - *Coffee break while it installs* ☕

2. **Deploy HyperConverged Resource**
   - One YAML to rule them all
   - Check pods are happy: `oc get pods -n openshift-cnv`

3. **Label Your Nodes**
   - Check virtualization support: `lscpu | grep Virtualization`
   - If it says nothing, you're gonna have a bad time
   - Label nodes: `oc label node <name> kubevirt.io/schedulable=true`

4. **Set Up Storage Classes**
   - VMs need disks (shocker!)
   - Use `oc get storageclass` to see what you've got
   - ReadWriteMany = can migrate; ReadWriteOnce = VM is stuck like Chuck

5. **Monitor the Things**
   - `oc get pods -n openshift-cnv` (add to your daily routine)
   - If virt-handler is down, nothing works—kind of important

6. **Set Resource Quotas**
   - Without limits, Bob from accounting will spin up 500 VMs for "testing"
   - `oc create quota vm-quota --hard=requests.memory=64Gi`

---

## 2. Live Migration

**What is it?**  
Moving a running VM from one node to another without turning it off. It's like switching horses mid-gallop, except less likely to result in a faceplant.

**Why should I care?**  
Because "scheduled downtime" is just "we broke your stuff at 2 AM but planned it" to your users. Live migration = zero downtime maintenance = happy users = you keep your job.

**The Cell Tower Analogy:**  
Ever notice your phone call doesn't drop when you're driving and switch cell towers? That's because telecom engineers are wizards. Live migration does the same for VMs—users don't even notice the "tower" (server) changed underneath them.

### Quick Admin Checklist:

1. **Check Prerequisites**
   - Storage must support ReadWriteMany (RWX)
   - No RWX = no migration = sad admin
   - `oc get storageclass -o json | grep accessModes`

2. **Configure Migration Settings**
   ```bash
   oc edit hyperconverged kubevirt-hyperconverged -n openshift-cnv
   ```
   - Set bandwidth limits (don't saturate your network)
   - Parallel migrations: start small, increase gradually

3. **Test in Non-Prod First**
   - Seriously, test it
   - Measure how long it takes
   - Breaking production during migration is a résumé-generating event

4. **Manual Migration**
   ```bash
   virtctl migrate <vm-name>
   oc get virtualmachineinstancemigration -w
   ```
   - Watch it migrate like a proud parent

5. **Automate All The Things**
   - Add to VM spec:
   ```yaml
   spec:
     evictionStrategy: LiveMigrate
   ```
   - Now VMs auto-evacuate during node maintenance (fancy!)

6. **Troubleshoot Failures**
   - Check logs: `oc describe virtualmachineinstancemigration <name>`
   - Common culprits: no space on target node, network hiccups, cosmic rays

7. **Document Performance Baselines**
   - "Small VM: 30 seconds, Large VM: 5 minutes, Bob's Test VM: eternity"

---

## 3. Multus CNI (Multiple Networks)

**What is it?**  
Multus lets VMs have multiple network interfaces, like giving your VM multiple arms to reach different networks simultaneously.

**Why should I care?**  
Because putting management traffic, application traffic, and backup traffic on the same network is like using one highway lane for ambulances, trucks, and Sunday drivers. Chaos ensues.

**The Multiple Network Cards Analogy:**  
Your office computer might have one cable for internet, another for the secure finance network, and a third for backups. Multus gives VMs the same superpower—multiple "network cards" for different purposes.

### Quick Admin Checklist:

1. **Verify Multus is Running**
   ```bash
   oc get ds multus -n openshift-multus
   oc get pods -n openshift-multus
   ```
   - Should be running everywhere. If not, panic mildly.

2. **Create Network Attachment Definitions (NADs)**
   ```yaml
   apiVersion: k8s.cni.cncf.io/v1
   kind: NetworkAttachmentDefinition
   metadata:
     name: database-network
   spec:
     config: '{
       "type": "bridge",
       "bridge": "br1",
       "vlan": 100
     }'
   ```
   - This is your network template

3. **Set Up Linux Bridges**
   - Use NodeNetworkConfigurationPolicy
   - Connects VMs to physical networks
   - Layer 2 magic ✨

4. **Attach Networks to VMs**
   ```yaml
   spec:
     networks:
     - name: default
       pod: {}
     - name: database-net
       multus:
         networkName: database-network
   ```
   - Now your VM is multi-talented

5. **Configure SR-IOV** (For Speed Demons)
   - Direct hardware access
   - Near-native performance
   - Required for: high-frequency trading, low-latency apps, impressing your boss

6. **Apply Network Policies**
   - Because multiple networks without isolation is like having multiple doors with no locks
   - Define who can talk to whom

7. **Troubleshoot Connectivity**
   ```bash
   virtctl console <vm-name>
   ip addr show  # Inside the VM
   ```
   - When in doubt, ping everything

---

## 4. CDI (Containerized Data Importer)

**What is it?**  
CDI automates importing VM disk images into Kubernetes storage. It's like having a robot that downloads, converts, and organizes your VM images while you do literally anything else.

**Why should I care?**  
Because manually copying VM images is the IT equivalent of copying files by hand with a quill and parchment. CDI turns hours of tedious work into "kubectl apply" and a coffee break.

**The Office Copy Service Analogy:**  
Instead of running around with USB drives copying files to each computer, you upload once to a central system that automatically converts, copies, and delivers to the right places. CDI is that automated system for VM images.

### Quick Admin Checklist:

1. **Verify CDI is Alive**
   ```bash
   oc get pods -n openshift-cnv | grep cdi
   ```
   - If cdi-controller is dead, nothing works

2. **Create a DataVolume to Import Images**
   ```yaml
   apiVersion: cdi.kubevirt.io/v1beta1
   kind: DataVolume
   metadata:
     name: rhel8-golden-image
   spec:
     source:
       http:
         url: "https://example.com/rhel8.qcow2"
     pvc:
       accessModes:
         - ReadWriteOnce
       resources:
         requests:
           storage: 30Gi
   ```
   - Point, shoot, and CDI does the rest

3. **Upload Local Images**
   ```bash
   virtctl image-upload dv <name> --image-path=/path/to/image.qcow2 --size=30Gi
   ```
   - For when your image is "on my laptop"

4. **Clone Existing Volumes**
   - Way faster than re-downloading
   - Perfect for spinning up multiple identical VMs
   - Clone responsibly

5. **Use Container Registries**
   - Store VM images in Quay/Harbor alongside container images
   - Leverage existing registry security and scanning
   - One registry to rule them all

6. **Golden Image Strategy**
   - Create a dedicated namespace: `golden-images`
   - Version your images (no more "latest-final-v2-really-final")
   - Control who can modify them (hint: not everyone)

7. **Troubleshoot Import Failures**
   ```bash
   oc get dv -A  # Check status
   oc describe dv <name>  # Read the tea leaves
   oc logs <importer-pod>  # Actual answers here
   ```
   - Common issues: network timeouts, wrong format, not enough space

---

## 5. Node Maintenance Mode

**What is it?**  
A safe way to take a node offline for maintenance without yeeting all the VMs into the void.

**Why should I care?**  
Because "I accidentally shut down the node with the production database" is not a career highlight. Maintenance mode = graceful evacuation = VMs live to serve another day.

**The Highway Lane Closure Analogy:**  
Highway workers don't just throw up barriers and cause a 50-car pileup. They use signs to redirect traffic, ensure everyone moved over safely, then close the lane. Maintenance mode does the same—redirect workloads, wait for safe migration, then proceed.

### Quick Admin Checklist:

1. **Check Cluster Capacity**
   ```bash
   oc describe nodes | grep -A 5 "Allocated resources"
   ```
   - Do other nodes have room? If not, abort mission

2. **Use Node Maintenance Operator**
   ```yaml
   apiVersion: nodemaintenance.medik8s.io/v1beta1
   kind: NodeMaintenance
   metadata:
     name: worker-3-maintenance
   spec:
     nodeName: worker-node-3
     reason: "OS patching (Bob finally approved it)"
   ```
   - Automation = fewer mistakes

3. **Manual Method** (For the Brave)
   ```bash
   oc adm cordon <node-name>  # No new workloads
   oc adm drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```
   - More control, more responsibility

4. **Monitor Migration Progress**
   ```bash
   oc get vmi -o wide --watch
   oc get virtualmachineinstancemigration -w
   ```
   - Watch VMs evacuate like it's a fire drill

5. **Do the Maintenance**
   - Patch the OS
   - Update firmware
   - Replace that weird rattling fan
   - Document everything (future you will thank present you)

6. **Return Node to Service**
   ```bash
   oc delete nodemaintenance <name>  # OR
   oc adm uncordon <node-name>
   ```
   - Node rejoins the party

7. **Verify Everything is Fine**
   ```bash
   oc get nodes  # Should be Ready
   oc describe node <name>  # Check for weirdness
   oc get vmi -A -o wide  # VMs distributed properly?
   ```
   - If something seems off, investigate before it becomes a fire

---

## Conclusion

Congrats! You now know the secret sauce of running VMs in Kubernetes:

- **KubeVirt**: VMs and containers living together (mass hysteria!)
- **Live Migration**: Moving VMs without downtime (magic!)
- **Multus**: Multiple networks for VMs (like having multiple personalities, but productive)
- **CDI**: Automated image imports (because you have better things to do)
- **Node Maintenance**: Safe maintenance workflows (job security!)


---

**Author:** Sajal Jana  
**Disclaimer:** No VMs were harmed in the making of this guide 🖥️
