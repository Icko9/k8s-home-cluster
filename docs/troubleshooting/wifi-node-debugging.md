# Debugging Story: Why Pods on My WiFi Node Couldn't Reach the Kubernetes API

## The Setup

A 3-node Kubernetes homelab cluster:

| Node | IP | Interface | Role |
|------|-----|-----------|------|
| master | 192.168.1.50 | enp1s0 (Ethernet) | Control Plane |
| asuslpt | 192.168.1.104 | Ethernet | Worker |
| precision5540 | 192.168.1.105 | wlp59s0 (**WiFi**) | Worker |

- **CNI**: Calico with IPIP tunneling
- **Extra**: Tailscale VPN on all nodes
- **Pod CIDR**: 192.168.0.0/16

---

## The Problem

Longhorn manager pods on `precision5540` kept failing with:

```
Get "https://10.96.0.1:443/...": net/http: TLS handshake timeout
```

Pods on the other two nodes worked fine.

---

## Our Debugging Journey

### Step 1: Can the Host Reach the API Server?

First question: Is this a node-level problem or pod-level problem?

```bash
# On precision5540
curl -k https://10.96.0.1:443
```

**Result**: Got `403 Forbidden` - which is actually **good**! It means the host reached the API server (403 is an auth error, not a network error).

**Conclusion**: Host networking works. Problem is specific to pods.

---

### Step 2: Can Pods Reach Anything?

Created a test pod with networking tools:

```bash
kubectl run nettest --image=nicolaka/netshoot \
  --overrides='{"spec": {"nodeName": "precision5540"}}' \
  --command -- sleep 3600

kubectl exec -it nettest -- curl -k --max-time 5 https://kubernetes.default.svc:443/healthz
```

**Result**: `curl: (28) Connection timed out after 5002 milliseconds`

**Conclusion**: Confirmed - pods on precision5540 can't reach the API server.

---

### Step 3: Is Calico Detecting the Right Interface?

With Tailscale installed, maybe Calico picked the wrong network interface?

```bash
kubectl get node precision5540 -o yaml | grep -A10 projectcalico
```

**Result**:
```yaml
projectcalico.org/IPv4Address: 192.168.1.105/24
projectcalico.org/IPv4IPIPTunnelAddr: 192.168.88.64
projectcalico.org/Interfaces: '[{"name":"tailscale0",...},{"name":"wlp59s0","addresses":["192.168.1.105"]}]'
```

**Conclusion**: Calico correctly identified `wlp59s0` (WiFi) with IP `192.168.1.105`. Interface detection is fine.

---

### Step 4: Are the Routes Correct?

```bash
# On precision5540
ip route get 192.168.219.64   # Master's tunnel IP
```

**Result**:
```
192.168.219.64 via 192.168.1.50 dev tunl0 src 192.168.88.64
```

**Conclusion**: Routes look correct. Traffic to other nodes' pod networks goes through `tunl0` (IPIP tunnel).

---

### Step 5: Are iptables Rules Correct?

Maybe something is blocking traffic in the firewall?

```bash
sudo iptables -S FORWARD
sudo iptables -S cali-FORWARD
```

**Result**: All the expected Calico chains were present. Policy was ACCEPT.

We noticed `ts-forward` (Tailscale) in the FORWARD chain - suspicious! But checking it:

```bash
sudo iptables -S ts-forward
```

**Result**: It only handles Tailscale subnet (100.64.0.0/10), not pod traffic.

**Conclusion**: iptables rules look fine. Not a firewall issue.

---

### Step 6: Let's Trace the Packets

This is where we got serious. Enabled iptables packet tracing:

```bash
sudo iptables -t raw -A PREROUTING -s 192.168.88.0/24 -j TRACE
sudo iptables -t raw -A OUTPUT -s 192.168.88.0/24 -j TRACE
sudo dmesg -wT | grep TRACE
```

Then triggered traffic from a pod and watched the trace.

