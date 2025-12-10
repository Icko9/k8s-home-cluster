# Kubernetes Home Cluster - Architecture Overview

## 📋 Executive Summary

This document describes the architecture of a 3-node Kubernetes homelab cluster running on 2 physical laptops. The cluster provides a self-hosted platform for running applications like Ghost (blogging), BookStack (wiki/documentation), AdGuard Home (DNS/ad blocking), and other services with persistent storage via Longhorn.

**Cluster Type:** On-premises homelab  
**Kubernetes Version:** v1.34.2  
**CNI:** Calico (VXLAN encapsulation, BGP disabled)  
**Storage:** Longhorn (distributed block storage)  
**External Access:** Cloudflare Tunnel

> **Note:** Replace all IP addresses (192.168.1.x) with your actual network addresses when replicating this setup.

---

## 🖥️ Hardware Overview

### Physical Hardware Summary

| Physical Machine | Hostname(s) | Role(s) | Specs |
|-----------------|-------------|---------|-------|
| **ASUS VivoBook** | `master` (VM) + `asuslpt` (host) | Control Plane + Worker | Intel i3 2.2GHz, 8GB RAM, 512GB SSD |
| **Dell Precision 5540** | `precision5540` | Worker | Intel i7-9850H, 16GB RAM, NVMe SSD |

> **Key Insight:** The ASUS VivoBook does double duty — it hosts a VM running the Kubernetes master node while simultaneously participating as a worker node. This gives us 3 logical Kubernetes nodes on just 2 physical laptops.

---

## 🖥️ Node Details

### Node 1: Control Plane (VM on ASUS VivoBook)

