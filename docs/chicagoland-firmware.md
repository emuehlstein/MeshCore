---
title: Chicagoland Firmware
---

# Chicagoland MeshCore Firmware

The **Chicagoland firmware** is a pre-configured build of [MeshCore](https://meshcore.io) optimized for the Chicago-area mesh network. It's a fork of [agessaman/MeshCore `mqtt-bridge-implementation-flex`](https://github.com/agessaman/MeshCore/tree/mqtt-bridge-implementation-flex) with Chicagoland-specific radio parameters, MQTT broker presets, and regional defaults — so your node works on the Chicago mesh right out of the box.

!!! info "Current Version"
    **v1.15.0-chioff-0.5.3** (based on MeshCore v1.15.0)  
    [Download firmware →](https://github.com/emuehlstein/MeshCore/releases/tag/chicagoland-v0.5.3)

---

## What's Pre-Configured

When you flash the Chicagoland firmware, these settings are baked in — no manual configuration required:

| Setting | Value |
|---------|-------|
| **Frequency** | 910.525 MHz |
| **Bandwidth** | 62.5 kHz |
| **Spread Factor** | 7 |
| **Coding Rate** | 5 |
| **MQTT IATA Region** | `ORD` (Chicago) |
| **Path Hash Mode** | 2 (3-byte) |
| **Loop Detection** | Moderate |

These are the standard Chicagoland mesh radio parameters. All nodes on the network use these settings.

---

## Supported Hardware

The firmware ships pre-built binaries for the following boards:

| Board | PSRAM | Notes |
|-------|-------|-------|
| **Heltec V3** | ❌ | Most common starter board |
| **Heltec V4** | ✅ | |
| **Heltec V4 Expansion Kit** | ✅ | |
| **Heltec WSL3** | ❌ | Wireless Stick Lite V3 |
| **Station G2** | ✅ | High-power base station |

Each board has three firmware variants:

| Variant | Use Case |
|---------|----------|
| `*_repeater_observer_mqtt` | **Repeater** — routes packets and reports to MQTT. Best for nodes with good line-of-sight. |
| `*_dedicated_observer_mqtt` | **Dedicated observer** — reports to MQTT but does not route packets. Good for monitoring. |
| `*_room_server_observer_mqtt` | **Room server** — handles local messaging and reports to MQTT. Good for community hubs. |


---

## Flashing the Firmware

### Step 1: Download the firmware

Go to the [latest release page](https://github.com/emuehlstein/MeshCore/releases/tag/chicagoland-v0.5.3) and download the `*-merged.bin` file for your board and desired role.

!!! example "Example"
    For a Heltec V3 repeater, download:  
    `Heltec_v3_repeater_observer_mqtt-v1.15.0-chioff-0.5.3-xxxxxxx-merged.bin`

### Step 2: Open the web flasher

1. Go to [flasher.meshcore.co.uk](https://flasher.meshcore.co.uk/) in **Chrome** or **Edge** (Web Serial requires a Chromium-based browser)
2. Connect your device via USB
3. Click **"Custom Firmware"** (not a stock firmware option)

### Step 3: Flash the device

1. Select your downloaded `-merged.bin` file
2. **Enable "Erase device"** — this ensures a clean flash with the Chicago defaults
3. Click **Flash**
4. Wait for the flash to complete

!!! warning "Use the merged binary"
    Always use the `*-merged.bin` files. These include the bootloader + partition table + application and are safe to flash at offset `0x0`. The non-merged `.bin` files require manual offset configuration.

!!! danger "Antenna First"
    **Never power on your device without an antenna connected.** This can permanently damage the radio hardware.

---

## Post-Flash Configuration

After flashing, connect to your node via serial terminal (115200 baud) and run these commands:

```
set name <your-node-name>
set wifi.ssid <your-wifi-network>
set wifi.pwd <your-wifi-password>
set lat <your-latitude>
set lon <your-longitude>
reboot
```

!!! info "MQTT region is pre-set"
    `mqtt.iata` is already set to `ORD` — you do **not** need to set it manually unless you're deploying outside the Chicago area.

### Setting your location

Your latitude and longitude help the mesh map show where nodes are. You can find your coordinates at [latlong.net](https://www.latlong.net/).

```
set lat 41.8781
set lon -87.6298
```

!!! tip "Privacy"
    You don't need to use your exact location. A nearby intersection or general area is fine.

### Verifying your configuration

After rebooting, you can verify your settings:

```
get name
get wifi.ssid
get lat
get lon
get mqtt.iata
```

---

## Pre-Loaded MQTT Brokers

The Chicagoland firmware comes with MQTT broker presets pre-loaded. Your node will automatically connect to these brokers and begin reporting to the mesh network.

**Default active slots (all boards):**

| Slot | Preset | Broker | Auth |
|------|--------|--------|------|
| 0 | `analyzer-us` | LetsMesh US | JWT |
| 1 | `chimesh` | ChiMesh.org | JWT |
| 2 | `chioff` | Chicago Offline | JWT |

You can view and change your active MQTT slots via serial:

```
mqtt.preset 0          # Show slot 0 preset
mqtt.preset 0 chimesh  # Set slot 0 to chimesh
mqtt.presets           # List all available presets
```

### All available presets

These presets are compiled into the firmware and available on any slot:

| Preset | URL | Auth |
|--------|-----|------|
| `analyzer-us` | LetsMesh US | JWT |
| `analyzer-eu` | LetsMesh EU | JWT |
| `chimesh` | ChiMesh.org | JWT |
| `chioff` | Chicago Offline (prod) | JWT |
| `chioff-dev` | Chicago Offline (dev) | JWT |
| `meshmapper` | MeshMapper | JWT |
| `meshrank` | MeshRank | None |
| `waev` | WAEV | JWT |
| `meshomatic` | Meshomatic | JWT |
| `cascadiamesh` | Cascadia Mesh | JWT |
| `tennmesh` | Tennessee Mesh | User/Pass |
| `nashmesh` | Nashville Mesh | User/Pass |
| `meshat.se` | meshat.se | User/Pass |
| `eastidahomesh` | East Idaho Mesh | None |
| `coloradomesh` | Colorado Mesh | JWT |

---

## Web Config Tool

If you prefer a graphical interface over serial commands, you can use the **MeshCore Config Tool**:

[chicago-offline.github.io/meshcore-config](https://chicago-offline.github.io/meshcore-config/)

- Works in **Chrome** and **Edge** (Web Serial API required)
- Connect your node via USB
- Configure name, WiFi, location, and MQTT settings through a browser UI

---

## Verifying Your Node is Online

Once your node is configured and connected to WiFi, it will begin reporting to the MQTT brokers. You can verify it's working by checking:

- **[corescope.chimesh.org](https://corescope.chimesh.org)** — CoreScope packet analyzer, shows live mesh traffic

Your node should appear on these dashboards within a few minutes of coming online.

---

## Troubleshooting

### Node not appearing on the map

1. Verify WiFi is connected: `get wifi.ssid` and check for an IP address in the serial output after reboot
2. Verify MQTT is connected: look for MQTT connection messages in the serial output
3. Verify location is set: `get lat` and `get lon` should return non-zero values
4. Check your antenna connection

### MQTT connection failures

- Ensure your WiFi network has internet access
- JWT-authenticated brokers handle auth automatically — no passwords to configure
- Try rebooting: `reboot`

### Wrong frequency or can't hear other nodes

If you flashed stock MeshCore firmware previously, make sure you used the **Chicagoland firmware** (`*-chioff-*` in the filename) and enabled **"Erase device"** during flashing to clear old settings.

---

## Source Code

The Chicagoland firmware is open source:

- **Fork:** [emuehlstein/MeshCore](https://github.com/emuehlstein/MeshCore) — branch [`chioff-flex`](https://github.com/emuehlstein/MeshCore/tree/chioff-flex)
- **Upstream:** [agessaman/MeshCore](https://github.com/agessaman/MeshCore/tree/mqtt-bridge-implementation-flex) — `mqtt-bridge-implementation-flex` branch
- **Releases:** [GitHub Releases](https://github.com/emuehlstein/MeshCore/releases)

---

## Getting Help

- **Discord:** Join the [Chicagoland Mesh Discord](https://chicagolandmesh.org/discord) for community support
- **MeshCore docs:** [docs.meshcore.io](https://docs.meshcore.io/)
