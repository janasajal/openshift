# Project Report: End-to-End Implementation of Red Hat OpenShift Container Platform (UPI) on HP ProLiant Gen10 Bare-Metal Infrastructure

## 1. Executive Summary

This project report outlines the planning, architectural prerequisites, implementation steps, and post-deployment validation required to deploy a production-ready **Red Hat OpenShift Container Platform (OCP)** cluster on bare-metal physical servers (**HP ProLiant Gen10**) using the **User-Provisioned Infrastructure (UPI)** method.

Unlike cloud deployments where network routers, load balancers, and virtual compute nodes are dynamically provisioned via cloud provider APIs, bare-metal UPI requires system administrators to manually design and deploy local infrastructure services. This report provides a structured technical walkthrough tailored for students and engineering trainees to understand both the high-level concepts and ground-level configurations.

## 2. Theoretical Architecture & Core Components

To understand OpenShift on bare metal, each supporting component can be conceptualized through standard computing roles:
                           [ Incoming Client / User Traffic ]                                           â”‚                                           â–¼                                    [ HAProxy VIP ]                      â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”´â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”                      â”‚                                         â”‚                Port 6443 / 22623                         Port 80 / 443                      â”‚                                         â”‚                      â–¼                                         â–¼          â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”                 â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”          â”‚ Control Plane Nodes   â”‚                 â”‚ Compute / Worker Nodesâ”‚          â”‚ (master01, 02, 03)    â”‚                 â”‚ (worker01, worker02)  â”‚          â”‚   + Bootstrap (Temp)  â”‚                 â”‚                       â”‚          â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜                 â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜                      â”‚                                         â”‚                      â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¬â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜                                           â”‚                                           â–¼                               [ Local BIND DNS Server ]                                 (Forward & Reverse PTR)  
### Component Analysis


- **User-Provisioned Infrastructure (UPI):** A deployment model where the engineer manages server provisioning, disk partitioning, network routes, and load distribution rather than relying on an automated installer.

- **Ignition Configurations (`.ign`):** Low-level machine configurations consumed during the first boot by Red Hat Enterprise Linux CoreOS (RHCOS) in the initramfs stage. Ignition configures partition schemes, network interfaces, systemd units, and SSH keys.

- **The Bootstrap Node:** A temporary virtual or physical node acting as an operational "starter motor." It initiates a standalone Kubernetes control plane, coordinates the installation across permanent control plane nodes, transfers leadership, and is retired once etcd forms quorum.

- **HAProxy (Load Balancer):** Directs traffic across control plane and worker members, preventing single points of failure for management interfaces (API) and application workloads (Ingress Router).

- **API VIP & Apps VIP:**

  - **API VIP (`api.<cluster>.<domain>`):** Static ingress IP providing access to port `6443` (Kubernetes API server) for CLI commands and node check-ins.

  - **Apps VIP (`*.apps.<cluster>.<domain>`):** Wildcard ingress IP exposing HTTP/HTTPS traffic (`80`/`443`) to applications and the OpenShift Web Console.




- **BIND DNS:** Provides forward resolution and reverse PTR resolution for every node. Cluster nodes cannot form an etcd quorum or establish TLS sessions without functional bidirectional name resolution.



## 3. Hardware & Network Topology

### Node Allocation Matrix

Hostname
Role
IP Address
MAC Address / Interface
Specs (vCPU / RAM / Disk)

`helper.example.com`
Bastion / DNS / HAProxy / HTTP
`192.168.10.10`
`ens1f0`
4 vCPU, 16 GB, 100 GB HDD

`bootstrap.ocp-baremetal.example.com`
Temporary Bootstrap
`192.168.10.20`
`ens1f0`
4 vCPU, 16 GB, 120 GB SSD

`master01.ocp-baremetal.example.com`
Control Plane 1 / etcd
`192.168.10.21`
`ens1f0`
8 vCPU, 32 GB, 120 GB SSD

`master02.ocp-baremetal.example.com`
Control Plane 2 / etcd
`192.168.10.22`
`ens1f0`
8 vCPU, 32 GB, 120 GB SSD

