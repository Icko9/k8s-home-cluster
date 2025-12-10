# Homelab Kubernetes Cluster Postmortem

**Date:** December 10, 2025  
**Duration:** ~2 hours debugging  
**Severity:** Cluster networking down  

---

## Incident Summary

Returned from France trip to find Kubernetes homelab cluster networking broken. Pods failing to create with DNS resolution errors, Calico not establishing connectivity between nodes.

---

## Timeline

| Time | Event |
|------|-------|
| Dec 4, 19:05 | Laptop (asuslpt) rebooted due to kernel update (6.8.0-87 → 6.8.0-88) |
| Dec 4 - Dec 10 | Cluster offline while traveling |
| Dec 10, ~16:30 | Discovered bridge `br0` missing, USB ethernet adapter DOWN |
| Dec 10, ~16:45 | Recreated bridge config in netplan |
| Dec 10, ~16:50 | Fixed bridge IP (was 192.168.100.104, needed 192.168.1.104) |
| Dec 10, ~17:00 | Discovered systemd-resolved not running, pods failing with `/run/systemd/resolve/resolv.conf: no such file or directory` |
| Dec 10, ~17:10 | Added MASQUERADE iptables rule to **wrong node** (asuslpt instead of precision) |
| Dec 10, ~17:20 | Removed incorrect iptables rules from asuslpt |

---

## Root Causes

### 1. Bridge Configuration Lost
- Kernel update triggered automatic reboot while away
- Bridge configuration was NOT persisted in netplan
- Only WiFi config survived in `/etc/netplan/01-network-manager-all.yaml`
- USB ethernet adapter `enx00e04c361c71` came up DOWN with no config

### 2. Wrong IP Initially Assigned
- Accidentally configured bridge as `192.168.100.104/24` instead of `192.168.1.104/24`
- Wrong subnet would have isolated the bridge from the home network

### 3. MASQUERADE Rule on Wrong Node
- The WiFi worker node is **precision**, not asuslpt
- asuslpt connects via bridge/ethernet
- MASQUERADE rule needed on WiFi node to NAT pod traffic (192.168.0.0/16) through WiFi interface

---

## Network Architecture Reminder

```
┌─────────────────────────────────────────────────────────────┐
│                     Home Router                              │
│                    192.168.1.1                               │
└─────────────────┬───────────────────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
    ▼             ▼             ▼
┌────────┐  ┌──────────┐  ┌───────────┐
│ master │  │ asuslpt  │  │ precision │
│  VM    │  │ (bridge) │  │  (WiFi)   │
│        │  │192.168.1.│  │192.168.1. │
│        │  │   104    │  │   105     │
└────────┘  └──────────┘  └───────────┘
    │             │             │
    └─────────────┴─────────────┘
              │
        Calico VXLAN
        Pod CIDR: 192.168.0.0/16
```

**Key Point:** precision is the WiFi node that needs MASQUERADE because home router can't route pod IPs back through WiFi.

---

## Calico Configuration

Working configuration (VXLAN mode, BGP disabled):

```yaml
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

---

## Required MASQUERADE Rule (on precision only!)

```bash
# On PRECISION (WiFi worker node)
iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -o wlp59s0 -j MASQUERADE
```

**Check if exists:**
```bash
iptables -t nat -L POSTROUTING -n -v | grep 192.168
```

**Remove if wrong:**
```bash
iptables -t nat -D POSTROUTING -s 192.168.0.0/16 -o <interface> -j MASQUERADE
```

---

## Action Items

- [ ] **Persist bridge config** - Verify netplan config is complete and backed up
- [ ] **Persist MASQUERADE rule** on precision - Add to `/etc/rc.local` or use `iptables-persistent`
- [ ] **Document node roles clearly:**
  - `master` - Control plane (VM on asuslpt)
  - `asuslpt` - Worker node via bridge (192.168.1.104)
  - `precision` - Worker node via WiFi (192.168.1.105) - **needs MASQUERADE**
- [ ] **Consider disabling automatic reboots** for kernel updates

---

## Commands Cheat Sheet

```bash
# Check bridge status
brctl show
ip addr show br0

# Check netplan config
cat /etc/netplan/*.yaml

# Apply netplan
sudo netplan apply

# Check iptables NAT rules
iptables -t nat -L POSTROUTING -n -v

# Check last reboot
who -b
last reboot | head -5

# Check Calico config
kubectl get installation default -o yaml | grep -A 30 "^spec:"
kubectl get ippool -o yaml | grep -iE "encapsulation|vxlan|ipip"

# Check VXLAN tunnel
ip addr | grep -i vxlan

# Restart calico-node pods
kubectl delete pods -n calico-system -l k8s-app=calico-node
```

---

## Lessons Learned

1. **Always persist network configs** - Bridge configs don't survive reboot without netplan
2. **Know your node roles** - WiFi node needs special handling (MASQUERADE)
3. **Document the architecture** - Easy to forget which node is which after time away
4. **Kernel updates can reboot** - Consider unattended-upgrades config
