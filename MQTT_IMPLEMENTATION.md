# MQTT Bridge Implementation for MeshCore

Native WiFi + MQTT observer firmware for MeshCore repeaters. Publishes mesh packet data to multiple MQTT brokers simultaneously.

## Quick Start

### 1. Flash the firmware

**Heltec V3:**
```bash
esptool.py --chip esp32s3 --baud 921600 write_flash \
  0x0000 bootloader.bin 0x8000 partitions.bin 0x10000 firmware.bin
```

**Station G2:**
```bash
esptool.py --chip esp32s3 --baud 921600 write_flash \
  0x0000 bootloader.bin 0x8000 partitions.bin 0x10000 firmware.bin
```

Or use [flasher.meshcore.io](https://flasher.meshcore.io) with the firmware.bin file.

### 2. Configure WiFi (serial console, 115200 baud)
```
set wifi.ssid YourNetwork
set wifi.pwd YourPassword
reboot
```

### 3. Verify
```
get wifi.status
get mqtt.origin
get mqtt.iata
```

That's it. The device connects to WiFi, syncs time via NTP, and starts publishing to all configured brokers.

---

## Serial CLI Reference

All commands are available via serial console (115200 baud) or LoRa repeater console. Commands marked ⚠️ are serial-only (blocked over LoRa for security).

### WiFi

| Command | Description |
|---------|-------------|
| `get wifi.ssid` | Current WiFi SSID |
| `get wifi.pwd` | Current WiFi password |
| `get wifi.status` | Connection state, IP address, RSSI |
| `get wifi.powersave` | Power save mode (none/min/max) |
| `set wifi.ssid <ssid>` | Set WiFi SSID |
| `set wifi.pwd <password>` | Set WiFi password |
| `set wifi.powersave <mode>` | Set power save: `none`, `min`, or `max` |

**Notes:**
- `wifi.powersave none` = best latency, highest power draw
- `wifi.powersave min` = default, balances power and latency
- `wifi.powersave max` = lowest power, may increase MQTT latency
- Changes to `wifi.powersave` apply immediately if WiFi is connected

### MQTT — Identity & Publishing

| Command | Description |
|---------|-------------|
| `get mqtt.origin` | Device origin name (used in JSON payloads) |
| `get mqtt.iata` | IATA region code (used in MQTT topic path) |
| `set mqtt.origin <name>` | Set origin name (defaults to device name) |
| `set mqtt.iata <code>` | Set IATA code (auto-uppercased, e.g. `ORD`) |

### MQTT — Message Types

| Command | Description | Default |
|---------|-------------|---------|
| `get/set mqtt.status on\|off` | Status messages (online/offline, device info) | on |
| `get/set mqtt.packets on\|off` | Decoded packet messages (type, route, SNR, RSSI) | on |
| `get/set mqtt.raw on\|off` | Raw hex packet data | off |
| `get/set mqtt.tx on\|off` | Include transmitted packets (not just received) | off |
| `get/set mqtt.interval <minutes>` | Status publish interval (1–60 minutes) | 5 min |

**Note:** `mqtt.interval` takes minutes (not milliseconds). Setting `set mqtt.interval 3` publishes status every 3 minutes. Changing the interval restarts the bridge to apply immediately.

### MQTT — Custom Broker (Broker Slot 0)

| Command | Description |
|---------|-------------|
| `get mqtt.server` | Custom broker hostname |
| `get mqtt.port` | Custom broker port |
| `get mqtt.username` | Broker username |
| `get mqtt.password` | Broker password |
| `get mqtt.tls` | TLS enabled (on/off) |
| `get mqtt.ws` | WebSocket mode enabled (on/off) |
| `set mqtt.server <host>` | Set broker hostname |
| `set mqtt.port <port>` | Set broker port (1–65535) |
| `set mqtt.username <user>` | Set username |
| `set mqtt.password <pass>` | Set password |
| `set mqtt.tls on\|off` | Enable/disable TLS |
| `set mqtt.ws on\|off` | Enable/disable WebSocket mode |

**Custom broker notes:**
- When `mqtt.ws` is `on`, the broker connects via WebSocket (`wss://` if TLS also on)
- When `mqtt.ws` is `on`, username/password are optional — JWT auth is used instead
- When `mqtt.ws` is `off`, username and password are required for the broker to be considered valid
- Use `get mqtt.config.valid` to check if your custom broker config passes validation

### MQTT — LetsMesh Analyzer Servers

| Command | Description | Default |
|---------|-------------|---------|
| `get/set mqtt.analyzer.us on\|off` | US server (mqtt-us-v1.letsmesh.net:443) | on |
| `get/set mqtt.analyzer.eu on\|off` | EU server (mqtt-eu-v1.letsmesh.net:443) | on |

LetsMesh servers use WebSocket + TLS + JWT authentication automatically. No credentials needed — the device signs tokens with its Ed25519 key.

### MQTT — Device Registration (LetsMesh)

| Command | Description |
|---------|-------------|
| ⚠️ `get mqtt.owner` | Owner's public key (serial only) |
| ⚠️ `get mqtt.email` | Owner's email (serial only) |
| ⚠️ `set mqtt.owner <64-hex-chars>` | Set owner public key (serial only) |
| ⚠️ `set mqtt.email <email>` | Set owner email (serial only) |

These are used for LetsMesh device registration. The owner public key must be exactly 64 hex characters (32 bytes).

### MQTT — Diagnostics

| Command | Description |
|---------|-------------|
| `get mqtt.config.valid` | Check if custom broker config is valid |

Returns `valid` or `invalid`. Checks that server, port, and (if not WebSocket mode) username/password are all set.

### Timezone

| Command | Description |
|---------|-------------|
| `get timezone` | Current timezone string |
| `get timezone.offset` | UTC offset in hours |
| `set timezone <tz>` | Set timezone (IANA, abbreviation, or UTC offset) |
| `set timezone.offset <hours>` | Set UTC offset (-12 to +14) |

**Supported timezone formats:**
- IANA: `America/Chicago`, `Europe/London`, `Asia/Tokyo`
- Abbreviations: `CDT`, `CST`, `EST`, `PST`, `GMT`, `CET`
- UTC offsets: `UTC-6`, `UTC+1`, `+5`, `-8`

### Bridge Control

| Command | Description |
|---------|-------------|
| `get bridge.enabled` | Bridge master switch (on/off) |
| `get bridge.source` | Packet source: `logRx` or `logTx` |
| `set bridge.enabled on\|off` | Enable/disable the MQTT bridge |
| `set bridge.source rx\|tx` | Capture received or transmitted packets |

### General

| Command | Description |
|---------|-------------|
| `get name` | Device name |
| `set name <name>` | Set device name (supports UTF-8/emoji) |
| `reboot` | Reboot the device |

---

## Broker Architecture

The MQTT bridge supports up to 3 simultaneous broker connections:

| Slot | Broker | Protocol | Auth | Configurable via CLI |
|------|--------|----------|------|---------------------|
| 0 | Custom | TCP or WSS | User/pass or JWT | Yes (`mqtt.server`, etc.) |
| 1 | LetsMesh US | WSS | JWT (Ed25519) | Toggle only (`mqtt.analyzer.us`) |
| 2 | LetsMesh EU | WSS | JWT (Ed25519) | Toggle only (`mqtt.analyzer.eu`) |

All brokers receive the same messages. Each broker reconnects independently with exponential backoff.

### JWT Authentication

For WebSocket brokers (LetsMesh + custom when `mqtt.ws` is on):
- Device generates Ed25519-signed JWT tokens
- Username format: `v1_{UPPERCASE_PUBLIC_KEY}`
- Tokens auto-renew before expiration
- No manual credential setup needed

### Custom Broker with WebSocket + TLS

To connect to a WSS broker (like chicagooffline.com):
```
set mqtt.server wsmqtt.chicagooffline.com
set mqtt.port 443
set mqtt.tls on
set mqtt.ws on
reboot
```

No username/password needed — JWT auth handles it. Verify with:
```
get mqtt.config.valid
```

---

## MQTT Topics

All messages publish under:
```
meshcore/{IATA}/{DEVICE_PUBLIC_KEY}/{type}
```

| Type | Topic Suffix | Retained | Description |
|------|-------------|----------|-------------|
| Status | `/status` | Yes | Device online/offline, metadata |
| Packets | `/packets` | No | Decoded packet data with RF info |
| Raw | `/raw` | No | Minimal raw hex packet data |

Example: `meshcore/ORD/7E7662...9400/status`

---

## JSON Message Formats

### Status
```json
{
  "status": "online",
  "timestamp": "2026-04-24T12:00:00.000000",
  "origin": "🪷 Lotus 🍄",
  "origin_id": "7E7662676F7F0850...",
  "model": "heltec_wifi_lora_32_V3",
  "firmware_version": "1.8.6",
  "radio": "SX1262 @ 906.875 MHz",
  "client_version": "meshcore-custom-repeater/Apr 24 2026"
}
```

### Packet
```json
{
  "origin": "🪷 Lotus 🍄",
  "origin_id": "7E7662676F7F0850...",
  "timestamp": "2026-04-24T12:00:00.000000",
  "type": "PACKET",
  "direction": "rx",
  "len": "45",
  "packet_type": "4",
  "route": "F",
  "payload_len": "32",
  "raw": "F5930103807E5F1E...",
  "SNR": "12.5",
  "RSSI": "-65",
  "hash": "A1B2C3D4E5F67890",
  "path": "node1,node2,node3"
}
```

### Raw
```json
{
  "origin": "🪷 Lotus 🍄",
  "origin_id": "7E7662676F7F0850...",
  "timestamp": "2026-04-24T12:00:00.000000",
  "type": "RAW",
  "data": "F5930103807E5F1E..."
}
```

---

## Building from Source

### Heltec V3
```bash
pio run -e Heltec_v3_repeater_observer_mqtt
```

### Station G2
```bash
pio run -e Station_G2_repeater_observer_mqtt
```

### Compile-time Overrides

You can bake in defaults via `platformio.ini` build flags:

```ini
build_flags =
  -D WITH_MQTT_BRIDGE=1
  -D MAX_MQTT_BROKERS=3
  -D MQTT_MAX_PACKET_SIZE=1024
  -D MQTT_DEBUG=1              ; verbose MQTT logging to serial
  -D MESH_PACKET_LOGGING=1    ; log all mesh packets
  -D CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y  ; TLS root cert bundle
```

---

## Files

| File | Purpose |
|------|---------|
| `src/helpers/bridges/MQTTBridge.h/.cpp` | MQTT bridge — broker connections, reconnect, publishing |
| `src/helpers/MQTTMessageBuilder.h/.cpp` | JSON message formatting |
| `src/helpers/JWTHelper.h/.cpp` | Ed25519 JWT token generation for LetsMesh auth |
| `src/helpers/CommonCLI.cpp` | Serial/LoRa CLI command handlers |
| `examples/simple_repeater/MyMesh.h/.cpp` | Repeater integration hooks |

---

## Dependencies

| Library | Purpose |
|---------|---------|
| PsychicMqttClient | ESP-IDF native MQTT client (TCP + WSS) |
| ArduinoJson | JSON message formatting |
| NTPClient | Time synchronization |
| JChristensen/Timezone | DST-aware timezone conversion |
| paulstoffregen/Time | Time management |
