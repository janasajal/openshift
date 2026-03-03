# OpenShift Networking Hands-On Lab Guide
### Red Hat OpenShift Virtualization Administration — OCP v4.18

---

**Author:** Sajal Jana  
**Platform:** Red Hat OpenShift Virtualization Administration Rapid Track v4.18  
**Cluster:** 3-Node Compact Cluster (Masters acting as Workers)  
**CNI:** OVN-Kubernetes  

---

## Table of Contents

1. [Cluster Orientation](#1-cluster-orientation)
2. [CNI Plugin Discovery](#2-cni-plugin-discovery)
3. [Network CIDR Ranges](#3-network-cidr-ranges)
4. [Pod IP Assignment](#4-pod-ip-assignment)
5. [Same-Node Pod Communication](#5-same-node-pod-communication)
6. [Cross-Node Pod Communication](#6-cross-node-pod-communication)
7. [OVN-Kubernetes Tunnel Inspection](#7-ovn-kubernetes-tunnel-inspection)
8. [ClusterIP Service Networking](#8-clusterip-service-networking)
9. [NodePort Service](#9-nodeport-service)
10. [CoreDNS and Service Discovery](#10-coredns-and-service-discovery)
11. [OpenShift Routes and Ingress](#11-openshift-routes-and-ingress)
12. [Secure Routes with TLS](#12-secure-routes-with-tls)
13. [OpenShift Virtualization Networking](#13-openshift-virtualization-networking)
14. [Multus CNI and Secondary Networks](#14-multus-cni-and-secondary-networks)
15. [VM to VM Communication](#15-vm-to-vm-communication)
16. [NetworkPolicy and Zero Trust](#16-networkpolicy-and-zero-trust)

---

## 1. Cluster Orientation

### Concepts
Before touching anything network-related, orient to the cluster. Every node gets an internal IP address — these are used for node-to-node communication and form the physical underlay for OVN tunnels.

### Commands

```bash
# Login to cluster
oc login -u admin -p <password> https://api.ocp4.example.com:6443

# List nodes with IPs
oc get nodes -o wide
```

### Lab Output

```
NAME       STATUS   ROLES                         AGE    VERSION   INTERNAL-IP
master01   Ready    control-plane,master,worker   274d   v1.31.6   192.168.50.10
master02   Ready    control-plane,master,worker   274d   v1.31.6   192.168.50.11
master03   Ready    control-plane,master,worker   274d   v1.31.6   192.168.50.12
```

### Key Learnings
- 3-node compact cluster — masters act as workers (double duty!)
- Running OCP 4.18 on RHCOS with CRI-O container runtime
- No external IPs — private lab environment
- Node IPs: `192.168.50.10`, `.11`, `.12`

---

## 2. CNI Plugin Discovery

### Concepts
OpenShift uses a CNI (Container Network Interface) plugin as a traffic cop — it ensures packets are routed correctly between pods. OCP 4.18 uses **OVN-Kubernetes** (Open Virtual Network), which uses tunnels and flow tables to connect pods and VMs across nodes.

### Commands

```bash
oc get network.config/cluster -o yaml | grep -A5 networkType
```

### Lab Output

```yaml
networkType: OVNKubernetes
clusterNetwork:
serviceNetwork:
clusterNetworkMTU: 1400
```

### Key Learnings
- CNI Plugin: **OVNKubernetes** — the modern, powerful CNI since OCP 4.12+
- MTU is `1400` (reduced from 1500) because OVN wraps packets in **Geneve tunnel headers** (100 bytes overhead)
- Packets travel in disguise — the tunnel header is the "envelope inside an envelope"

---

## 3. Network CIDR Ranges

### Concepts
OpenShift uses pre-planned IP ranges (CIDRs). Every pod gets an IP from `clusterNetwork`, every Service gets an IP from `serviceNetwork`. Each node gets a `/23` subnet carved from the clusterNetwork.

### Commands

```bash
oc get network.config cluster -o yaml | grep -A3 -E 'clusterNetwork|serviceNetwork'
```

### Lab Output

```yaml
clusterNetwork:
  - cidr: 10.8.0.0/14
    hostPrefix: 23
serviceNetwork:
  - 172.30.0.0/16
clusterNetworkMTU: 1400
```

### Key Learnings

| Network | CIDR | Total IPs | Purpose |
|---------|------|-----------|---------|
| Pod Network | `10.8.0.0/14` | 262,144 | Every pod gets an IP from here |
| Service Network | `172.30.0.0/16` | 65,536 | Every ClusterIP Service lives here |
| Host Prefix | `/23` per node | 512/node | Each node's personal pod subnet |

- Node subnets discovered in lab:
  - `master01` → `10.10.0.0/23`
  - `master02` → `10.8.0.0/23`
  - `master03` → `10.9.0.0/23`

---

## 4. Pod IP Assignment

### Concepts
When a pod starts, OVN-Kubernetes immediately assigns it an IP from the node's `/23` subnet. The pod has no say — OVN-K is the welcome committee that hands out address stickers!

### Commands

```bash
# Create namespace
oc new-project network-lab

# Deploy test pod
oc create deployment netshoot --image=nicolaka/netshoot -- sleep infinity

# Check pod IP
oc get pods -o wide
```

### Lab Output

```
NAME                        READY   STATUS    IP           NODE
netshoot-755766b58d-nw7j7   1/1     Running   10.10.0.21   master01
```

### Key Learnings
- Pod got `10.10.0.21` — inside `master01`'s `/23` subnet (`10.10.0.0/23`) ✅
- IP assignment is instant and automatic — OVN-K keeps its promises
- `nicolaka/netshoot` is the Swiss Army knife of networking containers — comes with `ping`, `curl`, `dig`, `tcpdump`, `traceroute` etc.

---

## 5. Same-Node Pod Communication

### Concepts
When two pods are on the same node, traffic flows through OVN's virtual switch (`br-int`) without crossing any physical network boundaries. This is the fastest mode — pure local switching.

### Commands

```bash
# Deploy second pod
oc create deployment netshoot2 --image=nicolaka/netshoot -- sleep infinity

# Check both pods
oc get pods -o wide

# Ping from pod1 to pod2
oc exec -it netshoot-755766b58d-nw7j7 -- ping -c4 10.10.0.23
```

### Lab Output

```
NAME                         IP           NODE
netshoot-755766b58d-nw7j7    10.10.0.21   master01
netshoot2-6975cf8c9d-phpxh   10.10.0.23   master01

PING 10.10.0.23: 64 bytes, icmp_seq=1 ttl=64 time=1.30 ms
                 64 bytes, icmp_seq=2 ttl=64 time=0.859 ms
                 64 bytes, icmp_seq=3 ttl=64 time=0.128 ms
                 64 bytes, icmp_seq=4 ttl=64 time=0.145 ms
```

### Key Learnings
- TTL=64 — no extra hops, direct local switching
- Latency drops to sub-millisecond after ARP cache warms up
- First ping is slower (ARP table population), subsequent pings are lightning fast
- Traffic path: `Pod → OVN virtual switch → Pod` (all on same node)

---

## 6. Cross-Node Pod Communication

### Concepts
When pods are on different nodes, traffic must cross physical network boundaries through OVN's **Geneve tunnels**. OVN wraps pod packets inside tunnel packets — pods are completely unaware of this extra journey.

### Commands

```bash
# Force pod onto master02 using nodeSelector
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: netshoot3
  namespace: network-lab
spec:
  nodeSelector:
    kubernetes.io/hostname: master02
  containers:
  - name: netshoot3
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
EOF

# Check pod placement
oc get pods -o wide

# Cross-node ping
oc exec -it netshoot-755766b58d-nw7j7 -- ping -c4 10.8.0.6
```

### Lab Output

```
NAME          IP           NODE
netshoot      10.10.0.21   master01
netshoot2     10.10.0.23   master01
netshoot3     10.8.0.6     master02   <- Different node!

PING 10.8.0.6: ttl=62 time=5.28 ms
               ttl=62 time=6.58 ms
               ttl=62 time=0.515 ms
               ttl=62 time=0.984 ms
```

### Key Learnings

| Metric | Same Node | Cross Node | Reason |
|--------|-----------|------------|--------|
| Latency | `0.128ms` | `0.515ms` | Geneve tunnel overhead |
| TTL | `64` | `62` | 2 extra hops (tunnel entry + exit) |
| Packet Loss | `0%` | `0%` | OVN never drops! |

**What the Geneve tunnel does:**
```
Original Pod Packet:
[IP: 10.10.0.21 -> 10.8.0.6] [PING data]

Inside Geneve Tunnel:
[IP: 192.168.50.10 -> 192.168.50.11] [Geneve Header] [IP: 10.10.0.21 -> 10.8.0.6] [PING data]
     master01 physical IP -> master02 physical IP
```

---

## 7. OVN-Kubernetes Tunnel Inspection

### Concepts
OVN-Kubernetes builds a **full mesh tunnel topology** — every node has a direct Geneve tunnel to every other node. These tunnels are built using Open vSwitch (OVS) underneath OVN.

### Commands

```bash
# Find OVN pods
oc get pods -n openshift-ovn-kubernetes -o wide

# Inspect OVS tunnels on master01
oc exec -n openshift-ovn-kubernetes ovnkube-node-kbnmv \
  -c ovn-controller -- ovs-vsctl show
```

### Lab Output — Key Findings

```
Bridge br-int
    Port ovn-4fdd0c-0
        Interface ovn-4fdd0c-0
            type: geneve
            options: {local_ip="192.168.50.10", remote_ip="192.168.50.11"}  <- master02!
    Port ovn-687868-0
        Interface ovn-687868-0
            type: geneve
            options: {local_ip="192.168.50.10", remote_ip="192.168.50.12"}  <- master03!
Bridge br-ex
    Port ens3           <- Physical NIC
```

### Key Learnings

| Component | Purpose |
|-----------|---------|
| `br-int` | Internal OVN bridge — ALL pod traffic flows here |
| `br-ex` | External bridge — connects cluster to outside world |
| `ovn-4fdd0c-0` | Geneve tunnel to master02 |
| `ovn-687868-0` | Geneve tunnel to master03 |
| `ovn-k8s-mp0` | Management port — node joins pod network |
| Hex-named ports | Each running pod connected to OVS switch |

- Full mesh topology: `master01 <-> master02 <-> master03`
- OVN pods run on every node: `ovnkube-node` (local cop) + `ovnkube-control-plane` (HQ)

---

## 8. ClusterIP Service Networking

### Concepts
A **ClusterIP Service** is a virtual IP (VIP) that load balances traffic across multiple backend pods. The VIP doesn't exist on any real network interface — it lives only in OVN flow tables. OVN-K does **DNAT** (Destination NAT) to rewrite the destination to a real pod IP.

### Commands

```bash
# Deploy web app with 3 replicas
oc create deployment nginx-app \
  --image=registry.access.redhat.com/ubi9/httpd-24 \
  --replicas=3

# Create ClusterIP service
oc expose deployment nginx-app --port=8080

# Check service
oc get svc nginx-app

# Check real pod IPs behind the service
oc get endpoints nginx-app

# Test service from pod
oc exec -it netshoot-755766b58d-nw7j7 -- \
  curl -s http://172.30.100.16:8080 | head -5
```

### Lab Output

```
NAME        TYPE        CLUSTER-IP      PORT(S)
nginx-app   ClusterIP   172.30.100.16   8080/TCP   <- From 172.30.0.0/16! OK

ENDPOINTS: 10.10.0.35:8080, 10.8.0.8:8080, 10.9.0.6:8080
           master01         master02       master03
```

### Key Learnings
- `172.30.100.16` is from `serviceNetwork: 172.30.0.0/16` — as predicted!
- ClusterIP is VIRTUAL — exists only in OVN flow tables
- Endpoints object contains REAL pod IPs that get traffic
- If a pod dies, it is automatically removed from endpoints — traffic stops going there
- DNAT magic: `172.30.100.16:8080` → `10.10.0.35:8080` (one of the pods)

---

## 9. NodePort Service

### Concepts
A **NodePort Service** opens a port (30000-32767) on EVERY node simultaneously. Traffic hitting any node on that port gets forwarded to the service — even if the pod isn't on that node! OVN-K handles cross-node forwarding transparently.

### Commands

```bash
# Create NodePort service
oc expose deployment nginx-app \
  --name=nginx-nodeport \
  --type=NodePort \
  --port=8080

# Check port assigned
oc get svc nginx-nodeport

# Test from all 3 nodes
curl -s http://192.168.50.10:31983 | head -3
curl -s http://192.168.50.11:31983 | head -3
curl -s http://192.168.50.12:31983 | head -3
```

### Lab Output

```
NAME             TYPE       CLUSTER-IP       PORT(S)
nginx-nodeport   NodePort   172.30.146.179   8080:31983/TCP

# All 3 nodes responded with HTML!
```

### Key Learnings

| Service Type | Address | Accessible From |
|-------------|---------|----------------|
| ClusterIP | `172.30.100.16:8080` | Inside cluster only |
| NodePort | `192.168.50.x:31983` | Anywhere on network |

- NodePort ALSO gets a ClusterIP — two ways to reach the same pods
- Port range: 30000-32767 (auto-assigned by OpenShift)
- Cross-node forwarding is transparent — knock on any node's door, get served!

---

## 10. CoreDNS and Service Discovery

### Concepts
**CoreDNS** is the cluster's phone book. Instead of memorizing IPs, pods use service names. CoreDNS runs on every node as a DaemonSet and automatically resolves service names to ClusterIPs. Every pod gets DNS search domains injected automatically via `/etc/resolv.conf`.

### Commands

```bash
# Find CoreDNS pods
oc get pods -n openshift-dns -o wide

# DNS lookup from inside pod (short name)
oc exec -it netshoot-755766b58d-nw7j7 -- nslookup nginx-app

# DNS lookup across namespaces (full FQDN)
oc exec -it netshoot-755766b58d-nw7j7 -- \
  nslookup kubernetes.default.svc.cluster.local

# Check pod's DNS config
oc exec -it netshoot-755766b58d-nw7j7 -- cat /etc/resolv.conf
```

### Lab Output

```
# nslookup nginx-app
Server: 172.30.0.10
Name:   nginx-app.network-lab.svc.cluster.local
Address: 172.30.100.16

# nslookup kubernetes.default.svc.cluster.local
Name: kubernetes.default.svc.cluster.local
Address: 172.30.0.1   <- Always the 1st IP! Most important service!

# /etc/resolv.conf inside pod
search network-lab.svc.cluster.local svc.cluster.local cluster.local ocp4.example.com
nameserver 172.30.0.10
options ndots:5
```

### Key Learnings
- DNS server IP: `172.30.0.10` — always the 10th IP in service network range
- Kubernetes API server: `172.30.0.1` — always the 1st IP, most important service!
- DNS name format: `<service>.<namespace>.svc.cluster.local`
- Short names work because of `search` domains — `nginx-app` expands to `nginx-app.network-lab.svc.cluster.local`
- `ndots:5` — if name has fewer than 5 dots, try search domains first

**DNS Resolution Journey:**
```
Pod asks: "nginx-app"  (0 dots < ndots:5)
    -> tries: nginx-app.network-lab.svc.cluster.local -> FOUND!
    -> returns: 172.30.100.16
```

---

## 11. OpenShift Routes and Ingress

### Concepts
**OpenShift Routes** give applications a proper domain name via the **HAProxy Ingress Controller**. HAProxy runs on specific nodes and checks a routing table — when a request arrives for a hostname, it forwards to the right Service.

### Commands

```bash
# Find Ingress Controller pods
oc get pods -n openshift-ingress -o wide

# Check base domain
oc get ingresscontroller default -n openshift-ingress-operator \
  -o yaml | grep -A5 -E 'domain|replicas'

# Create Route
oc expose service nginx-app

# Check route
oc get route nginx-app

# Test via domain name
curl -s http://nginx-app-network-lab.apps.ocp4.example.com | head -5
```

### Lab Output

```
# Ingress Controller
router-default-glwsz   Running   192.168.50.11   master02
router-default-htdcq   Running   192.168.50.12   master03

# IngressController config
domain: apps.ocp4.example.com
replicas: 2
httpPort: 80
httpsPort: 443

# Route created
NAME        HOST/PORT                                     TERMINATION
nginx-app   nginx-app-network-lab.apps.ocp4.example.com  (none)
```

### Key Learnings
- Base domain: `apps.ocp4.example.com`
- Route hostname pattern: `<service>-<namespace>.apps.<cluster-domain>`
- HAProxy runs on **host network** (`192.168.50.x`) to catch traffic before the pod network
- `endpointPublishingStrategy: hostNetwork` — router uses node's physical NIC
- Traffic flow: `Browser -> HAProxy (port 80) -> ClusterIP -> Pod`

---

## 12. Secure Routes with TLS

### Concepts
OpenShift supports 3 TLS termination modes:

| Mode | Path | Use Case |
|------|------|---------|
| Edge | `[HTTPS] -> HAProxy -> [HTTP] -> Pod` | Most common, HAProxy handles SSL |
| Passthrough | `[HTTPS] -> HAProxy -> [HTTPS] -> Pod` | Pod handles its own SSL |
| Re-encrypt | `[HTTPS] -> HAProxy -> [HTTPS] -> Pod` | Double encryption |

### Commands

```bash
# Create secure edge route with HTTP redirect
oc create route edge nginx-secure \
  --service=nginx-app \
  --port=8080 \
  --insecure-policy=Redirect

# Check both routes
oc get routes

# Test HTTPS
curl -sk https://nginx-secure-network-lab.apps.ocp4.example.com | head -5

# Test HTTP redirect
curl -v http://nginx-secure-network-lab.apps.ocp4.example.com 2>&1 | grep -E 'Location|HTTP/'
```

### Lab Output

```
NAME           HOST/PORT                                        TERMINATION
nginx-app      nginx-app-network-lab.apps.ocp4.example.com     (none)
nginx-secure   nginx-secure-network-lab.apps.ocp4.example.com  edge/Redirect

# HTTPS - HTML returned!
# HTTP redirect - 302 Found -> https://...
```

### Key Learnings
- `edge/Redirect` = TLS terminates at HAProxy + HTTP auto-redirects to HTTPS
- No certificates needed — OpenShift uses built-in `*.apps.ocp4.example.com` wildcard cert
- `-k` flag in curl = ignore cert warnings (lab use only, never in production!)
- Complete traffic flow:
```
Browser HTTP  -> HAProxy -> 302 Redirect -> Browser HTTPS
Browser HTTPS -> HAProxy (decrypts) -> ClusterIP (HTTP) -> Pod
```

---

## 13. OpenShift Virtualization Networking

### Concepts
In OpenShift Virtualization, a VM runs **inside a pod** called `virt-launcher`. The VM thinks it has real hardware but is actually running as QEMU/KVM inside a container. The VM connects to the pod network via **PASST** (Plug A Simple Socket Transport) binding in OCP 4.18.

### VM Architecture
```
virt-launcher POD (Kubernetes sees this)
  +-- QEMU/KVM Virtual Machine (VM sees its own OS)
        +-- eth0 -> PASST socket -> pod network (OVN-K)
```

### Commands

```bash
# Verify OpenShift Virtualization operator
oc get csv -n openshift-cnv | grep -i kubevirt

# Check available disk images
oc get datasource -n openshift-virtualization-os-images

# Deploy VM with cloud-init for password
cat <<EOF | oc apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel9-networklab
  namespace: network-lab
spec:
  running: true
  template:
    spec:
      domain:
        cpu:
          cores: 1
        memory:
          guest: 1Gi
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
            model: virtio
      networks:
      - name: default
        pod: {}
      volumes:
      - name: rootdisk
        dataVolume:
          name: rhel9-networklab-rootdisk
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            user: cloud-user
            password: redhat
            chpasswd:
              expire: false
            ssh_pwauth: true
  dataVolumeTemplates:
  - metadata:
      name: rhel9-networklab-rootdisk
    spec:
      sourceRef:
        kind: DataSource
        name: rhel9
        namespace: openshift-virtualization-os-images
      storage:
        resources:
          requests:
            storage: 30Gi
EOF

# Check VM status
oc get vm -n network-lab

# Find virt-launcher pod
oc get pods -n network-lab -o wide | grep virt-launcher

# Check VM network interfaces
oc get vmi rhel9-networklab -n network-lab \
  -o jsonpath='{.status.interfaces}' | python3 -m json.tool

# Test pod-to-VM communication
oc exec -it netshoot-755766b58d-nw7j7 -- ping -c4 10.10.0.44

# Access VM console (login: cloud-user / redhat)
virtctl console rhel9-networklab -n network-lab
```

### Lab Output

```json
[{
  "interfaceName": "eth0",
  "ipAddress": "10.10.0.44",
  "mac": "02:0f:26:00:00:01",
  "name": "default",
  "infoSource": "domain, guest-agent"
}]
```

```
# Pod to VM ping
64 bytes from 10.10.0.44: ttl=63 time=2.34 ms
64 bytes from 10.10.0.44: ttl=63 time=1.06 ms
64 bytes from 10.10.0.44: ttl=63 time=0.363 ms
0% packet loss
```

### Key Learnings
- VM runs inside `virt-launcher` pod — Kubernetes sees a pod, the VM sees hardware!
- VM gets the **actual pod IP** via PASST binding (OCP 4.18+)
- From inside the VM: `eth0` shows `10.0.2.2` (masquerade internal), external view: `10.10.0.47`
- **kubemacpool** assigns sequential MAC addresses: `02:0f:26:00:00:01`, `...02`, etc.
- TTL=63 for pod-to-VM (one extra hop due to PASST/virtio stack vs pod-to-pod TTL=64)
- `virtctl console` = direct keyboard access to VM (works without network)
- Use `cloud-init` `userData` to set VM passwords reliably

**VM Department Key Components:**

| Component | Role |
|-----------|------|
| `virt-operator` | Manages entire KubeVirt lifecycle |
| `virt-api` | Handles all VM API requests |
| `virt-controller` | Creates/deletes/manages VM pods |
| `virt-handler` | Runs on every node, manages local VMs |
| `cdi-*` | Imports/uploads VM disk images |
| `kubemacpool` | Assigns unique MAC addresses to VMs |
| `passt-binding-cni` | Connects VM to pod network |
| `bridge-marker` | Discovers bridge networks on nodes |

**Available Disk Images in Lab:**
`rhel7`, `rhel8`, `rhel9`, `rhel10`, `centos-stream9`, `centos-stream10`, `fedora`, `win10`, `win11`, `win2k16`, `win2k19`, `win2k22`, `win2k25`

---

## 14. Multus CNI and Secondary Networks

### Concepts
**Multus CNI** is a meta-CNI plugin that allows pods and VMs to have **multiple network interfaces**. It reads network annotations and calls multiple CNI plugins simultaneously — one per interface. A **NetworkAttachmentDefinition (NAD)** is the blueprint that tells Multus how to configure each secondary network.

### Architecture
```
Without Multus:           With Multus:
VM                        VM
+-- eth0 (pod network)    +-- eth0 (pod network - OVN-K)
                          +-- eth1 (secondary - Linux Bridge/VLAN)
```

### Commands

```bash
# Verify Multus is running
oc get pod -n openshift-multus -o wide

# Check node interfaces for available NICs
oc debug node/master01 -- chroot /host ip link show 2>/dev/null

# Create Linux bridge on all nodes (ens4 was free)
for node in master01 master02 master03; do
  oc debug node/$node -- chroot /host bash -c "
    ip link add br1 type bridge 2>/dev/null || true
    ip link set ens4 master br1
    ip link set br1 up
    ip link set ens4 up
    echo 'Bridge ready on $node'
    ip link show br1
  " 2>/dev/null
done

# Create NetworkAttachmentDefinition
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: br1-network
  namespace: network-lab
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "br1-network",
    "type": "cnv-bridge",
    "bridge": "br1",
    "macspoofchk": false,
    "ipam": {}
  }'
EOF

# Verify NAD created
oc get network-attachment-definitions -n network-lab

# Patch VM to add secondary NIC (add to interfaces and networks sections)
# Then restart VM
virtctl restart rhel9-networklab -n network-lab

# Verify dual NIC
oc get vmi rhel9-networklab -n network-lab \
  -o jsonpath='{.status.interfaces}' | python3 -m json.tool
```

### Lab Output

```json
[
  {
    "interfaceName": "eth0",
    "ipAddress": "10.10.0.47",
    "mac": "02:0f:26:00:00:01",
    "name": "default",
    "infoSource": "domain, guest-agent"
  },
  {
    "interfaceName": "eth1",
    "ipAddress": "192.168.51.100",
    "ipAddresses": ["192.168.51.100", "fe80::5a4c:1f94:3cc3:862c"],
    "mac": "02:0f:26:00:00:02",
    "name": "secondary",
    "infoSource": "domain, guest-agent, multus-status"
  }
]
```

### Key Learnings

| Component | Purpose |
|-----------|---------|
| `multus` DaemonSet | Main CNI multiplexer (one per node) |
| `multus-additional-cni-plugins` | Plugin library (bridge, VLAN, MACVLAN) |
| `multus-admission-controller` | Validates NetworkAttachmentDefinitions |
| `bridge-marker` | Discovers Linux bridges on nodes |
| `kube-cni-linux-bridge-plugin` | Plugs VMs into bridge networks |
| `kubemacpool` | Assigns unique MACs (sequential: `...01`, `...02`) |

- Secondary NIC (`eth1`) got `192.168.51.100` — a REAL physical network IP from `ens4`!
- `infoSource: multus-status` on `eth1` — Multus itself confirms the interface
- Bridge network has NO NAT — IP is same from inside AND outside VM
- Linux bridge created via `oc debug node` is **temporary** (lost on reboot)
- For production: use **NMState operator** to make bridges persistent

**Network Isolation:**
```
Pod network (10.x.x.x)      -> Cannot reach bridge network (192.168.51.x)
Workstation (192.168.50.x)  -> Cannot reach bridge network (192.168.51.x)
Bridge = isolated L2 segment on ens4
```

---

## 15. VM to VM Communication

### Concepts
Two VMs can communicate over two independent networks simultaneously — the **pod network** (OVN-K managed, masquerade NAT) and the **bridge network** (direct L2, no NAT, pure physical switching).

### Commands

```bash
# Get VM2 network interfaces
oc get vmi rhel9-networklab2 -n network-lab \
  -o jsonpath='{.status.interfaces}' | python3 -m json.tool

# Access VM1 console
virtctl console rhel9-networklab -n network-lab
# Login: cloud-user / redhat

# Run all tests from inside VM1
ip route show

# Bridge network VM to VM
ping -c3 192.168.51.101

# Pod network VM to VM
ping -c3 10.10.0.50
```

### Lab Output

```
# VM Network Map
VM1 eth0: 10.10.0.47  (pod network)    VM2 eth0: 10.10.0.50  (pod network)
VM1 eth1: 192.168.51.100 (bridge)      VM2 eth1: 192.168.51.101 (bridge)

# Routing table inside VM1
default via 10.0.2.1 dev eth0       <- Masquerade gateway (KubeVirt)
10.0.2.0/24 dev eth0                <- Pod network (masqueraded as 10.0.2.2)
192.168.51.0/24 dev eth1            <- Bridge network (direct, no NAT!)

# Bridge network test (VM1 eth1 -> VM2 eth1)
64 bytes from 192.168.51.101: time=0.378 ms
64 bytes from 192.168.51.101: time=0.419 ms
64 bytes from 192.168.51.101: time=0.428 ms
0% packet loss

# Pod network test (VM1 eth0 -> VM2 eth0)
64 bytes from 10.10.0.50: time=14.1 ms  <- First ping (masquerade NAT warmup)
64 bytes from 10.10.0.50: time=1.14 ms
64 bytes from 10.10.0.50: time=0.512 ms
0% packet loss
```

### Key Learnings

| Network | Path | Latency | NAT Hops | Best For |
|---------|------|---------|----------|---------|
| Bridge (eth1) | VM -> br1 -> ens4 -> VM | ~0.4ms | 0 | Legacy apps, direct L2 |
| Pod (eth0) | VM -> masquerade -> OVN -> masquerade -> VM | ~0.5ms | 4 | Kubernetes services |

- Bridge: TTL=64, zero extra hops, pure L2 switching — fastest!
- Pod: TTL=62 — crosses 4 NAT hops (VM1-NAT-in + OVN + OVN + VM2-NAT-in)
- Both networks: 0% packet loss
- **Masquerade revelation:** Inside VM, `eth0` shows `10.0.2.2` — KubeVirt NATs between `10.0.2.2` (VM internal) and `10.10.0.47` (pod external) invisibly
- **Real world use:** VM keeps physical network IP (`192.168.51.x`) for legacy apps WHILE getting pod IP for Kubernetes service integration

---

## 16. NetworkPolicy and Zero Trust

### Concepts
By default, ALL pods can communicate with ALL other pods — zero restrictions. **NetworkPolicy** implements zero-trust networking with explicit allow rules. Without a matching allow rule, ALL traffic is blocked. OVN-K enforces policies at the virtual switch level — packets are dropped before they even leave the source pod.

**Golden Rule:** Start with `deny-all`, then selectively allow only what's needed.

### Commands

```bash
# STEP 1: Document open access (BEFORE state)
oc exec -it netshoot-755766b58d-nw7j7 -n network-lab -- \
  curl -s --max-time 5 http://172.30.100.16:8080 | grep title

# STEP 2: Apply deny-all policy
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: network-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# STEP 3: Verify traffic is blocked
oc exec -it netshoot-755766b58d-nw7j7 -n network-lab -- \
  curl -s --max-time 5 http://172.30.100.16:8080
# Result: exit code 28 (timeout - blocked!)

# STEP 4: Apply selective allow policies
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-netshoot-http-egress
  namespace: network-lab
spec:
  podSelector:
    matchLabels:
      app: netshoot
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: nginx-app
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-ingress-from-netshoot
  namespace: network-lab
spec:
  podSelector:
    matchLabels:
      app: nginx-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: netshoot
    ports:
    - protocol: TCP
      port: 8080
EOF

# STEP 5: Verify selective access
oc get networkpolicy -n network-lab
```

### Lab Results — Zero Trust Verified!

| Test | Before Policy | After deny-all | After Selective Allow |
|------|--------------|----------------|----------------------|
| DNS resolution | Works | Blocked | Works |
| HTTP to nginx | Works | Blocked (exit 28) | Works |
| Ping to VM | Works | Blocked (100% loss) | Blocked (correct!) |

### Key Learnings
- `podSelector: {}` = applies to ALL pods in namespace (empty = match all)
- `policyTypes: [Ingress, Egress]` = controls both incoming AND outgoing traffic
- `exit code 28` = curl timeout — OVN-K drops packet silently at virtual switch level
- **DNS gotcha:** DNS requires BOTH egress (query outgoing) AND ingress (response incoming)
- `namespaceSelector` with `kubernetes.io/metadata.name` = reach pods in other namespaces
- Multiple NetworkPolicies use **OR logic** — any matching policy allows traffic

**Important DNS Policy Note:**
```yaml
# Always allow DNS from openshift-dns namespace
egress:
- to:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: openshift-dns
  ports:
  - protocol: UDP
    port: 53
```

**NetworkPolicy Rules Summary:**
- No policy applied = allow all traffic (dangerous in production!)
- Any policy applied = implicit deny for all non-matching traffic
- Always test DNS separately after applying NetworkPolicy
- Use `--max-time 5` with curl to quickly detect blocked connections

---

## Complete Networking Architecture

```
External User / Browser
        |
        v  https://nginx-secure-network-lab.apps.ocp4.example.com
+-----------------------------------------------+
|   HAProxy Router (192.168.50.11 / .12)        |  <- Routes / Ingress
|   Port 80 (HTTP redirect) / Port 443 (HTTPS)  |
+-----------------------------------------------+
        |  ClusterIP 172.30.100.16:8080
        v
+-----------------------------------------------+
|   CoreDNS (172.30.0.10)                       |  <- Service Discovery
|   Service / Endpoints (DNAT)                  |
+-----------------------------------------------+
        |  Pod IP 10.x.x.x
        v
+-----------------------------------------------+
|   OVN-K Virtual Switch (br-int)               |  <- CNI / Tunnels
|   Geneve Tunnels between nodes                |
+-----------------------------------------------+
        |
   +----+----+
   |         |
   v         v
Pod/VM    VM eth1 -> br1 -> ens4 -> Physical Network
           192.168.51.x (secondary - Multus)
```

---

## Complete Reference Tables

### Network Addresses in This Lab

| IP / Range | Purpose |
|------------|---------|
| `192.168.50.10-12` | Node physical IPs (management network) |
| `10.8.0.0/14` | Pod network CIDR (total cluster) |
| `10.10.0.0/23` | master01 pod subnet (512 IPs) |
| `10.8.0.0/23` | master02 pod subnet (512 IPs) |
| `10.9.0.0/23` | master03 pod subnet (512 IPs) |
| `172.30.0.0/16` | Service network CIDR |
| `172.30.0.1` | Kubernetes API service (always first!) |
| `172.30.0.10` | CoreDNS service (always 10th!) |
| `172.30.100.16` | nginx-app ClusterIP |
| `172.30.146.179` | nginx-nodeport ClusterIP |
| `192.168.51.100` | VM1 secondary NIC (bridge network) |
| `192.168.51.101` | VM2 secondary NIC (bridge network) |

### Access Methods Comparison

| Method | Address | Accessible From | Use Case |
|--------|---------|----------------|---------|
| Pod IP (direct) | `10.10.0.35:8080` | Inside cluster | Never in production |
| ClusterIP | `172.30.100.16:8080` | Inside cluster | Service-to-service |
| NodePort | `192.168.50.10:31983` | Physical network | Special cases only |
| Route HTTP | `nginx-app-network-lab.apps.ocp4.example.com` | Anywhere | External access |
| Route HTTPS | `nginx-secure-network-lab.apps.ocp4.example.com` | Anywhere | Production standard |
| VM Bridge NIC | `192.168.51.100` | Physical L2 segment | Legacy VM workloads |

### CLI Tools Reference

| Tool | Purpose | Example |
|------|---------|---------|
| `oc get pods -o wide` | See pod IPs and node placement | Pod debugging |
| `oc exec -it POD -- ping` | Test ICMP connectivity | Network debugging |
| `oc exec -it POD -- curl` | Test HTTP services | Service testing |
| `oc exec -it POD -- nslookup` | Test DNS resolution | DNS debugging |
| `oc exec -it POD -- cat /etc/resolv.conf` | Check DNS config | DNS config |
| `oc debug node/NODE` | Temporary shell on node | Node inspection |
| `oc get endpoints SVC` | See real pod IPs behind service | Service debugging |
| `oc get networkpolicy` | List NetworkPolicies | Security audit |
| `virtctl console VM` | VM serial console access | VM access |
| `virtctl ssh user@VM` | SSH into VM | VM access |
| `virtctl restart VM` | Graceful VM restart | VM operations |
| `ovs-vsctl show` | Show OVS bridges and tunnels | OVN debugging |
| `virsh list --all` | List VMs on node | VM inspection |
| `ip link show` | Show all network interfaces | Interface debugging |
| `ip route show` | Show routing table | Route debugging |

---

*Document prepared for educational purposes as part of the Red Hat OpenShift Virtualization Administration Rapid Track v4.18 hands-on lab.*

**Author: Sajal Jana**
