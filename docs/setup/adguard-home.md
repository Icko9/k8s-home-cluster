# AdGuard Home on Kubernetes - Complete Setup Guide

## 📋 Overview

This guide documents the complete setup of AdGuard Home on a Kubernetes cluster to provide network-wide DNS-based ad blocking and tracker protection.

**Final Configuration:**
- **AdGuard Home IP:** `10.0.1.20` (Worker node IP)
- **Web UI:** `http://10.0.1.10:30646`
- **DNS Port:** 53 (UDP/TCP)
- **Router:** Home Router
- **Network:** 10.0.1.0/24

---

## 🎯 Prerequisites

- Kubernetes cluster (v1.34.2)
- Helm installed
- Worker node with static IP
- Router with DHCP server access
- Root/administrator access to worker node

---

## 📦 Step 1: Create Namespace and Install AdGuard Home via Helm

### Create Namespace

First, create a dedicated namespace for AdGuard Home:

```bash
kubectl create namespace adguard
```

### Install the Chart

```bash
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo update
helm install adguard-home k8s-at-home/adguard-home -n adguard
```

### Verify Installation

```bash
# Check pod status
kubectl get pods -n adguard -l app.kubernetes.io/name=adguard-home

# Check service
kubectl get svc adguard-home -n adguard
```

---

## 🌐 Step 1.5: Access AdGuard Home Web UI (NodePort)

By default, the Helm chart creates a **ClusterIP** service, which is only accessible from within the cluster. To access the AdGuard Home web UI from your network, change the service type to **NodePort**.

### Change Service to NodePort

```bash
kubectl patch svc adguard-home -n adguard -p '{"spec": {"type": "NodePort"}}'
```

### Find the Assigned NodePort

```bash
kubectl get svc adguard-home -n adguard -o wide
```

Output example:
```
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
adguard-home   NodePort   10.96.xxx.xxx   <none>        8080:30646/TCP   10m   app.kubernetes.io/name=adguard-home
```

The NodePort is `30646` in this example (the number after the colon in `8080:30646`).

### Access the Web UI

Open your browser and go to:
```
http://<master-node-ip>:<nodeport>
```

Example: `http://10.0.1.10:30646`

You can use any node's IP in your cluster (master or worker) with the NodePort to access the web UI.

### Why NodePort Instead of ClusterIP?

| Service Type | Accessibility | Use Case |
|--------------|---------------|----------|
| **ClusterIP** | Only within cluster | Internal services |
| **NodePort** | From any node IP + port | External access without LoadBalancer |
| **LoadBalancer** | External IP | Cloud environments with LB support |

For home labs without a cloud load balancer, **NodePort** is the simplest way to expose services externally.

---

## 🔧 Step 2: Configure DNS Port Binding

### Problem
Kubernetes NodePort services cannot bind to port 53 (privileged port). The NodePort range is 30000-32767, but DNS clients expect port 53. We need to use `hostPort` to bind directly to the host.

### Solution: Add hostPort to Deployment

```bash
kubectl patch deployment adguard-home -n adguard --type='json' -p='[
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/0/hostPort", "value": 53},
  {"op": "replace", "path": "/spec/template/spec/containers/0/ports/1/hostPort", "value": 53}
]'
```

### Verify Port Configuration

```bash
kubectl get deployment adguard-home -n adguard -o yaml | grep -A 10 "ports:"
```

Should show:
```yaml
ports:
- containerPort: 53
  hostPort: 53
  name: dns-tcp
  protocol: TCP
- containerPort: 53
  hostPort: 53
  name: dns-udp
  protocol: UDP
```

---

## 🛑 Step 3: Disable systemd-resolved on Worker Node

### Problem
`systemd-resolved` uses port 53 and conflicts with AdGuard Home. We need to disable it **only on the worker node** where AdGuard Home is running.

### Why This is Safe

**Common Concerns Addressed:**

1. **"Won't /etc/resolv.conf break?"**
   - No! `/etc/resolv.conf` can work directly without systemd-resolved
   - Applications read `/etc/resolv.conf` directly - systemd-resolved is just a wrapper
   - You can manually configure `/etc/resolv.conf` to point to your DNS servers
   - After disabling systemd-resolved, `/etc/resolv.conf` will point directly to your DNS (e.g., `nameserver 1.1.1.1`)

2. **"Will CoreDNS still work?"**
   - **Yes!** CoreDNS runs **inside Kubernetes pods**, not on the host
   - CoreDNS doesn't depend on systemd-resolved at all
   - CoreDNS has its own network namespace and DNS resolution
   - Restarting CoreDNS pods will work fine - they're completely independent
   - Kubernetes cluster DNS (CoreDNS) is unaffected by host-level systemd-resolved

3. **"What about other nodes?"**
   - Only disable systemd-resolved on the **worker node** running AdGuard Home
   - Master node and other worker nodes can keep systemd-resolved running
   - This is a node-specific change, not cluster-wide

