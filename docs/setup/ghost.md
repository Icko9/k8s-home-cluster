# Ghost on Kubernetes - Complete Setup Guide

## 📋 Overview

This guide documents the complete setup of Ghost (a headless CMS/blogging platform) on a Kubernetes cluster using a custom Helm chart with the official Ghost Docker image from [Docker Hub](https://hub.docker.com/_/ghost/).

**Final Configuration:**
- **Ghost URL:** `http://10.0.1.10:30080`
- **Storage:** Longhorn (10Gi persistent volume)
- **Database:** SQLite (built-in, default)
- **Service Type:** NodePort
- **Image:** Official Ghost Docker image

---

## 🎯 Prerequisites

- Kubernetes cluster (v1.34.2)
- Helm 3.x installed
- Longhorn storage provisioner configured
- Access to cluster nodes

---

## 📦 Step 1: Review the Helm Chart

The custom Helm chart is located in `apps/ghost/` and includes:

- **Chart.yaml** - Chart metadata
- **values.yaml** - Configuration values
- **templates/** - Kubernetes manifest templates
  - `deployment.yaml` - Ghost deployment
  - `service.yaml` - NodePort service
  - `pvc.yaml` - Persistent volume claim
  - `_helpers.tpl` - Template helpers

### Chart Structure

```
apps/ghost/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    └── pvc.yaml
```

---

## 📝 Step 2: Customize Configuration

### Review Default Values

The `apps/ghost/values.yaml` file contains all configuration options:

```bash
cat apps/ghost/values.yaml
```

### Key Configuration Options

1. **Ghost URL** (`ghost.url`):
   - Set to your master node IP and NodePort
   - Example: `http://10.0.1.10:30080`
   - **Important:** This must match your actual access URL

2. **Image Version** (`image.tag`):
   - Default: `5.85.0` (specific version)
   - Can use `latest` for latest version
   - Recommended: Use specific version tags for production

3. **NodePort** (`service.nodePort`):
   - Default: `30080`
   - Set to `null` for auto-assignment
   - Must be in range 30000-32767

4. **Storage Size** (`persistence.size`):
   - Default: `10Gi`
   - Adjust based on your content needs

5. **Database** (`ghost.database.client`):
   - Default: `sqlite3` (built-in, no setup needed)
   - Can configure MySQL/MariaDB for production

6. **Email Configuration** (`ghost.mail`):
   - Optional but recommended
   - Needed for password resets, notifications

### Edit Configuration

```bash
# Edit the values file
nano apps/ghost/values.yaml
```

**Important Settings to Review:**
- `ghost.url`: Must match your actual access URL (IP + NodePort)
- `service.nodePort`: Port for external access
- `persistence.size`: Storage size
- `image.tag`: Ghost version

---

## 🚀 Step 3: Create Namespace

Create a dedicated namespace for Ghost:

```bash
kubectl create namespace ghost
```

---

## 🚀 Step 4: Install Ghost

### Install from Local Chart

```bash
helm install ghost ./apps/ghost \
  --namespace ghost
```

### Install with Custom Values (Alternative)

If you want to override specific values without editing the file:

```bash
helm install ghost ./apps/ghost \
  --namespace ghost \
  --set ghost.url=http://10.0.1.10:30080 \
  --set service.nodePort=30080 \
  --set persistence.storageClass=longhorn
```

### Install with Custom Values File

```bash
# Create a custom values file
cp apps/ghost/values.yaml apps/ghost/values-custom.yaml
# Edit values-custom.yaml
nano apps/ghost/values-custom.yaml

# Install with custom values
helm install ghost ./apps/ghost \
  --namespace ghost \
  -f apps/ghost/values-custom.yaml
```

---

## ✅ Step 5: Verify Installation

### Check Pod Status

```bash
kubectl get pods -n ghost
```

Wait for the pod to be in `Running` state. Initial startup may take 1-2 minutes.

**Expected output:**
```
NAME                     READY   STATUS    RESTARTS   AGE
ghost-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### Check Service

```bash
kubectl get svc -n ghost
```

**Expected output:**
```
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
ghost   NodePort   10.96.xxx.xxx   <none>        2368:30080/TCP   2m
```

The NodePort is `30080` (or the port you configured).

### Check Persistent Volume

```bash
kubectl get pvc -n ghost
```

Should show a PVC bound to a Longhorn volume:
```
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ghost   Bound    pvc-xxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx      10Gi       RWO            longhorn       2m
```

### Check Pod Logs

```bash
kubectl logs -n ghost -l app.kubernetes.io/name=ghost --tail=50
```

Look for:
- "Ghost is running in production"
- "Listening on: 0.0.0.0:2368"
- No error messages

### Describe Pod (if issues)

```bash
kubectl describe pod -n ghost -l app.kubernetes.io/name=ghost
```

---

## 🌐 Step 6: Access Ghost Web Interface

### Find the NodePort

```bash
kubectl get svc ghost -n ghost -o jsonpath='{.spec.ports[0].nodePort}'
```

Or check the service directly:
```bash
kubectl get svc ghost -n ghost
```

### Access the Web UI

Open your browser and go to:
```
http://<master-node-ip>:<nodeport>
```

Example: `http://10.0.1.10:30080`

You can use any node's IP in your cluster (master or worker) with the NodePort to access Ghost.

### Initial Setup

1. **First Visit:** You'll see the Ghost setup page
2. **Create Admin Account:**
   - Site Title: Your blog name
   - Full Name: Your name
   - Email: Your email address
   - Password: Set a strong password
3. **Complete Setup:** Click "Create account & start publishing"

---

## 🔐 Step 7: Configure Email (Recommended)

Ghost needs email configuration for:
- Password resets
- New user invitations
- Comment notifications
- Newsletter emails

### Edit Values and Upgrade

1. **Edit `apps/ghost/values.yaml`:**

```yaml
ghost:
  mail:
    transport: "SMTP"
    options:
      host: "smtp.gmail.com"  # Your SMTP server
      port: 587
      secure: false
      auth:
        user: "your-email@gmail.com"
        pass: "your-app-password"  # Use app-specific password for Gmail
      from: "noreply@yourdomain.com"
```

2. **Upgrade the deployment:**

```bash
helm upgrade ghost ./apps/ghost \
  --namespace ghost
```

3. **Restart the pod (if needed):**

```bash
kubectl rollout restart deployment ghost -n ghost
```

### Gmail Setup

For Gmail, you need to:
1. Enable 2-factor authentication
2. Generate an app-specific password
3. Use the app-specific password in the config

---

## 📊 Step 8: Configure External Database (Optional)

By default, Ghost uses SQLite stored in `/var/lib/ghost/content`. For better performance and scalability, consider using MySQL/MariaDB.

### Option 1: Use External MySQL/MariaDB

1. **Edit `apps/ghost/values.yaml`:**

```yaml
ghost:
  database:
    client: "mysql"
    connection:
      host: "mysql-host"  # Your MySQL host
      port: 3306
      user: "ghost"
      password: "your-password"
      database: "ghost"
```

2. **Create the database:**

```sql
CREATE DATABASE ghost CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'ghost'@'%' IDENTIFIED BY 'your-password';
GRANT ALL PRIVILEGES ON ghost.* TO 'ghost'@'%';
FLUSH PRIVILEGES;
```

3. **Upgrade:**

```bash
helm upgrade ghost ./apps/ghost \
  --namespace ghost
```

4. **Restart the pod:**

```bash
kubectl rollout restart deployment ghost -n ghost
```

**Note:** When switching from SQLite to MySQL, you'll need to migrate your data or start fresh.

---

## 🔄 Step 9: Backup and Restore

### Backup Ghost Data

Ghost data is stored in the persistent volume at `/var/lib/ghost/content`. To backup:

```bash
# Create a backup pod
kubectl run ghost-backup --rm -i --tty --image=ghost:5.85.0 \
  --namespace ghost \
  --restart=Never \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "ghost-backup",
      "image": "ghost:5.85.0",
      "command": ["/bin/bash"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "mountPath": "/var/lib/ghost/content",
        "name": "ghost-content"
      }]
    }],
    "volumes": [{
      "name": "ghost-content",
      "persistentVolumeClaim": {
        "claimName": "ghost"
      }
    }]
  }
}'

# Inside the pod, create a tar archive
tar -czf /tmp/ghost-backup-$(date +%Y%m%d).tar.gz /var/lib/ghost/content

# Copy backup out (from another terminal)
kubectl cp ghost/ghost-backup:/tmp/ghost-backup-YYYYMMDD.tar.gz ./ghost-backup.tar.gz
```

### Restore Ghost Data

```bash
# Copy backup into pod
kubectl cp ./ghost-backup.tar.gz ghost/ghost-backup:/tmp/

# Inside the pod, extract
tar -xzf /tmp/ghost-backup.tar.gz -C /
```

### Automated Backup Script

You can create a cron job or scheduled task to automate backups:

```bash
#!/bin/bash
# backup-ghost.sh
DATE=$(date +%Y%m%d-%H%M%S)
kubectl exec -n ghost deployment/ghost -- tar -czf /tmp/ghost-backup-$DATE.tar.gz /var/lib/ghost/content
kubectl cp ghost/ghost-xxxxx-xxxxx:/tmp/ghost-backup-$DATE.tar.gz ./backups/ghost-backup-$DATE.tar.gz
kubectl exec -n ghost deployment/ghost -- rm /tmp/ghost-backup-$DATE.tar.gz
```

---

## 🔧 Troubleshooting

### Issue: Pod Not Starting

**Symptoms:** Pod stuck in `Pending` or `CrashLoopBackOff`

**Solutions:**
1. Check pod logs: `kubectl logs -n ghost <pod-name>`
2. Check pod events: `kubectl describe pod -n ghost <pod-name>`
3. Check PVC status: `kubectl get pvc -n ghost`
4. Verify Longhorn is working: `kubectl get storageclass`
5. Check resource limits: `kubectl describe pod -n ghost <pod-name>`

### Issue: Cannot Access Web UI

**Symptoms:** Connection refused or timeout

**Solutions:**
1. Verify NodePort: `kubectl get svc ghost -n ghost`
2. Check firewall rules on nodes
3. Verify pod is running: `kubectl get pods -n ghost`
4. Check service endpoints: `kubectl get endpoints ghost -n ghost`
5. Verify `ghost.url` in values.yaml matches your access URL

### Issue: Ghost URL Mismatch

**Symptoms:** Ghost shows "URL mismatch" error or redirects incorrectly

**Solutions:**
1. Update `ghost.url` in values.yaml to match your actual access URL
2. Upgrade the deployment: `helm upgrade ghost ./apps/ghost --namespace ghost`
3. Restart the pod: `kubectl rollout restart deployment ghost -n ghost`

### Issue: Database Connection Errors

**Symptoms:** Database connection errors in logs (if using MySQL)

**Solutions:**
1. Verify database credentials in values.yaml
2. Check database is accessible from cluster
3. Verify database exists and user has permissions
4. Check network policies if using external database

### Issue: Storage Issues

**Symptoms:** PVC not binding or storage errors

**Solutions:**
1. Verify Longhorn is running: `kubectl get pods -n longhorn-system`
2. Check storage class: `kubectl get storageclass longhorn`
3. Check PVC events: `kubectl describe pvc ghost -n ghost`
4. Verify storage size is available

### Issue: Permission Errors

**Symptoms:** Permission denied errors in logs

**Solutions:**
1. Verify security context in values.yaml
2. Check PVC permissions
3. Ensure fsGroup matches Ghost's UID (1000)

---

## 📊 Monitoring & Maintenance

### Check Status

```bash
# Pod status
kubectl get pods -n ghost

# Service status
kubectl get svc ghost -n ghost

# PVC status
kubectl get pvc -n ghost

# Deployment status
kubectl get deployment ghost -n ghost

# Pod logs
kubectl logs -n ghost -l app.kubernetes.io/name=ghost --tail=50
```

### Update Ghost

1. **Update image tag in values.yaml:**

```yaml
image:
  tag: "5.85.0"  # Update to new version
```

2. **Upgrade the deployment:**

```bash
helm upgrade ghost ./apps/ghost \
  --namespace ghost
```

3. **Verify the update:**

```bash
kubectl rollout status deployment ghost -n ghost
```

### Scale Ghost (if needed)

**Note:** Ghost with SQLite cannot be scaled (single instance only). For scaling, you need:
- External database (MySQL/MariaDB)
- Shared storage (ReadWriteMany)

To scale:

```bash
helm upgrade ghost ./apps/ghost \
  --namespace ghost \
  --set replicaCount=2 \
  --set ghost.database.client=mysql \
  --set persistence.accessModes[0]=ReadWriteMany
```

---

## 🔐 Security Considerations

### Network Access

- Ghost is exposed via NodePort
- Consider using Ingress with TLS/SSL for production
- Restrict NodePort access via firewall if needed
- Use reverse proxy (nginx, traefik) for additional security

### Admin Access

- Use strong admin password
- Enable 2FA in Ghost admin panel (Settings → Security)
- Regularly update Ghost to latest version
- Limit admin access to trusted IPs if possible

### Storage

- Regular backups of persistent volume
- Consider encrypting PVC if sensitive content
- Monitor storage usage

### Updates

```bash
# Check for Ghost updates
# Visit: https://hub.docker.com/r/library/ghost/tags

# Update chart version in Chart.yaml
# Update image tag in values.yaml
helm upgrade ghost ./apps/ghost --namespace ghost
```

---

## 📝 Configuration Summary

### Kubernetes Resources

- **Namespace:** `ghost`
- **Deployment:** `ghost`
- **Service:** `ghost` (NodePort)
- **PVC:** `ghost` (Longhorn, 10Gi)
- **Port:** `2368:<NodePort>` (e.g., `2368:30080`)

### Network Configuration

- **Master Node IP:** `10.0.1.10`
- **Ghost URL:** `http://10.0.1.10:30080`
- **Service Type:** NodePort
- **Container Port:** 2368

### Ghost Settings

- **Database:** SQLite (default) or MySQL (optional)
- **Storage:** Longhorn (10Gi) at `/var/lib/ghost/content`
- **Image:** Official Ghost Docker image
- **Admin User:** Configured during first setup

---

## 🎉 Success Indicators

✅ Ghost pod is running  
✅ PVC is bound to Longhorn volume  
✅ Service is accessible via NodePort  
✅ Web UI loads and shows setup page  
✅ Admin account created successfully  
✅ Can create and publish posts  
✅ Email notifications working (if configured)  

---

## 📚 Additional Resources

- [Ghost Documentation](https://ghost.org/docs/)
- [Ghost Docker Image](https://hub.docker.com/_/ghost/)
- [Ghost Configuration](https://ghost.org/docs/config/)
- [Ghost Themes](https://ghost.org/themes/)
- [Ghost API Documentation](https://ghost.org/docs/api/)
- [Helm Documentation](https://helm.sh/docs/)

---

## 🔄 Uninstall Procedure

If you need to remove Ghost:

```bash
# Uninstall Helm release
helm uninstall ghost -n ghost

# Delete PVC (optional - this will delete all data!)
kubectl delete pvc ghost -n ghost

# Delete namespace (optional)
kubectl delete namespace ghost
```

**Warning:** Deleting the PVC will permanently delete all Ghost data. Make sure you have backups!

---

## 🔄 Upgrade Procedure

To upgrade Ghost to a new version:

1. **Backup your data** (see Backup section)

2. **Update values.yaml:**

```yaml
image:
  tag: "5.85.0"  # New version
```

3. **Upgrade:**

```bash
helm upgrade ghost ./apps/ghost \
  --namespace ghost
```

4. **Verify:**

```bash
kubectl rollout status deployment ghost -n ghost
kubectl logs -n ghost -l app.kubernetes.io/name=ghost --tail=50
```

---

**Last Updated:** 2025-01-27  
**Kubernetes Version:** v1.34.2  
**Ghost Image:** Official Docker Hub image  
**Storage:** Longhorn  
**Chart Version:** 0.1.0
