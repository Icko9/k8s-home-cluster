# k8s-home-cluster

A GitOps-managed Kubernetes homelab running on repurposed laptops.

## 🏗️ Architecture

| Node | Hostname | Hardware | Role | Connection |
|------|----------|----------|------|------------|
| Master | `master` | VM on ASUS VivoBook | Control Plane | Bridge (Ethernet) |
| Worker 1 | `asuslpt` | ASUS VivoBook i3, 8GB RAM | Worker + VM Host | USB-to-LAN |
| Worker 2 | `precision5540` | Dell Precision i7-9850H, 16GB RAM | Worker | WiFi |

## 📦 Deployed Applications

| App | URL | Description |
|-----|-----|-------------|
| Homepage | [www.balkandevops.com](https://balkandevops.com) | Dashboard with cluster monitoring |
| Ghost | [blog.balkandevops.com](https://blog.balkandevops.com) | Tech blog |
| BookStack | [bookstack.balkandevops.com](https://bookstack.balkandevops.com) | Documentation wiki |
| CV | [me.balkandevops.com](https://me.balkandevops.com) | Personal portfolio |
| Grafana | [grafana.balkandevops.com](https://grafana.balkandevops.com) | Metrics & dashboards |
| AdGuard Home | Internal | DNS & ad blocking |

## 🗂️ Repository Structure

```
k8s-home-cluster/
├── apps/                        # Application manifests
│   ├── adguard/                 # AdGuard Home DNS
│   ├── bookstack/               # Documentation wiki
│   ├── cv/                      # Personal CV page
│   ├── ghost/                   # Ghost blog (Helm chart)
│   ├── grafana/                 # Monitoring dashboards
│   └── homepage/                # Homepage dashboard
│
├── infrastructure/              # Cluster infrastructure
│   ├── cloudflare-tunnel/       # External access via Cloudflare
│   ├── ingress/                 # Ingress resources
│   └── longhorn/                # Storage provisioner
│
├── docs/                        # Documentation
│   ├── architecture.md          # Cluster architecture details
│   ├── setup-guide.md           # Initial cluster setup
│   └── troubleshooting/         # Debug guides & postmortems
│
└── scripts/                     # Helper scripts
```

## 🚀 Tech Stack

- **Kubernetes**: kubeadm-bootstrapped cluster (v1.28+)
- **CNI**: Calico (BGP disabled for WiFi compatibility)
- **Storage**: Longhorn distributed storage
- **Ingress**: Cloudflare Tunnel (bypasses CGNAT)
- **Monitoring**: Prometheus + Grafana
- **DNS**: AdGuard Home (cluster + home network)

## 🔧 Quick Start

```bash
# Clone the repository
git clone git@github.com:icko0/k8s-home-cluster.git
cd k8s-home-cluster

# Deploy an application
kubectl apply -f apps/homepage/

# Deploy infrastructure component
kubectl apply -f infrastructure/ingress/
```

## 🎯 Roadmap

- [ ] ArgoCD for GitOps automation
- [ ] Sealed Secrets for secret management
- [ ] Renovate for dependency updates
- [ ] Automated backups with Velero

## 📝 License

MIT

---

*Built with ☕ and late nights debugging WiFi nodes.*