4. **"How does DNS work without systemd-resolved?"**
   - Applications query DNS servers listed in `/etc/resolv.conf` directly
   - Without systemd-resolved, there's no local DNS cache, but that's fine
   - AdGuard Home will handle DNS caching instead
   - The host can still resolve DNS - it just queries upstream servers directly

### Solution

**SSH to worker node:**
```bash
ssh <worker-node>  # or your worker node hostname
```

**Stop and disable systemd-resolved:**
```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

**Optional: Update /etc/resolv.conf manually (if needed):**
```bash
# Backup original
sudo cp /etc/resolv.conf /etc/resolv.conf.backup

# Create new resolv.conf pointing to upstream DNS
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
```

**Note:** Most systems will automatically update `/etc/resolv.conf` after disabling systemd-resolved, but you can set it manually if needed.

### Verify Port 53 is Free

```bash
sudo ss -tulpn | grep ':53 '
```

Should NOT show `systemd-resolved` listening on port 53.

---

## 🔄 Step 4: Restart AdGuard Home Pod

```bash
kubectl rollout restart deployment adguard-home -n adguard
```

### Verify Pod is Running

```bash
kubectl get pods -n adguard -l app.kubernetes.io/name=adguard-home -o wide
```

### Check Logs

```bash
kubectl logs -n adguard <pod-name> | grep -i "53\|port\|listen"
```

Should show:
```
Listening to udp://[::]:53
Listening to tcp://[::]:53
```

---

## 🌐 Step 5: Configure Router DHCP DNS

### Find Worker Node IP

```bash
kubectl get nodes -o wide
```

Note the **INTERNAL-IP** of your worker node (e.g., `10.0.1.20`).

### Configure Router DHCP Server

1. **Access Router:** Open router web interface
2. **Navigate to:** `Network → DHCP Server` (NOT Internet/WAN DNS)
3. **Configure:**
   - **Primary DNS:** `10.0.1.20` (your worker node IP)
   - **Secondary DNS:** `1.1.1.1` (Cloudflare backup)
4. **Save** settings

### Important Notes

- ⚠️ **DO NOT** configure DNS in "Internet" or "WAN DNS" section
- ✅ Configure DNS in **"DHCP Server"** section
- This ensures all devices get the DNS automatically via DHCP

---

## ✅ Step 6: Verify DNS is Working

### Test from Worker Node

```bash
nslookup google.com 10.0.1.20
```

Should resolve successfully.

### Check AdGuard Home Dashboard

1. Open AdGuard Home web UI: `http://10.0.1.10:30646`
2. Go to **Dashboard**
3. You should see DNS queries appearing
4. Check **Top clients** - should show multiple device IPs

### Test from Devices

**On each device:**
1. **Disconnect and reconnect Wi‑Fi** (forces DHCP renewal)
2. Or manually renew DHCP:
   - **Windows:** `ipconfig /release && ipconfig /renew`
   - **Linux:** `sudo dhclient -r && sudo dhclient`
   - **Android/iOS:** Toggle Wi‑Fi off/on

**Verify DNS on device:**
- Check network settings - DNS should show `10.0.1.20`
- Visit websites - should work normally
- Check AdGuard Home dashboard - should see queries from device

---

## 🛡️ Step 7: Enable Ad Blocking Filters

### Configure Filter Lists

1. **Open AdGuard Home:** `http://10.0.1.10:30646`
2. **Navigate to:** `Filters → DNS blocklists`
3. **Enable recommended lists:**
   - ✅ AdGuard DNS filter
   - ✅ AdAway
   - ✅ EasyList
   - ✅ EasyPrivacy
   - ✅ Malware Domains
   - ✅ Phishing Army
4. **Click "Update"** to download filters

### Configure Upstream DNS Servers

1. **Navigate to:** `Settings → DNS settings`
2. **Upstream DNS servers:** Add:
   ```
   1.1.1.1
   1.0.0.1
   ```
   (Cloudflare DNS)
3. **Click "Apply"**

---

## 🧪 Step 8: Test Ad Blocking

### Test Sites

1. **News/Blog Sites:**
   - Visit CNN, BBC, tech blogs, etc.
   - Banner ads and popups should be blocked
   - Check AdGuard Home Query Log for blocked domains

2. **News Sites:**
   - Visit CNN, BBC, etc.
   - Banner ads should be gone
   - Page should load faster

3. **AdGuard Home Dashboard:**
   - Check **"Blocked by Filters"** - should show blocked count
   - Check **"Top Blocked Domains"** - should list blocked domains
   - Check **Query Log** - blocked queries appear in red

### What Gets Blocked

✅ **Works well:**
- Tracking scripts
- Analytics trackers
- Most banner ads
- Malware/phishing sites
- Social media trackers

⚠️ **Cannot block (same domain as content):**
- YouTube ads (served from youtube.com/googlevideo.com)
- Twitch ads
- Some in-app ads
- First-party ads embedded in content

---

## 🔍 Troubleshooting

### Issue: No DNS Queries in Dashboard

