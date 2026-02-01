# Ghost Helm Chart

A custom Helm chart for deploying Ghost CMS using the official [Ghost Docker image](https://hub.docker.com/_/ghost/).

## Features

- Uses official Ghost Docker image
- Persistent storage with Longhorn
- NodePort service for external access
- Configurable database (SQLite default, MySQL optional)
- Email configuration support
- Resource limits and health checks
- Security contexts

## Quick Start

```bash
# Create namespace
kubectl create namespace ghost

# Install
helm install ghost ./apps/ghost --namespace ghost

# Check status
kubectl get pods -n ghost
```

## Configuration

Edit `values.yaml` to customize:

- **Ghost URL**: `ghost.url` - Must match your access URL
- **NodePort**: `service.nodePort` - Port for external access
- **Storage**: `persistence.size` - Storage size
- **Image Version**: `image.tag` - Ghost version
- **Database**: `ghost.database.client` - sqlite3 or mysql
- **Email**: `ghost.mail` - SMTP configuration

## Access

After installation, access Ghost at:
```
http://<node-ip>:<nodeport>
```

Find the NodePort:
```bash
kubectl get svc ghost -n ghost
```

## Documentation

See [Ghost Kubernetes Setup Guide](../../docs/Ghost-Kubernetes-Setup.md) for complete documentation.

## Chart Structure

```
apps/ghost/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default values
├── README.md              # This file
└── templates/
    ├── _helpers.tpl       # Template helpers
    ├── deployment.yaml    # Ghost deployment
    ├── service.yaml       # NodePort service
    └── pvc.yaml          # Persistent volume claim
```