**Result**: We saw packets successfully traverse ALL iptables chains:
```
TRACE: raw:PREROUTING → mangle:PREROUTING → mangle:FORWARD → filter:FORWARD 
→ filter:cali-FORWARD → filter:cali-from-wl-dispatch → mangle:POSTROUTING 
→ mangle:POSTROUTING:policy:2 OUT=wlp59s0
```

Packets reached the final `POSTROUTING` policy and exited via `wlp59s0`!

**Conclusion**: **Outbound traffic is working!** Packets are leaving the node correctly. The problem must be on the return path.

---

### Step 7: The "Aha!" Moment

If outbound works but pods don't get responses, where are return packets dying?

Think about what happens:

```
1. Pod (192.168.88.69) sends packet to API server (192.168.1.50)
2. Packet leaves precision5540 via WiFi
3. Router forwards to master
4. Master responds... to 192.168.88.69
5. Router receives response... "Who is 192.168.88.69?"
```

**The router only knows about 192.168.1.0/24!** It has no idea that 192.168.88.x (pod network) lives behind 192.168.1.105 (precision5540).

On Ethernet nodes, IPIP encapsulation wraps pod-to-pod traffic inside node-to-node packets, so the router only sees 192.168.1.x addresses. But something about the WiFi node was exposing raw pod IPs to the router.

---

### Step 8: Testing the Theory

If the theory is correct, NAT-ing pod traffic to the node IP should fix it:

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

Then tested immediately:

```bash
kubectl exec -it nettest -- curl -k --max-time 5 https://kubernetes.default.svc:443/healthz
```

**Result**: `ok`

**IT WORKED!**

---

## The Root Cause

**Home routers don't have routes to pod CIDRs.**

When traffic from pods on the WiFi node left with a pod source IP (192.168.88.x), the router couldn't route return traffic back because it only knows about the node network (192.168.1.0/24).

### Why It Worked on Ethernet Nodes

On Ethernet nodes, Calico's IPIP tunneling properly encapsulated pod traffic inside IP-in-IP packets between node IPs. The router only ever saw 192.168.1.x traffic.

### Why WiFi Was Different

Something about the WiFi path (possibly how IPIP interacts with WiFi, or routing decisions) caused raw pod IPs to be exposed to the router on certain traffic paths.

---

## What MASQUERADE Does

```
BEFORE:
Pod (192.168.88.69) → Router → Master
Master replies to 192.168.88.69 → Router: "???" → DROP

AFTER (with MASQUERADE):
Pod (192.168.88.69) → NAT rewrites to 192.168.1.105 → Router → Master
Master replies to 192.168.1.105 → Router: "That's precision5540!" → Delivered
precision5540 un-NATs back to 192.168.88.69 → Pod gets response
```

---

## The Fix

### Temporary (test)
```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

### Permanent (survives reboot)
```bash
sudo tee /etc/systemd/system/k8s-masquerade.service << 'EOF'
[Unit]
Description=Masquerade for Kubernetes pods on WiFi
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable k8s-masquerade.service
```

---

## What We Tried That Didn't Work

### 1. MASQUERADE Only (with IPIP + BGP Enabled)

After discovering the MASQUERADE fix, we tried reverting Calico to defaults:

```yaml
spec:
  calicoNetwork:
    bgp: Enabled
    ipPools:
    - encapsulation: IPIP
```

**Result**: Failed! BGP peering from the WiFi node couldn't establish:

```
calico/node is not ready: BIRD is not ready: BGP not established with 192.168.1.104,192.168.1.50
```

BGP peering itself has the same routing problem - the WiFi node can't properly communicate with other nodes for BGP even with MASQUERADE (which only covers pod traffic, not BGP control plane).

### 2. VXLAN + BGP Disabled (without MASQUERADE)

```yaml
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - encapsulation: VXLAN
```

**Result**: Still didn't work alone. Pods still couldn't reach the API server.

### 3. MASQUERADE Only (without Calico changes)

Just adding the iptables rule without changing Calico config.

**Result**: Worked momentarily for the test, but BGP issues caused instability.

---

## The Actual Working Solution

**It requires BOTH changes:**

### 1. Patch Calico: VXLAN + BGP Disabled

```bash
kubectl patch installation default --type=merge -p '
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
    nodeAddressAutodetectionV4:
      skipInterface: tailscale0
