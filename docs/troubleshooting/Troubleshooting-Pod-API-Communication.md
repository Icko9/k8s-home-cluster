# Troubleshooting: Pods Failing to Communicate with Kubernetes API

## Overview

This guide covers how to diagnose and fix issues where pods cannot reach the Kubernetes API server or other cluster services. Based on real troubleshooting experience with a WiFi worker node.

---

## Symptoms

Common signs that pods can't reach the Kubernetes API:

- `Get "https://10.96.0.1:443/...": net/http: TLS handshake timeout`
- `connection refused` or `connection timed out` errors
- Pods stuck in `CrashLoopBackOff` or not becoming Ready
- Leader election failures (`error retrieving resource lock`)
- Service discovery failures

**Key indicator:** Host can reach the API server, but pods inside can't.

---

## Quick Diagnosis Checklist

### 1. Verify Host Connectivity (Should Work)

```bash
# On the problem node
curl -k https://10.96.0.1:443/version --max-time 5
```

If this **works** (returns JSON or 403), the node is fine. Move to step 2.

If this **fails**, you have a node-level networking problem (not covered in this guide).

### 2. Verify Pod Connectivity (Will Fail)

```bash
# Create a debug pod on the problem node
kubectl run nettest --image=nicolaka/netshoot \
  --overrides='{"spec": {"nodeName": "<problem-node>"}}' \
  --command -- sleep 3600

# Test from inside the pod
kubectl exec -it nettest -- curl -k --max-time 5 https://10.96.0.1:443/version
```

If this **fails** but host works → You have a **pod networking issue**. Continue with this guide.

### 3. Check Pod Routes

```bash
# Exec into the failing pod
kubectl exec -it <pod-name> -n <namespace> -c <container> -- sh

# Check routing table
ip route
```

**Expected:** Should show routes to service CIDR (10.96.0.0/12) and pod CIDR (192.168.0.0/16).

**Problem:** Only shows `default via 169.254.1.1 dev eth0` with no service/pod routes.

If routes are missing, this is a **CNI issue**.

---

## Common Causes & Solutions

### Issue 1: Firewall Blocking Kubelet Port

**Symptom:** API server can't fetch pod logs or exec into pods.

**Check:**
```bash
# From master
kubectl logs -n <namespace> <pod-on-problem-node>
```

If you get `connection refused` or timeout errors like:
```
Error from server: Get "https://192.168.1.105:10250/containerLogs/...": dial tcp 192.168.1.105:10250: connection refused
```

**Fix:**
```bash
# On the problem node
sudo ufw allow 10250/tcp   # Kubelet API
sudo ufw allow 10255/tcp   # Kubelet read-only (optional)

# Or allow all cluster traffic
sudo ufw allow from <cluster-cidr>

# Restart kubelet
sudo systemctl restart kubelet
```

---

### Issue 2: MTU Mismatch

**Symptom:** Some packets work, TLS handshakes fail. In tcpdump you see:
```
ICMP ... unreachable - need to frag (mtu 1450)
```

**Cause:** Packets exceed path MTU and have "Don't Fragment" flag set.

**Check:**
```bash
# On problem node
ip link show <interface>  # Check MTU
```

WiFi interfaces often have effective MTU < 1500 due to overhead.

**Fix (Tigera Operator - recommended):**
```bash
# Edit Calico Installation
kubectl edit installation default

# Add under spec:
spec:
  calicoNetwork:
    mtu: 1400  # Lower MTU for WiFi compatibility
```

**Fix (Manual - temporary):**
```bash
# On problem node
sudo ip link set <interface> mtu 1450
sudo ip link set tunl0 mtu 1400

# Restart calico-node pod
kubectl delete pod -n calico-system calico-node-<pod-on-problem-node>
```

---

### Issue 3: Missing /etc/hosts Entries

**Symptom:** Pods can't resolve node hostnames.

**Check:**
```bash
# On problem node
cat /etc/hosts | grep -E "master|worker"
```

**Fix:**
```bash
# Add all cluster nodes to /etc/hosts
sudo tee -a /etc/hosts << EOF
10.0.1.10  master
10.0.1.20  worker1
10.0.1.30  worker2
EOF

# Restart kubelet
sudo systemctl restart kubelet
```

---

### Issue 4: iptables Mode Mismatch

**Symptom:** CNI creates iptables rules but they don't apply.

**Check:**
```bash
sudo update-alternatives --query iptables
```

Kubernetes/Calico work better with `iptables-legacy` on older systems.