`master03.ocp-baremetal.example.com`
Control Plane 3 / etcd
`192.168.10.23`
`ens1f0`
8 vCPU, 32 GB, 120 GB SSD

`worker01.ocp-baremetal.example.com`
Compute / Application Pods
`192.168.10.31`
`ens1f0`
8 vCPU, 32 GB, 120 GB SSD

`worker02.ocp-baremetal.example.com`
Compute / Application Pods
`192.168.10.32`
`ens1f0`
8 vCPU, 32 GB, 120 GB SSD


*Subnet: `192.168.10.0/24` | Gateway: `192.168.10.1` | Domain: `example.com` | Cluster Name: `ocp-baremetal`*

## 4. Implementation Phase

### Step 1: Base Network Configuration (DNS & HAProxy)

On the dedicated Helper/Bastion node (`192.168.10.10`), configure authoritative forward and reverse records using BIND (`named`).

#### Forward Zone File (`/var/named/ocp-baremetal.zone`)
 $TTL 1D @       IN SOA  helper.example.com. admin.example.com. (                                         2026100101      ; Serial                                         1D 1H 1W 3H ) @               IN      NS      helper.example.com.  ; Helper / VIPs helper          IN      A       192.168.10.10 api             IN      A       192.168.10.10 api-int         IN      A       192.168.10.10 *.apps          IN      A       192.168.10.10  ; Cluster Nodes bootstrap       IN      A       192.168.10.20 master01        IN      A       192.168.10.21 master02        IN      A       192.168.10.22 master03        IN      A       192.168.10.23 worker01        IN      A       192.168.10.31 worker02        IN      A       192.168.10.32  ; ETCD Cluster Records etcd-0          IN      A       192.168.10.21 etcd-1          IN      A       192.168.10.22 etcd-2          IN      A       192.168.10.23  _etcd-server-ssl._tcp IN SRV    0 10 2380 etcd-0 _etcd-server-ssl._tcp IN SRV    0 10 2380 etcd-1 _etcd-server-ssl._tcp IN SRV    0 10 2380 etcd-2  
#### Reverse Zone File (`/var/named/10.168.192.zone`)
 $TTL 1D @       IN SOA  helper.example.com. admin.example.com. (                                         2026100101                                         1D 1H 1W 3H ) @       IN      NS      helper.example.com.  10      IN      PTR     api.ocp-baremetal.example.com. 10      IN      PTR     api-int.ocp-baremetal.example.com. 20      IN      PTR     bootstrap.ocp-baremetal.example.com. 21      IN      PTR     master01.ocp-baremetal.example.com. 22      IN      PTR     master02.ocp-baremetal.example.com. 23      IN      PTR     master03.ocp-baremetal.example.com. 31      IN      PTR     worker01.ocp-baremetal.example.com. 32      IN      PTR     worker02.ocp-baremetal.example.com.  
#### HAProxy Configuration (`/etc/haproxy/haproxy.cfg`)
 frontend openshift-api-server     bind *:6443     default_backend openshift-api-server  backend openshift-api-server     balance roundrobin     server bootstrap 192.168.10.20:6443 check     server master01 192.168.10.21:6443 check     server master02 192.168.10.22:6443 check     server master03 192.168.10.23:6443 check  frontend machine-config-server     bind *:22623     default_backend machine-config-server  backend machine-config-server     balance roundrobin     server bootstrap 192.168.10.20:22623 check     server master01 192.168.10.21:22623 check     server master02 192.168.10.22:22623 check     server master03 192.168.10.23:22623 check  frontend ingress-https     bind *:443     default_backend ingress-https  backend ingress-https     balance roundrobin     server worker01 192.168.10.31:443 check     server worker02 192.168.10.32:443 check  
### Step 2: Cluster Manifest Creation & Ignition Generation


1. Initialize `install-config.yaml`:apiVersion: v1 baseDomain: example.com metadata:   name: ocp-baremetal compute: - name: worker   replicas: 2 controlPlane: - name: master   replicas: 3 platform:   none: {} pullSecret: '{"auths":{...}}' sshKey: 'ssh-ed25519 AAAAC3NzaC1...' networking:   networkType: OVNKubernetes   machineNetwork:   - cidr: 192.168.10.0/24  