| Component | Specification |
|-----------|--------------|
| **Hostname** | `master` |
| **Type** | Virtual Machine |
| **Host** | ASUS VivoBook (`asuslpt`) |
| **CPU** | Virtual CPUs (allocated from host i3) |
| **RAM** | Virtual RAM (allocated from host's 8GB) |
| **Storage** | Virtual disk on host |
| **Network** | Bridge: `vnet1` (VM) + `eth01` (physical) |
| **IP Address** | `192.168.1.50` |
| **Interface** | `enp1s0` |
| **Role** | Control Plane (single master) |
| **OS** | Ubuntu Server 24.04 |

**Notes:**
- Runs as a KVM/QEMU VM on the ASUS VivoBook
- Bridge networking allows VM to appear directly on the LAN
- Shares physical resources with the host worker node

---

### Node 2: Worker (ASUS VivoBook - Same Physical Machine as Master VM)

| Component | Specification |
|-----------|--------------|
| **Hostname** | `asuslpt` |
| **Type** | Physical host (bare metal) |
| **Hardware** | ASUS VivoBook |
| **CPU** | Intel Core i3 @ 2.2GHz (4 cores) |
| **RAM** | 8 GB (shared with master VM) |
| **Storage** | 512GB SSD |
| **Network** | USB-to-Ethernet adapter |
| **IP Address** | `192.168.1.104` |
| **Interface** | `eth01` |
| **Role** | Worker Node + VM Host |
| **OS** | Ubuntu Server 24.04 |

**Notes:**
- This is the **same physical laptop** that hosts the master VM
- Runs kubelet directly on the host OS (not in the VM)
- Resources are shared between host worker processes and master VM
- Connected to router via USB-to-Ethernet adapter

---

### Node 3: Worker (Dell Precision 5540)

| Component | Specification |
|-----------|--------------|
| **Hostname** | `precision5540` |
| **Type** | Physical host (bare metal) |
| **Hardware** | Dell Precision 5540 |
| **CPU** | Intel Core i7-9850H (6 cores, 12 threads) |
| **RAM** | 16 GB |
| **Storage** | NVMe SSD |
| **Network** | WiFi |
| **IP Address** | `192.168.1.105` |
| **Interface** | `wlp59s0` (WiFi) |
| **Role** | Worker Node |
| **OS** | Ubuntu Server 24.04 |

**Notes:**
- Higher-end workstation laptop with best specs in the cluster
- Connected via WiFi (requires special networking configuration)
- Preferred node for resource-intensive workloads (Ghost, BookStack)
- See [WiFi Node Debugging Story](../05-troubleshooting/WiFi-Node-Kubernetes-Debugging-StoryBGPDisabled.md) for networking challenges

---

## 🌐 Network Topology

### Physical & Logical Network

```
                         [Home Router]
                         192.168.1.1
                              |
           +------------------+------------------+
           |                                     |
      [Ethernet]                              [WiFi]
           |                                     |
   ┌───────┴───────┐                    ┌────────┴────────┐
   │ ASUS VivoBook │                    │ Dell Precision  │
   │   (1 laptop)  │                    │     5540        │
   │               │                    │                 │
   │  ┌─────────┐  │                    │                 │
   │  │  VM:    │  │                    │                 │
   │  │ master  │  │                    │                 │
   │  │ .50     │  │                    │                 │
   │  └────┬────┘  │                    │                 │
   │       │       │                    │                 │
   │   [Bridge]    │                    │                 │
   │  vnet1+eth01  │                    │                 │
   │       │       │                    │                 │
   │  Host: asuslpt│                    │  precision5540  │
   │     .104      │                    │      .105       │
   └───────────────┘                    └─────────────────┘

   Legend:
   .50  = 192.168.1.50  (master VM)
   .104 = 192.168.1.104 (asuslpt host)
   .105 = 192.168.1.105 (precision5540)
```

### Network Configuration

| Network Segment | CIDR | Description |
|----------------|------|-------------|
| **Home Network** | `192.168.1.0/24` | Physical LAN (router subnet) |
| **Pod Network** | `192.168.0.0/16` | Kubernetes pod IP range |
| **Service Network** | `10.96.0.0/12` | Kubernetes service IP range |

### Node Network Summary

| Node | IP | Type | Connection | Notes |
|------|-----|------|------------|-------|
| `master` | `192.168.1.50` | VM | Bridge (vnet1 + eth01) | Control plane |
| `asuslpt` | `192.168.1.104` | Physical | USB-to-Ethernet | Worker + hosts master VM |
| `precision5540` | `192.168.1.105` | Physical | WiFi | Worker (requires MASQUERADE) |

---

## 🛠️ Software Stack

### Operating System

- **All Nodes:** Ubuntu Server 24.04 LTS

### Container Runtime

- **Runtime:** containerd
- **CRI:** Kubernetes Container Runtime Interface

### Kubernetes Components

| Component | Version | Notes |
|-----------|---------|-------|
| **Kubernetes** | v1.34.2 | Cluster version |
| **kubelet** | v1.34.2 | Node agent |
| **kube-proxy** | v1.34.2 | Service proxy |
| **CoreDNS** | (bundled) | Cluster DNS |

### CNI (Container Network Interface)

- **CNI:** Calico
- **Encapsulation:** VXLAN (UDP port 4789)
- **BGP:** Disabled (WiFi node compatibility)
- **Pod CIDR:** `192.168.0.0/16`

**Special Configuration for WiFi Node:**
```bash
# Required on precision5540 for pod traffic routing
iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

See [WiFi Node Debugging Story](../05-troubleshooting/WiFi-Node-Kubernetes-Debugging-StoryBGPDisabled.md) for details.

### Storage

- **Storage Class:** Longhorn
- **Type:** Distributed block storage
- **Replication:** 3 replicas across nodes
- **Provisioner:** `driver.longhorn.io`

### Package Management

- **Helm:** v3.x
- **Charts:** Custom + community (k8s-at-home, cloudflare, mariadb-operator)

### Additional Software

- **Tailscale VPN:** Remote access to all nodes
- **Cloudflare Tunnel:** External HTTPS access to services

---

## 📦 Applications & Services

### Ghost (Blogging Platform)

| Property | Value |
|----------|-------|
| **Namespace** | `ghost` |
| **Chart** | Custom Helm chart (`apps/ghost/`) |
| **Image** | `ghost:5.85.0` |
| **Database** | SQLite (built-in) |
| **Storage** | Longhorn (10Gi PVC) |
| **Service Type** | ClusterIP |
| **Port** | 2368 |
| **External URL** | `https://www.balkandevops.com` |
| **Node** | `precision5540` (via nodeSelector) |
| **Access** | Cloudflare Tunnel → Ingress → Service |

**Documentation:** [Ghost Kubernetes Setup](../Ghost-Kubernetes-Setup.md)

---

### BookStack (Documentation/Wiki)

| Property | Value |
|----------|-------|
| **Namespace** | `bookstack` |
| **Chart** | Community Helm chart |
| **Database** | MariaDB (via MariaDB Operator) |
| **Storage** | Longhorn PVC |
| **Service Type** | ClusterIP |
| **External URL** | *(configure via Cloudflare Tunnel)* |
| **Purpose** | Internal documentation and knowledge base |

---

### MariaDB Operator

| Property | Value |
|----------|-------|
| **Namespace** | `mariadb-operator` |
| **Chart** | `mariadb-operator/mariadb-operator` |
| **Purpose** | Manages MariaDB instances for applications |
| **Used By** | BookStack |

---

### AdGuard Home (DNS & Ad Blocking)

| Property | Value |
|----------|-------|
| **Namespace** | `adguard` |
| **Chart** | `k8s-at-home/adguard-home` |
| **Version** | v0.107.7 |
| **DNS Port** | 53 (UDP/TCP) via hostPort |
| **Web UI Port** | 30646 (NodePort) |
| **Node IP** | `192.168.1.104` (asuslpt) |
| **Access** | `http://192.168.1.104:30646` |

**Configuration:**
- Network-wide DNS-based ad blocking
- Router DHCP configured to use AdGuard Home as DNS
- Blocks ads and trackers for all devices on the network

**Documentation:** [AdGuard Home Kubernetes Setup](../AdGuard-Home-Kubernetes-Setup.md)

---

### Cloudflare Tunnel

| Property | Value |
|----------|-------|
| **Namespace** | `cloudflare` |
| **Chart** | `cloudflare/cloudflare-tunnel` |
| **Tunnel Name** | `matrica` |
| **Tunnel ID** | `d4d7f917-3152-4d10-aaf6-4b2e62743fc6` |

**Routes:**
| Domain | Backend |
|--------|---------|
| `balkandevops.com` | Homepage (dashboard) |
| `blog.balkandevops.com` | Ghost |
| `bookstack.balkandevops.com` | BookStack |
| `me.balkandevops.com` | CV/Portfolio |
| `grafana.balkandevops.com` | Grafana |

**Benefits:**
- No router port forwarding needed
- Automatic HTTPS/TLS
- DDoS protection
- Works behind NAT/CGNAT

**Documentation:** [Cloudflare Tunnel Troubleshooting](../Cloudflare-Tunnel-Troubleshooting.md)

---

### Longhorn (Distributed Storage)

| Property | Value |
|----------|-------|
| **Namespace** | `longhorn-system` |
| **Type** | Distributed block storage |
| **Replication** | 3 replicas |
| **Storage Class** | `longhorn` (default) |

**Configuration:**
- Provides persistent volumes for all applications
- Data replicated across nodes for redundancy
- Web UI available for volume management

---

## 🔧 Cluster Configuration

### Node Roles & Scheduling

| Node | Roles | Taints | Preferred Workloads |
|------|-------|--------|---------------------|
| `master` | `control-plane` | `NoSchedule` | Control plane only |
| `asuslpt` | `worker` | None | AdGuard Home, lightweight pods |
| `precision5540` | `worker` | None | Ghost, BookStack, heavy workloads |

### Resource Allocation

| Node | CPU | RAM | Storage | Notes |
|------|-----|-----|---------|-------|
| `master` | Virtual (shared) | Virtual (shared) | Virtual disk | Shares with asuslpt |
| `asuslpt` | i3 2.2GHz (4 cores) | 8GB (shared with VM) | 512GB SSD | Limited resources |
| `precision5540` | i7-9850H (6c/12t) | 16GB | NVMe SSD | Best for heavy workloads |

---

## 💾 Storage Architecture

### Storage Classes

| Storage Class | Provisioner | Type | Default |
|---------------|-------------|------|---------|
| `longhorn` | `driver.longhorn.io` | Distributed block | Yes |

### Persistent Volumes

| Application | PVC | Size | Storage Class | Node Affinity |
|-------------|-----|------|---------------|---------------|
| Ghost | `ghost` | 10Gi | `longhorn` | `precision5540` |
| BookStack | `bookstack-data` | 5Gi | `longhorn` | - |
| MariaDB | `mariadb-data` | 10Gi | `longhorn` | - |

---

## 🌍 Networking Architecture

### Calico Configuration

```yaml
calicoNetwork:
  bgp: Disabled
  ipPools:
  - cidr: 192.168.0.0/16
    encapsulation: VXLAN
    natOutgoing: Enabled
  nodeAddressAutodetectionV4:
    skipInterface: tailscale0
```

**Why VXLAN?** UDP-based encapsulation works reliably over WiFi (unlike IPIP).

**Why BGP Disabled?** BGP peering fails on WiFi nodes due to routing issues.

### WiFi Node Special Configuration

The `precision5540` node requires iptables MASQUERADE:

```bash
# Pod traffic must appear to come from node IP
iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

**Permanent setup via systemd:**
```bash
# /etc/systemd/system/k8s-masquerade.service
[Unit]
Description=Masquerade for Kubernetes pods on WiFi
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

## 🌐 External Access

### Cloudflare Tunnel

| Domain | Backend Service | Port |
|--------|-----------------|------|
| `balkandevops.com` | `homepage.homepage.svc.cluster.local` | 3000 |
| `blog.balkandevops.com` | `ghost.ghost.svc.cluster.local` | 2368 |
| `bookstack.balkandevops.com` | `bookstack.default.svc.cluster.local` | 80 |
| `me.balkandevops.com` | `cv.cv.svc.cluster.local` | 80 |
| `grafana.balkandevops.com` | `grafana.default.svc.cluster.local` | 80 |

**Flow:**
```
Internet → Cloudflare Edge → Tunnel Connector Pod → K8s Service → Pod
```

### DNS Configuration

- **Public DNS:** Cloudflare (CNAME to tunnel)
- **Local DNS:** AdGuard Home (`192.168.1.104:53`)
- **Router:** Points to AdGuard Home for network-wide ad blocking

**Note:** Tailscale VPN can override DNS. Configure Tailscale DNS settings or disconnect when accessing local services.

---

## 🔐 Security

### Network Security

- **External Access:** Cloudflare Tunnel (encrypted, no exposed ports)
- **Pod-to-Pod:** Calico network policies (optional)
- **DNS Filtering:** AdGuard Home blocks malicious domains

### Node Security

- **SSH:** Key-based authentication
- **Remote Access:** Tailscale VPN (encrypted mesh network)
- **Firewall:** iptables managed by Calico + custom rules

### Application Security

- **Ghost:** Runs as non-root (UID 1000)
- **Resource Limits:** Defined for all deployments

---

## 📊 Monitoring & Observability

**Status:** ✅ Deployed

**Stack:**
- **Metrics:** Prometheus (18 targets up)
- **Dashboards:** Grafana (`grafana.balkandevops.com`)
- **Power Monitoring:** Scaphandre (RAPL-based CPU power metrics)
- **DNS Stats:** AdGuard Home (queries, blocked, latency)

**Key Metrics:**
- Cluster power consumption: ~9W RAPL / ~11W wall estimate
- Monthly estimated cost: ~60 MKD

---

## 🔄 Backup & Disaster Recovery

**Current Strategy:**
- **Configuration:** Git repository (this repo)
- **Application Data:** Longhorn PVCs (replicated across nodes)

**Planned:**
- Longhorn backup to external storage (S3-compatible)
- Automated backup schedules

---

## 📝 Known Issues & Workarounds

### WiFi Node Networking

The `precision5540` WiFi node requires:
1. VXLAN encapsulation (not IPIP)
2. BGP disabled
3. iptables MASQUERADE rule

See: [WiFi Node Debugging Story](../05-troubleshooting/WiFi-Node-Kubernetes-Debugging-StoryBGPDisabled.md)

### Tailscale DNS Interference

Tailscale can override local DNS, causing resolution issues. Solutions:
- Configure Tailscale to use local DNS (`192.168.1.104`)
- Or disconnect Tailscale when accessing local services

### Resource Constraints

- ASUS VivoBook has limited resources (8GB RAM shared between VM and host)
- Heavy workloads should be scheduled on `precision5540`
- Use resource limits to prevent OOM issues

---

## 🔍 Quick Reference Commands

```bash
# Cluster status
kubectl get nodes -o wide
kubectl get pods -A

# Check specific applications
kubectl get pods -n ghost
kubectl get pods -n bookstack
kubectl get pods -n adguard

# Storage
kubectl get pvc -A
kubectl get pv

# Network
kubectl get svc -A
kubectl describe ingress -A

# Logs
kubectl logs -n ghost deployment/ghost
kubectl logs -n cloudflare deployment/cloudflare-tunnel
```

---

## 📚 Related Documentation

- [Ghost Setup Guide](../Ghost-Kubernetes-Setup.md)
- [AdGuard Home Setup](../AdGuard-Home-Kubernetes-Setup.md)
- [Cloudflare Tunnel Troubleshooting](../Cloudflare-Tunnel-Troubleshooting.md)
- [WiFi Node Debugging Story](../05-troubleshooting/WiFi-Node-Kubernetes-Debugging-StoryBGPDisabled.md)
- [Helm Chart Development Guide](../Helm-Chart-Development-Guide.md)

---

## 🔄 Update History

| Date | Changes |
|------|---------|
| 2025-12-01 | Cleaned up architecture, added BookStack/MariaDB, fixed node topology |
| 2025-11-30 | Initial architecture document |

---

**Last Updated:** 2025-12-01  
**Domain:** balkandevops.com  
**Repository:** github.com/icko0/k8s-home-cluster
