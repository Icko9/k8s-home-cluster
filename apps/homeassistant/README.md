# Home Assistant on Kubernetes — Deployment Guide

## Overview

This guide documents the deployment of Home Assistant with Zigbee support on a Kubernetes cluster using Zigbee2MQTT and Mosquitto MQTT broker.

---

## Architecture

```
[Zigbee devices] → [Sonoff Dongle] → [Zigbee2MQTT] → [Mosquitto] → [Home Assistant]
                                                            ↑
[WiFi devices (Tuya)] ──── Tuya Cloud ──────────────────────┘
```

### Components

| Service | Image | Purpose |
|---|---|---|
| Home Assistant | `homeassistant/home-assistant:stable` | Main automation hub |
| Zigbee2MQTT | `koenkk/zigbee2mqtt:latest` | Zigbee coordinator bridge |
| Mosquitto | `eclipse-mosquitto:latest` | MQTT message broker |

---

## Directory Structure

```
apps/homeassistant/
├── namespace.yaml
├── mosquitto/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── zigbee2mqtt/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── homeassistant/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

---

## Deployment Order

```bash
# 1. Create namespace
kubectl apply -f apps/homeassistant/namespace.yaml

# 2. Deploy Mosquitto (MQTT broker)
kubectl apply -f apps/homeassistant/mosquitto/

# 3. Deploy Zigbee2MQTT
kubectl apply -f apps/homeassistant/zigbee2mqtt/

# 4. Deploy Home Assistant
kubectl apply -f apps/homeassistant/homeassistant/
```

---

## Hardware

- **Zigbee Dongle:** Sonoff Zigbee 3.0 USB Dongle Plus V2 (ZBDongle-E)
- **Chip:** EFR32MG21 (Silicon Labs)
- **Protocol:** EZSP (EmberZNet Serial Protocol)
- **Node:** `precision5540` (dongle physically connected here)

---

## Key Configuration Details

### Zigbee2MQTT

The Zigbee2MQTT deployment uses an **init container** to copy the configuration from a ConfigMap to a writable volume. This is necessary because:
1. ConfigMaps are read-only
2. Zigbee2MQTT needs to write to `configuration.yaml` during operation

```yaml
initContainers:
  - name: copy-config
    image: busybox
    command:
      - sh
      - -c
      - |
        cp /config/configuration.yaml /app/data/configuration.yaml
```

### USB Passthrough

The Zigbee dongle is passed through to the container using:
- `nodeName: precision5540` — pins pod to the node with the dongle
- `privileged: true` — required for USB device access
- `hostPath` with stable `/dev/serial/by-id/` path (not `/dev/ttyUSB0`)

```yaml
volumes:
  - name: zigbee-dongle
    hostPath:
      path: /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_...-if00-port0
      type: CharDevice
```

### Data Persistence

All services use `hostPath` volumes for persistence:

| Service | Path on Node |
|---|---|
| Mosquitto | `/opt/k8s-data/mosquitto/data` |
| Zigbee2MQTT | `/opt/k8s-data/zigbee2mqtt/data` |
| Home Assistant | `/opt/k8s-data/homeassistant/config` |

---

## Access URLs

| Service | URL |
|---|---|
| Home Assistant | `http://homeassistant.local:8123` |
| Zigbee2MQTT | `http://zigbee2mqtt.local:8080` |

Or via port-forward:
```bash
kubectl port-forward svc/homeassistant 8123:8123 -n homeassistant
kubectl port-forward svc/zigbee2mqtt 8080:8080 -n homeassistant
```

---

## Home Assistant Integration

### MQTT (for Zigbee devices)

1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **MQTT**
3. Enter broker: `mosquitto.homeassistant.svc.cluster.local` or `mosquitto`
4. Port: `1883`
5. No authentication required

Zigbee devices paired in Zigbee2MQTT will auto-discover in Home Assistant.

### Tuya (for WiFi devices)

1. Create account at https://iot.tuya.com/
2. Create Cloud Project (Western Europe Data Center)
3. Subscribe to APIs: IoT Core, Authorization Token Management, Smart Home Basic Service
4. Link Smart Life app account in Devices tab
5. In Home Assistant: **Settings → Add Integration → Tuya**
6. Enter User Code from Smart Life app (Settings → Account and Security → User Code)

---

## Pairing Zigbee Devices

1. Open Zigbee2MQTT UI (`http://zigbee2mqtt.local:8080`)
2. Click **"Permit join (All)"** (bottom left)
3. Put device in pairing mode (usually hold button 5-10 seconds)
4. Device appears in Devices list within 30 seconds
5. Auto-discovered in Home Assistant via MQTT

---

## Tested Devices

| Device | Type | Protocol | Status |
|---|---|---|---|
| Tuya TS011F Smart Plug | Outlet with power monitoring | Zigbee | ✓ Working |
| Tuya Temperature Sensor | Temp & Humidity | WiFi (Tuya Cloud) | Requires Tuya integration |

---

## Troubleshooting

### Zigbee2MQTT: "read-only file system" error

**Cause:** ConfigMap mounted directly to `/app/data/configuration.yaml`

**Solution:** Use init container to copy config to writable hostPath volume

### Zigbee2MQTT: "configuration.yaml is expected to be an object"

**Cause:** Corrupted or empty config file on node

**Solution:** 
```bash
# On the node (precision5540):
sudo rm -rf /opt/k8s-data/zigbee2mqtt/data/*
# Then restart pod:
kubectl rollout restart deployment/zigbee2mqtt -n homeassistant
```

### Ingress: "configuration-snippet annotation cannot be used"

**Cause:** Snippet directives disabled by ingress controller

**Solution:** Use standard websocket annotations instead:
```yaml
annotations:
  nginx.ingress.kubernetes.io/websocket-services: "homeassistant"
  nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
```

### EZSP driver deprecation warning

**Note:** The `ezsp` adapter is deprecated for firmware 7.4.x+. Consider updating config to use `adapter: ember` in the future.

---

## References

- [Zigbee2MQTT Documentation](https://www.zigbee2mqtt.io/)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt/)
- [Tuya Integration](https://www.home-assistant.io/integrations/tuya/)
- [Silabs Firmware Builder](https://github.com/darkxst/silabs-firmware-builder)
