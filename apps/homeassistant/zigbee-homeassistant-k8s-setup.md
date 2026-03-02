# Zigbee Dongle + Home Assistant on Kubernetes — Setup Guide

## Hardware

- **Dongle:** Sonoff Zigbee 3.0 USB Dongle Plus V2 (Model: ZBDongle-E)
- **Chip:** EFR32MG21 (Silicon Labs)
- **Server:** Ubuntu (hostname: `precision5540`)
- **Deployment target:** Existing Kubernetes (AKS-style) cluster

---

## Step 1 — Verify Dongle is Detected

Plug the dongle into the Ubuntu server and verify detection:

```bash
ls /dev/serial/by-id/
```

Expected output:
```
usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_b2631a778112f01198f90914773d9da9-if00-port0
```

Also verify via USB:
```bash
lsusb | grep -i silabs
```

> **Why use the server?** The dongle needs to be always-on and connected to the same node where the Zigbee2MQTT pod runs. A desktop is unsuitable due to reboots and power-offs.

---

## Step 2 — Flash Firmware

The ZBDongle-E ships with older firmware. Flash the latest **EmberZNet NCP** firmware for full Zigbee2MQTT and ZHA compatibility.

### Firmware

| Property | Value |
|---|---|
| File | `ncp-uart-hw-v7.4.5.0-zbdonglee-115200.gbl` |
| Protocol | EZSP (EmberZNet Serial Protocol) |
| Baudrate | 115200 |
| Flow control | Disabled (broken in 2024.12.x builds) |

> **Do NOT use** RCP Multi-PAN firmware — it's no longer recommended due to instability with concurrent Zigbee + Thread.

### Flash Commands

```bash
# Install flasher
pip install universal-silabs-flasher --break-system-packages

# Download firmware
wget https://github.com/darkxst/silabs-firmware-builder/raw/main/firmware_builds/zbdonglee/ncp-uart-hw-v7.4.5.0-zbdonglee-115200.gbl

# Flash
universal-silabs-flasher \
  --device /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_b2631a778112f01198f90914773d9da9-if00-port0 \
  flash \
  --firmware ncp-uart-hw-v7.4.5.0-zbdonglee-115200.gbl
```

---

## Step 3 — Kubernetes Architecture

Home Assistant **Supervisor/OS mode is not used** — instead deploy individual containers as K8s workloads.

### Stack

| Service | Image | Purpose |
|---|---|---|
| Home Assistant | `homeassistant/home-assistant:stable` | Main HA instance |
| Zigbee2MQTT | `koenkk/zigbee2mqtt` | Zigbee coordinator bridge |
| Mosquitto | `eclipse-mosquitto` | MQTT broker |

**Data flow:**
```
[Zigbee devices] → [Dongle] → [Zigbee2MQTT pod] → [Mosquitto/MQTT] → [Home Assistant]
```

---

## Step 4 — USB Passthrough to Kubernetes

The dongle is physically on one node. Pin the Zigbee2MQTT pod to that node and mount the device.

### Key points

- Use the stable `by-id` path, NOT `/dev/ttyUSB0` (changes on reboot)
- Pod needs `privileged: true` or at minimum `allowPrivilegeEscalation`
- Use `nodeName` or `nodeSelector` to pin the pod

### Example Zigbee2MQTT Deployment snippet

```yaml
spec:
  nodeName: precision5540   # pin to node with the dongle
  containers:
    - name: zigbee2mqtt
      image: koenkk/zigbee2mqtt
      securityContext:
        privileged: true
      volumeMounts:
        - name: zigbee-dongle
          mountPath: /dev/ttyUSB0
        - name: z2m-data
          mountPath: /app/data
  volumes:
    - name: zigbee-dongle
      hostPath:
        path: /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_b2631a778112f01198f90914773d9da9-if00-port0
        type: CharDevice
    - name: z2m-data
      persistentVolumeClaim:
        claimName: zigbee2mqtt-data
```

---

## Step 5 — Zigbee2MQTT Configuration

`configuration.yaml` for Zigbee2MQTT:

```yaml
homeassistant: true
permit_join: true

mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883   # K8s service name

serial:
  port: /dev/ttyUSB0
  baudrate: 115200
  adapter: ezsp                   # required for EFR32MG21 chip

advanced:
  log_level: info
```

---

## Notes

- **No Supervisor** = no HA add-ons. Deploy Zigbee2MQTT, Mosquitto, etc. as separate K8s pods instead.
- **Persistence** is critical — use PVCs for HA config, Z2M data, and Mosquitto data.
- **Only one pod** needs the USB device (Zigbee2MQTT). HA communicates over MQTT, no USB access needed for HA itself.
- Firmware source: [darkxst/silabs-firmware-builder](https://github.com/darkxst/silabs-firmware-builder)