2. Generate the machine configs and expose them over HTTP:mkdir ~/ocp-install && cd ~/ocp-install cp ~/install-config.yaml . openshift-install create ignition-configs --dir=. cp *.ign /var/www/html/ignition/ chmod 644 /var/www/html/ignition/*.ign  



### Step 3: Bare-Metal Provisioning via HP iLO 5


1. Connect to the **HP iLO 5** console of each server.

2. Mount the **RHCOS Live ISO** using Virtual Media.

3. Configure the HP Smart Array RAID controller (RAID 1 mirrored volume on `/dev/sda`).

4. Boot into the live environment and run `coreos-installer` with static Dracut kernel arguments:


 # Executed on Master 01 sudo coreos-installer install /dev/sda \   --ignition-url=http://192.168.10.10:8080/ignition/master.ign \   --append-karg "ip=192.168.10.21::192.168.10.1:255.255.255.0:master01.ocp-baremetal.example.com:ens1f0:none nameserver=192.168.10.10"  
Repeat this procedure for `bootstrap` (`192.168.10.20`), `master02`, `master03`, `worker01`, and `worker02`, updating IP addresses, hostnames, and corresponding ignition targets (`bootstrap.ign`, `master.ign`, `worker.ign`).

## 5. Post-Installation Lifecycle & Node Join

### 1. Bootstrap Coordination & Retirement

Track the bootstrap completion sequence from the bastion host:
 openshift-install wait-for bootstrap-complete --dir=~/ocp-install/ --log-level=info  
Upon verification:


- Decommission/power down the bootstrap host.

- Remove `server bootstrap` lines from `/etc/haproxy/haproxy.cfg`.

- Reload the service: `systemctl reload haproxy`.



### 2. Worker Node Joining (Certificate Signing Requests)

Worker nodes do not enter the cluster automatically. An administrator must approve their CSRs:
 export KUBECONFIG=~/ocp-install/auth/kubeconfig  # View and approve pending certificates oc get csr -o name | xargs -r oc adm certificate approve  
### 3. Verification of Cluster Operators

Monitor the cluster operators until all display `AVAILABLE=True` and `DEGRADED=False`:
 watch -d "oc get clusteroperators"  
Finalize the installation sequence:
 openshift-install wait-for install-complete --dir=~/ocp-install/  
## 6. Troubleshooting & Common Pitfalls

Problem Observed
Primary Cause
Remediation

**Nodes fail to retrieve `.ign` file**
Firewall blocking HTTP port `8080` or bad physical routing
Verify connectivity via `curl -I [http://192.168.10.10:8080/ignition/master.ign](http://192.168.10.10:8080/ignition/master.ign)` from live shell; ensure Apache has `chmod 644` permissions.

**`etcd` fails to form quorum**
Reverse DNS (PTR) records missing or incorrect time synchronization (NTP)
Validate with `dig -x 192.168.10.21`. Verify all HP Gen10 servers share the identical time source via iLO UEFI settings.

**`image-registry` operator degraded**
Bare-metal UPI lacks cloud-native block storage
Configure storage to use an external NFS share or patch as ephemeral: `oc patch configs.imageregistry.operator.openshift.io/cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}},"managementState":"Managed"}}'`.

**Worker nodes stuck in `NotReady`**
Serving certificates (Round 2 CSRs) remain unapproved
Re-run `oc get csr` and approve the second round of certificates generated by `system:node:<worker>`.


## 7. Conclusion & Learning Outcomes

Deploying OpenShift on bare-metal HP ProLiant Gen10 servers using the UPI approach provides a practical understanding of:


1. **Infrastructure Fundamentals:** How enterprise container platforms interact with underlying hardware, firmware, and storage arrays.

2. **Core Networking:** The critical dependency of Kubernetes platforms on split-horizon DNS, layer-4 load balancers, and static IP routing.

3. **Immutable Operating Systems:** How RHCOS uses declarative Ignition files to configure state before user-space initialization.



This lab-proven workflow forms the baseline for running self-hosted, highly available enterprise OpenShift clusters.