**Fix:**
```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

# Restart kubelet
sudo systemctl restart kubelet

# Restart CNI
kubectl delete pod -n calico-system calico-node-<pod>
kubectl delete pod -n kube-system kube-proxy-<pod>
```

---

### Issue 5: IP Forwarding Disabled

**Symptom:** Pods can't route traffic through the host.

**Check:**
```bash
sysctl net.ipv4.ip_forward
```

Should be `1`.

**Fix:**
```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# Make permanent
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
```

---

### Issue 6: systemd-resolved Conflict

**Symptom:** Port 53 conflicts, DNS resolution issues.

**Check:**
```bash
sudo ss -tulpn | grep ':53 '
```

If `systemd-resolved` is listening on port 53:

**Fix:**
```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved

# Update resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

---

### Issue 7: Missing Kernel Modules

**Symptom:** Calico IPIP tunneling doesn't work.

**Check:**
```bash
lsmod | grep ipip
```

**Fix:**
```bash
sudo modprobe ipip

# Load on boot
echo "ipip" | sudo tee -a /etc/modules
```

For VXLAN:
```bash
sudo modprobe vxlan
echo "vxlan" | sudo tee -a /etc/modules
```

---

### Issue 8: Router Not Routing Pod IPs (WiFi Specific)

**Symptom (ADVANCED):**
- Host connectivity works
- iptables rules correct
- Packets leave node successfully
- Return packets don't arrive back at pods

**Diagnosis with tcpdump:**
```bash
# On problem node
sudo tcpdump -i <interface> -n 'host <pod-ip> and (icmp or port 443)' -v
```

Look for:
- Outbound packets from pod IP
- Return packets from destination
- "need to frag" ICMP messages

If you see **outbound packets but no returns**, or **ICMP fragmentation needed**, the router can't route pod IPs back.

**Root Cause:** Home router doesn't have routes for pod CIDR (192.168.88.0/24, etc.). Packets with pod source IPs get lost.

**Fix: MASQUERADE (NAT) pod traffic**
```bash
# On WiFi node, NAT pod traffic to node IP
sudo iptables -t nat -A POSTROUTING -s <pod-cidr> -o <wifi-interface> -j MASQUERADE

# Example:
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

**Make it permanent:**
```bash
# Create systemd service
sudo tee /etc/systemd/system/k8s-masquerade.service << 'EOF'
[Unit]
Description=Masquerade Kubernetes pod traffic on WiFi
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now k8s-masquerade.service
```

**Verify:**
```bash
sudo iptables -t nat -L POSTROUTING -n -v | grep MASQUERADE
```

---

## Systematic Debugging Process

### Step 1: Isolate the Layer

```
                   ┌─────────────┐
                   │   Pod App   │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │ Container   │
                   │  Network    │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │ Pod Network │ ← Routes to services?
                   │  (eth0)     │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │ Host veth   │ ← Calico/CNI
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │  iptables   │ ← Rules applied?
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │ Host Route  │ ← Forwarding enabled?
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │  Physical   │ ← Firewall? MTU?
                   │  Interface  │
                   └──────┬──────┘
                          │
                   ┌──────▼──────┐
                   │   Router    │ ← Return path?
                   └─────────────┘
```

Test each layer from bottom to top.

### Step 2: Compare Working vs Broken

Always compare a working node to the broken one:

```bash
# Routes
diff <(ssh working-node "ip route") <(ssh broken-node "ip route")

# iptables
diff <(ssh working-node "sudo iptables -S") <(ssh broken-node "sudo iptables -S")

# Interfaces
diff <(ssh working-node "ip addr") <(ssh broken-node "ip addr")
```

### Step 3: Enable Packet Tracing

**iptables trace (most detailed):**
```bash
# On problem node
sudo iptables -t raw -A PREROUTING -s <pod-cidr> -j TRACE
sudo iptables -t raw -A OUTPUT -d <pod-cidr> -j TRACE
sudo dmesg -wT | grep TRACE
```

Watch where packets go through iptables chains.

**tcpdump (network level):**
```bash
# Capture on physical interface
sudo tcpdump -i <interface> -n 'host <pod-ip>' -v

# Capture on tunnel
sudo tcpdump -i tunl0 -n
```

Shows actual packets on the wire.

### Step 4: Check CNI Logs

```bash
# Calico
kubectl logs -n calico-system calico-node-<pod> -c calico-node --tail=100

# kube-proxy
kubectl logs -n kube-system kube-proxy-<pod> --tail=100
```

Look for errors about:
- Failed to create routes
- Failed to configure interfaces
- BGP peering issues

