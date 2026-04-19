# WitMotion WT901BLECL — Browser Dashboard

## Overview

A single-page web application that connects to a **WitMotion WT901BLECL** IMU sensor via **Web Bluetooth API** and displays live motion data in real time. No backend, no install, no drivers — runs entirely in the browser.

---

## Target Stack

- **Plain HTML + CSS + JavaScript** (single file, no framework)
- **Web Bluetooth API** (native browser API, no libraries needed)
- Chrome or Edge only (Web Bluetooth is not supported in Safari or Firefox)

---

## Sensor Background

The WT901BLECL is a 9-axis IMU (accelerometer + gyroscope + magnetometer) that communicates over Bluetooth Low Energy (BLE). It uses WitMotion's proprietary binary protocol over GATT.

### BLE GATT Profile

| Role            | UUID                                   |
|-----------------|----------------------------------------|
| Service         | `0000ffe5-0000-1000-8000-00805f9a34fb` |
| Notify Char.    | `0000ffe4-0000-1000-8000-00805f9a34fb` |
| Write Char.     | `0000ffe9-0000-1000-8000-00805f9a34fb` |

- Subscribe to the **notify characteristic** to receive sensor data
- Write to the **write characteristic** to send configuration commands

### Data Packet Format

Each packet is **11 bytes**:

```
[0x55] [TYPE] [D1_L] [D1_H] [D2_L] [D2_H] [D3_L] [D3_H] [D4_L] [D4_H] [SUM]
```

| Byte | Description                        |
|------|------------------------------------|
| 0    | Start marker — always `0x55`       |
| 1    | Packet type (see table below)      |
| 2–3  | Value 1 (little-endian int16)      |
| 4–5  | Value 2 (little-endian int16)      |
| 6–7  | Value 3 (little-endian int16)      |
| 8–9  | Value 4 (little-endian int16)      |
| 10   | Checksum (sum of bytes 0–9 & 0xFF) |

### Packet Types

| Type Byte | Data             | Scale Factor     | Unit  |
|-----------|------------------|------------------|-------|
| `0x51`    | Acceleration X/Y/Z + temp | / 32768 × 16 | g     |
| `0x52`    | Angular velocity X/Y/Z + temp | / 32768 × 2000 | °/s   |
| `0x53`    | Angle Roll/Pitch/Yaw + temp | / 32768 × 180 | °     |
| `0x54`    | Magnetic field X/Y/Z + temp | / 32768 × 1 | μT    |

### Parsing Formula (int16 from two bytes)

```javascript
function toInt16(low, high) {
  const val = (high << 8) | low;
  return val > 32767 ? val - 65536 : val;
}
```

### Incoming Buffer Handling

The BLE notify characteristic may deliver **partial packets** or **multiple packets** in a single callback. You must maintain a running buffer and extract complete 11-byte packets from it, not assume each callback equals one packet.

---

## Required Features

### 1. Connection Panel
- A single **"Connect Sensor"** button (must be triggered by user gesture — Web Bluetooth requirement)
- Status indicator: Disconnected / Connecting / Connected
- Device name display once connected
- **"Disconnect"** button when connected

### 2. Live Data Display
Show the following values updating in real time:

**Angles**
- Roll (°)
- Pitch (°)
- Yaw (°)

**Acceleration**
- X, Y, Z (g)

**Angular Velocity**
- X, Y, Z (°/s)

### 3. 3D Orientation Visualiser
- A simple 3D box/cube rendered using **Three.js** (load from CDN)
- The box rotates in real time to reflect the sensor's roll, pitch, yaw
- This is the centrepiece of the UI

### 4. Time-Series Chart
- A live scrolling line chart using **Chart.js** (load from CDN)
- Plots roll, pitch, and yaw over the last ~5 seconds
- Max 100 data points shown at a time (older points drop off the left)

### 5. Data Log
- A scrollable text area showing the last 50 raw parsed readings
- Format: `[HH:MM:SS.mmm] Roll: X° Pitch: Y° Yaw: Z°`
- Auto-scrolls to latest entry

---

## UI Design

- **Dark theme** — background `#0f1117`, cards `#1a1d27`
- Clean, minimal layout — sensor dashboard aesthetic
- Layout:
  - Top: connection bar (full width)
  - Middle row: 3D visualiser (left, larger) + live value cards (right)
  - Bottom row: chart (left) + data log (right)
- Responsive enough to work at 1280px+ width
- Font: system-ui or Inter from Google Fonts

---

## Error Handling

- If the browser doesn't support Web Bluetooth, show a clear message: _"Web Bluetooth is not supported in this browser. Please use Chrome or Edge."_
- If the device disconnects unexpectedly, update the status indicator and show a reconnect button
- If a packet fails checksum validation, silently skip it (do not crash)

---

## File Structure

Single file output:

```
index.html   ← everything: HTML structure, CSS styles, JS logic
```

No build step, no npm, no bundler. Just open in Chrome.

---

## CDN Dependencies

Load these from CDN (do not bundle locally):

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
```

---

## Implementation Notes for Claude Code

1. **Buffer management is critical** — incoming BLE data may be chunked. Maintain a `Uint8Array` buffer across callbacks and extract complete 11-byte packets starting with `0x55`.

2. **Three.js rotation order** — use `THREE.Euler` with order `'ZYX'` and convert degrees to radians. Apply roll → X axis, pitch → Y axis, yaw → Z axis.

3. **Chart.js real-time updates** — use `chart.data.labels.push()` and `chart.data.datasets[x].data.push()` then call `chart.update('none')` for smooth updates without animation lag.

4. **Web Bluetooth connection flow:**
```javascript
const device = await navigator.bluetooth.requestDevice({
  filters: [{ namePrefix: 'WT901' }],
  optionalServices: ['0000ffe5-0000-1000-8000-00805f9a34fb']
});
const server = await device.gatt.connect();
const service = await server.getPrimaryService('0000ffe5-0000-1000-8000-00805f9a34fb');
const characteristic = await service.getCharacteristic('0000ffe4-0000-1000-8000-00805f9a34fb');
await characteristic.startNotifications();
characteristic.addEventListener('characteristicvaluechanged', handleData);
```

5. **Handle disconnection events:**
```javascript
device.addEventListener('gattserverdisconnected', onDisconnected);
```

6. **Checksum validation:**
```javascript
function validateChecksum(packet) {
  const sum = packet.slice(0, 10).reduce((a, b) => a + b, 0);
  return (sum & 0xFF) === packet[10];
}
```

---

## Out of Scope (for this version)

- Sending configuration commands back to the sensor
- Recording / exporting data to CSV
- Mobile layout
- Multiple simultaneous sensors