**Symptoms:** Dashboard shows 0 queries

**Solutions:**
1. Verify router DHCP DNS is set to worker node IP
2. Reconnect devices to Wi‑Fi
3. Check device DNS settings manually
4. Test DNS: `nslookup google.com 10.0.1.20`

### Issue: Port 53 Already in Use

**Symptoms:** Pod won't start or can't bind to port 53

**Solutions:**
1. Check for `systemd-resolved`: `sudo systemctl stop systemd-resolved`
2. Check for `dnsmasq`: `sudo systemctl stop dnsmasq` (if not needed)
3. Verify port is free: `sudo ss -tulpn | grep ':53 '`

### Issue: Router Rejects DNS IP

**Symptoms:** Router shows "Invalid Primary DNS Server"

**Solutions:**
1. Make sure you're configuring **DHCP Server DNS**, not WAN DNS
2. Verify worker node IP is on same subnet as router
3. Check router allows local IPs for DNS

### Issue: Devices Not Using AdGuard DNS

**Symptoms:** Devices still using old DNS

**Solutions:**
1. Force DHCP renewal on devices
2. Manually set DNS on device to `10.0.1.20`
3. Check router DHCP settings are saved
4. Restart router if needed

---

## 📊 Monitoring & Maintenance

### Check Status

```bash
# Pod status
kubectl get pods -n adguard -l app.kubernetes.io/name=adguard-home

# Service status
kubectl get svc adguard-home -n adguard

# Pod logs
kubectl logs -n adguard -l app.kubernetes.io/name=adguard-home --tail=50
```

### Update Filters

1. Go to **Filters → DNS blocklists**
2. Click **"Update"** to refresh filter lists
3. Recommended: Update weekly

### View Statistics

- **Dashboard:** Overall statistics
- **Query Log:** Real-time DNS queries
- **Top Clients:** Which devices are using DNS
- **Top Blocked:** Most blocked domains

---

## 🔐 Security Considerations

### Network Isolation

- AdGuard Home runs in default namespace
- Consider moving to dedicated namespace for better isolation
- Use NetworkPolicies if needed

### Access Control

- AdGuard Home web UI is exposed via NodePort
- Consider using Ingress with authentication
- Or restrict NodePort access via firewall

### Updates

```bash
# Update Helm chart
helm repo update
helm upgrade adguard-home k8s-at-home/adguard-home -n adguard

# Update AdGuard Home itself
# Check for updates in web UI: Settings → About
```

---

## 📝 Configuration Summary

### Kubernetes Resources

- **Namespace:** `adguard`
- **Deployment:** `adguard-home`
- **Service:** `adguard-home` (NodePort)
- **Ports:**
  - HTTP: `30646` (NodePort)
  - DNS TCP: `53` (hostPort)
  - DNS UDP: `53` (hostPort)

### Network Configuration

- **Master Node IP:** `10.0.1.10`
- **Worker Node IP:** `10.0.1.20`
- **Router IP:** `10.0.1.1`
- **DHCP Range:** `10.0.1.100-249`
- **Router DNS (DHCP):** `10.0.1.20`

### AdGuard Home Settings

- **Upstream DNS:** `1.1.1.1`, `1.0.0.1` (Cloudflare)
- **Filter Lists:** AdGuard DNS, AdAway, EasyList, EasyPrivacy, Malware Domains, Phishing Army
- **Web UI:** `http://10.0.1.10:30646`

---

## 🎉 Success Indicators

✅ AdGuard Home pod is running  
✅ Port 53 is bound on worker node  
✅ Router DHCP assigns DNS to `10.0.1.20`  
✅ Devices show DNS queries in dashboard  
✅ Filter lists are enabled and updated  
✅ Blocked queries appear in dashboard  
✅ Ads are reduced/blocked on devices  

---

## 📚 Additional Resources

- [AdGuard Home Documentation](https://github.com/AdguardTeam/AdGuardHome)
- [k8s-at-home Helm Charts](https://github.com/k8s-at-home/charts)
- [Reference Article](https://pablomurga.com/posts/adguard-home/)

---

## 🔄 Rollback Procedure

If you need to revert changes:

```bash
# Remove hostPort binding
kubectl patch deployment adguard-home -n adguard --type='json' -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/ports/0/hostPort"},
  {"op": "remove", "path": "/spec/template/spec/containers/0/ports/1/hostPort"}
]'

# Re-enable systemd-resolved (if needed)
sudo systemctl enable systemd-resolved
sudo systemctl start systemd-resolved

# Reset router DHCP DNS to default
# (Set Primary DNS back to 0.0.0.0 or router IP)

# Uninstall AdGuard Home
helm uninstall adguard-home -n adguard

# Optional: Delete namespace
kubectl delete namespace adguard
```

---

**Last Updated:** 2025-11-28  
**Kubernetes Version:** v1.34.2  
**AdGuard Home Version:** v0.107.7  
**Chart Version:** adguard-home-5.5.2