---

## Environment-Specific Notes

### WiFi Nodes

WiFi has inherent limitations for Kubernetes:
- Lower/variable MTU
- Some routers/drivers don't handle IPIP protocol properly
- Higher latency and jitter
- Packet loss

**Solutions:**
1. Use MASQUERADE to NAT pod traffic
2. Use VXLAN instead of IPIP (UDP-based, WiFi-friendly)
3. Switch to Ethernet if possible

### VPN Software (Tailscale, WireGuard, etc.)

VPN software adds extra network interfaces that can confuse CNI auto-detection.

**Fix:** Explicitly tell CNI which interface to use:

```yaml
# For Calico (Tigera Operator)
spec:
  calicoNetwork:
    nodeAddressAutodetectionV4:
      skipInterface: "tailscale0"
```

Or:
```yaml
spec:
  calicoNetwork:
    nodeAddressAutodetectionV4:
      interface: "eth0|enp.*|wlp.*"
```

### Home Lab / No Cloud Load Balancer

Home routers don't have:
- Routes to pod CIDRs
- BGP support
- Advanced routing policies

**Workarounds:**
- Use MASQUERADE for pod traffic on edge nodes
- Use MetalLB for LoadBalancer services (handles ARP announcements)
- Accept NodePort limitations

---

## Advanced Debugging Tools

### tcpdump Examples

```bash
# Capture pod traffic on specific interface
sudo tcpdump -i <interface> -n 'host <pod-ip>' -v

# Filter for API server traffic
sudo tcpdump -i any -n 'dst 10.96.0.1 and port 443' -v

# Watch for fragmentation issues
sudo tcpdump -i any -n 'icmp[icmptype] == 3' -v
```

### iptables Trace Analysis

```bash
# Enable tracing
sudo iptables -t raw -A PREROUTING -s <pod-cidr> -j TRACE
sudo iptables -t raw -A OUTPUT -s <pod-cidr> -j TRACE

# Watch kernel logs
sudo dmesg -wT | grep TRACE

# Look for the final chain - where did the packet go?
# Example: TRACE: ... mangle:POSTROUTING:policy:2 OUT=wlp59s0
```

Last chain shows where packet exited. If it exits but pod doesn't get response, return path is broken.

### Check Specific iptables Chains

```bash
# NAT rules (check MASQUERADE)
sudo iptables -t nat -L POSTROUTING -n -v

# Filter rules (check DROP/REJECT)
sudo iptables -L FORWARD -n -v

# Service routing (kube-proxy)
sudo iptables -t nat -L KUBE-SERVICES -n -v

# Check specific service
sudo iptables -t nat -L KUBE-SVC-NPX46M4PTMTKRN6Y -n -v  # Kubernetes API
```

### Verify CNI Configuration

```bash
# Check CNI config files
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist

# Verify CNI plugins installed
ls -la /opt/cni/bin/

# Check Calico node annotation
kubectl get node <node-name> -o yaml | grep -A5 projectcalico.org
```

---

## The WiFi + Kubernetes Problem (Case Study)

### Environment

- **Problem Node:** precision5540 with WiFi interface (wlp59s0)
- **Working Nodes:** Ethernet connections
- **CNI:** Calico with IPIP tunneling
- **Additional:** Tailscale VPN on all nodes

### Discovery Process

1. **Initial symptom:** Longhorn pods timing out on TLS handshakes to API server

2. **First check:** Host could reach API → Pod-level issue confirmed

3. **Route check:** Pod had no routes to service CIDR → CNI problem suspected

4. **iptables trace:** Packets leaving correctly via wlp59s0 → Outbound OK, return path broken

5. **tcpdump analysis:** Saw TCP SYN retransmissions from API server with same sequence numbers → Return packets not arriving

6. **MTU discovery:** Found "need to frag (mtu 1450)" ICMP messages → MTU issue identified

7. **Router routing:** Realized router had no route for pod CIDR (192.168.88.0/24) → Root cause found

### Root Cause

**Home router couldn't route return traffic to pod IPs.**

```
Pod (192.168.88.69) → Worker2 (WiFi) → Router → Master
Master replies to 192.168.88.69 → Router: "Unknown destination!" → Dropped
```

Ethernet nodes worked because IPIP encapsulation kept pod IPs hidden inside node-to-node packets. WiFi somehow exposed raw pod IPs to the router.

### Solution