'
```

**Why VXLAN?** VXLAN uses UDP (port 4789) which WiFi handles like normal traffic. IPIP is a raw IP protocol (protocol 4) that can behave unpredictably on WiFi.

**Why BGP Disabled?** BGP peering between nodes has the same routing issues on WiFi. Without BGP, Calico uses direct routing based on node annotations instead.

### 2. MASQUERADE Rule on WiFi Node

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

**Why still needed?** Even with VXLAN, some traffic paths expose pod IPs to the router. MASQUERADE ensures all pod traffic appears from the node IP.

### Make MASQUERADE Permanent

```bash
sudo tee /etc/systemd/system/k8s-masquerade.service << 'EOF'
[Unit]
Description=Masquerade for Kubernetes pods on WiFi
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable k8s-masquerade.service
```

---

## Key Debugging Insights

### 1. "Host Works, Pods Don't" = Return Path Problem

When the host can reach destinations but pods can't, it's almost always a routing issue on the return path. Packets go out, responses can't come back.

### 2. iptables Tracing is Invaluable

```bash
sudo iptables -t raw -A PREROUTING -s <pod-cidr> -j TRACE
sudo dmesg -wT | grep TRACE
```

This showed us packets were leaving correctly, which ruled out all the iptables/Calico configuration issues and pointed us to the return path.

### 3. Think About What the Router Sees

Home routers are simple devices. They know:
- 192.168.1.0/24 (your home network)
- 0.0.0.0/0 → internet gateway

They DON'T know:
- 192.168.88.0/24 (pod network on node A)
- 192.168.170.0/24 (pod network on node B)
- 10.96.0.0/12 (service network)

If a packet arrives with destination 192.168.88.69, the router has no idea where to send it.

### 4. WiFi Adds Complexity

WiFi introduces variables that Ethernet doesn't have:
- Different MTU handling
- Power management
- Driver quirks with encapsulation protocols
- Potentially different routing behavior

When possible, use Ethernet for Kubernetes nodes.

---

## Final Working Configuration

**Both changes are required:**

| Component | Setting | Why |
|-----------|---------|-----|
| Calico Encapsulation | **VXLAN** | UDP-based, WiFi-friendly |
| Calico BGP | **Disabled** | BGP peering fails on WiFi |
| iptables (WiFi node) | **MASQUERADE** | NAT pod traffic to node IP |
| Interface Detection | `skipInterface: tailscale0` | Avoid Tailscale confusion |

```yaml
# Calico Installation
spec:
  calicoNetwork:
    bgp: Disabled
    ipPools:
    - cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
    nodeAddressAutodetectionV4:
      skipInterface: tailscale0
```

```bash
# On WiFi node only
iptables -t nat -A POSTROUTING -s 192.168.88.0/24 -o wlp59s0 -j MASQUERADE
```

---

## TL;DR

**Problem**: Pods on WiFi node couldn't reach API server.

**Root Cause**: Combination of:
1. Home router can't route pod IPs
2. IPIP encapsulation doesn't work well on WiFi
3. BGP peering fails from WiFi node

**Solution**: Two changes required:
1. Patch Calico: `encapsulation: VXLAN` + `bgp: Disabled`
2. Add MASQUERADE rule on WiFi node

**Lesson**: WiFi + Kubernetes requires special handling. Use Ethernet if possible, or be prepared for VXLAN + MASQUERADE combo.

---

**Debugging Time**: ~2 hours  
**Actual Fix**: 1 kubectl patch + 1 iptables command  
**Lesson Value**: Priceless 😄
