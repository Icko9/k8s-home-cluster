# k8s-home-cluster

A GitOps-managed Kubernetes homelab running on repurposed laptops.

## 🏗️ [Architecture](docs/troubleshooting/architecture.md).

### Physical Layout
- **1x ASUS VivoBook** - Hosts master VM + runs as worker node
- **1x Dell Precision 5540** - WiFi worker (requires MASQUERADE setup)

### Logical Kubernetes Nodes

| Node | Hostname | Type | Role | Connection | IP |
|------|----------|------|------|------------|-----|
| Master | `master` | VM | Control Plane | Bridged networking | 192.168.1.50 |
| Worker 1 | `asuslpt` | Physical | Worker + VM Host | USB-to-LAN | 192.168.1.104 |
| Worker 2 | `precision5540` | Physical | Worker | WiFi ⚠️ | 192.168.1.105 |

> **Note:** The `precision5540` WiFi node requires special iptables MASQUERADE configuration. See [WiFi Node Setup](docs/troubleshooting/wifi-node-debugging.md).

## 📦 Deployed Applications

| App | URL | Description |
|-----|-----|-------------|
| Homepage | [balkandevops.com](https://www.balkandevops.com) | Dashboard with cluster monitoring |
| Ghost | [blog.balkandevops.com](https://blog.balkandevops.com) | Tech blog |
| BookStack | [bookstack.balkandevops.com](https://bookstack.balkandevops.com) | Documentation wiki |
| CV | [me.balkandevops.com](https://me.balkandevops.com) | Personal portfolio |
| Grafana | [grafana.balkandevops.com](https://grafana.balkandevops.com) | Metrics & dashboards |
| AdGuard Home | Internal | DNS & ad blocking |

## 🗂️ Repository Structure

```
k8s-home-cluster/
├── apps/                        # Application manifests
│   ├── cv/                      # Personal CV page
│   ├── ghost/                   # Ghost blog (Helm chart)
│   ├── grafana/                 # Monitoring dashboards
|   ├── nextcloud/               # Family Cloud 
│   └── homepage/                # Homepage dashboard
│
├── infrastructure/              # Cluster infrastructure
│   └── cloudflare-tunnel/       # External access via Cloudflare
│
├── docs/                        # Documentation
│   ├── architecture.md          # Cluster architecture details
│   ├── getting-started.md       # Setup guide
│   ├── setup/                   # Application setup guides
│   └── troubleshooting/         # Debug guides & postmortems
│
└── scripts/                     # Helper scripts
    └── <Empty>
```

> **⚠️ Important:** Replace all IP addresses (192.168.1.x) in configs with your actual network addresses when replicating this setup.

## 🚀 Tech Stack

- **Kubernetes**: kubeadm-bootstrapped cluster (v1.28+)
- **CNI**: Calico (BGP disabled for WiFi compatibility)
- **Storage**: Longhorn distributed storage
- **Ingress**: Cloudflare Tunnel (bypasses CGNAT)
- **Monitoring**: Prometheus + Grafana
- **DNS**: AdGuard Home (cluster + home network)


*Built with ☕ and late nights debugging WiFi nodes.*