```bash
# On WiFi node - NAT pod traffic to appear from node IP
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

This made all pod traffic appear to come from the node IP (192.168.1.105), which the router knows how to route.

---

## Prevention & Best Practices

### 1. Use Ethernet for Worker Nodes

WiFi introduces:
- MTU issues
- Packet loss
- Routing complications
- Higher latency

**Solution:** USB-to-Ethernet adapters are cheap and eliminate these problems.

### 2. Set Appropriate MTU from the Start

If using WiFi, configure lower MTU during cluster setup:

```yaml
# In Calico Installation
spec:
  calicoNetwork:
    mtu: 1400
```

### 3. Configure Interface Detection Explicitly

Don't rely on auto-detection with VPNs:

```yaml
spec:
  calicoNetwork:
    nodeAddressAutodetectionV4:
      skipInterface: "tailscale0|docker0|lo"
    # Or explicitly specify
    nodeAddressAutodetectionV4:
      interface: "eth0|enp.*"
```

### 4. Document Node-Specific Configurations

Keep track of special rules needed for each node:
- MTU settings
- iptables MASQUERADE rules
- Disabled services (systemd-resolved, etc.)

### 5. Test New Nodes Immediately

When adding a node, test pod connectivity right away:

```bash
# Create test pod on new node
kubectl run test --image=nicolaka/netshoot \
  --overrides='{"spec": {"nodeName": "new-node"}}' \
  --command -- sleep 300

# Test API connectivity
kubectl exec -it test -- curl -k https://kubernetes.default.svc:443/version

# Cleanup
kubectl delete pod test
```

---

## Quick Reference Commands

### Check Node Status

```bash
# Node readiness
kubectl get nodes -o wide

# Calico node status
kubectl get nodes.longhorn.io -n longhorn-system  # If using Longhorn

# CNI pod status
kubectl get pods -n calico-system -o wide
kubectl get pods -n kube-system | grep proxy
```

### Debug Pod Networking

```bash
# Create debug pod
kubectl run nettest --image=nicolaka/netshoot --command -- sleep 3600

# Or on specific node
kubectl run nettest --image=nicolaka/netshoot \
  --overrides='{"spec": {"nodeName": "<node-name>"}}' \
  --command -- sleep 3600

# Exec into pod
kubectl exec -it nettest -- bash

# Inside pod:
curl -k https://10.96.0.1:443/version
ping 10.96.0.1
ip route
ip addr
nslookup kubernetes.default.svc.cluster.local
```

### Check CNI Health

```bash
# Calico nodes
kubectl get pods -n calico-system -l k8s-app=calico-node

# Calico BGP status
kubectl exec -n calico-system calico-node-<pod> -- calicoctl node status

# kube-proxy logs
kubectl logs -n kube-system kube-proxy-<pod> --tail=50
```

### Restart CNI Components

```bash
# Restart Calico on specific node
kubectl delete pod -n calico-system calico-node-<pod>

# Restart kube-proxy on specific node
kubectl delete pod -n kube-system kube-proxy-<pod>

# Restart all (careful!)
kubectl rollout restart daemonset/calico-node -n calico-system
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

---

## When to Give Up & Switch Solutions

If you've tried everything and WiFi still doesn't work:

### Option 1: Use Ethernet
Get a USB-to-Ethernet adapter. Solves all WiFi-related issues instantly.

### Option 2: Different CNI
Try Flannel (uses VXLAN by default, more WiFi-friendly):
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Option 3: Remove Problematic Node
```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node <node>
```

Continue with remaining nodes until hardware issue is resolved.

---

## Lessons Learned

1. **Host connectivity ≠ Pod connectivity** - They use different network paths

2. **WiFi + Kubernetes = Pain** - Ethernet is strongly recommended

3. **iptables tracing is invaluable** - Shows exactly where packets go

4. **Home routers are dumb** - Don't expect pod CIDR routing

5. **MASQUERADE is often needed** - Especially on WiFi or edge nodes

6. **MTU matters** - WiFi has lower effective MTU than Ethernet

7. **Test immediately** - Don't deploy workloads before verifying pod networking

---

## TL;DR - Quick Fix for WiFi Nodes

If you're running a Kubernetes worker on WiFi and pods can't reach the API:

```bash
# On the WiFi node
sudo iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -o <wifi-interface> -j MASQUERADE
sudo systemctl restart kubelet
```

Replace `192.168.0.0/16` with your pod CIDR and `<wifi-interface>` with your WiFi interface name.

This NATs pod traffic to appear from the node IP, solving router routing issues.

---

**Created:** 2025-11-28  
**Based on:** Real-world WiFi node troubleshooting experience

