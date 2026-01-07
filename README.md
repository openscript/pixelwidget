# PixelWidget - MQTT Pixel Control for Ulanzi TC001

Control individual pixels on a Ulanzi TC001 (32x8 LED matrix) via MQTT (with TLS).

## Hardware

- **Device**: Ulanzi TC001 (ESP32 + 32x8 WS2812 LED matrix)
- **Matrix**: 256 LEDs in zigzag arrangement
- **Coordinates**: X: 0-31 (left to right), Y: 0-7 (top to bottom)

## Setup

1. Fill in your WiFi and MQTT credentials in `secrets.yaml`
2. Flash with: `esphome run pixelwidget.yaml`

## MQTT Topics

### Control Commands

| Topic | Payload Format | Description |
|-------|----------------|-------------|
| `pixelwidget/pixel/set` | `x,y,r,g,b` | Set single pixel color |
| `pixelwidget/pixel/fill` | `r,g,b` | Fill entire display with color |
| `pixelwidget/pixel/clear` | (any) | Clear display (all black) |
| `pixelwidget/pixel/line` | `x1,y1,x2,y2,r,g,b` | Draw a line |
| `pixelwidget/pixel/rect` | `x,y,w,h,r,g,b` | Draw rectangle outline |
| `pixelwidget/pixel/fillrect` | `x,y,w,h,r,g,b` | Draw filled rectangle |
| `pixelwidget/pixel/image` | Base64 RGB data | Set entire display (see below) |
| `pixelwidget/brightness` | `0-255` | Set display brightness |

### Status Topics

| Topic | Description |
|-------|-------------|
| `pixelwidget/status` | Online/offline status (birth/will) |
| `pixelwidget/button` | Button press events: `left`, `middle`, `right` |

### Image Request Topic

| Topic | Payload | Trigger | Description |
|-------|---------|---------|-------------|
| `pixelwidget/image/request` | `1` | Automatic | Sent on MQTT connection (after 1s delay) and on middle button press to request the current display image from the backend |

## Image Synchronization

The device automatically requests the current display image in two scenarios:

1. **On MQTT connection** - After successfully connecting to the broker, the device sends an image request after a 1-second delay to ensure stability
2. **On middle button press** - When the middle button is pressed, it wakes the display and requests the current image

Your backend service should:
- Subscribe to `pixelwidget/image/request`
- When received, fetch or generate the current display image as a 32x8 RGB image (see [Sending a Full Image](#sending-a-full-image) section)
- Publish the base64-encoded image to `pixelwidget/pixel/image`

This enables the display to stay synchronized with your application state across reboots and wake events.

## Examples

### Using mosquitto_pub (CLI)

```bash
# Set pixel at (0,0) to red
mosquitto_pub -h <broker> -p 8883 -u <user> -P <pass> --cafile ca.crt \
  -t "pixelwidget/pixel/set" -m "0,0,255,0,0"

# Fill display with blue
mosquitto_pub -h <broker> -p 8883 -u <user> -P <pass> --cafile ca.crt \
  -t "pixelwidget/pixel/fill" -m "0,0,255"

# Clear display
mosquitto_pub -h <broker> -p 8883 -u <user> -P <pass> --cafile ca.crt \
  -t "pixelwidget/pixel/clear" -m ""

# Draw red line from (0,0) to (31,7)
mosquitto_pub -h <broker> -p 8883 -u <user> -P <pass> --cafile ca.crt \
  -t "pixelwidget/pixel/line" -m "0,0,31,7,255,0,0"

# Draw green rectangle at (5,2) with size 10x4
mosquitto_pub -h <broker> -p 8883 -u <user> -P <pass> --cafile ca.crt \
  -t "pixelwidget/pixel/rect" -m "5,2,10,4,0,255,0"

# Set brightness to 50%
mosquitto_pub -h <broker> -p 8883 -u <user> -P <pass> --cafile ca.crt \
  -t "pixelwidget/brightness" -m "128"
```

### Sending a Full Image

The `pixelwidget/pixel/image` topic accepts a base64-encoded RGB image. The image must be exactly 32x8 pixels (256 pixels × 3 bytes = 768 bytes of RGB data).

**Image format:**
- Raw RGB bytes, row by row (top to bottom, left to right)
- Each pixel is 3 bytes: R, G, B (0-255 each)
- Total: 768 bytes → Base64 encoded = 1024 characters

**Python example:**

```python
import base64
import ssl
import paho.mqtt.client as mqtt

# Create a 32x8 RGB image (red gradient)
rgb_data = bytearray()
for y in range(8):
    for x in range(32):
        rgb_data.extend([x * 8, 0, 0])  # Red gradient

# Encode to base64
image_b64 = base64.b64encode(rgb_data).decode('ascii')

# Send via MQTT
client = mqtt.Client()
client.username_pw_set("display", "your_password")
client.tls_set(cert_reqs=ssl.CERT_REQUIRED, tls_version=ssl.PROTOCOL_TLS)
client.connect("your-cluster.s1.eu.hivemq.cloud", 8883, 60)
client.publish("pixelwidget/pixel/image", image_b64)
client.disconnect()
```

**Node.js example:**

```javascript
const mqtt = require('mqtt');

// Create a 32x8 RGB image (blue gradient)
const rgbData = Buffer.alloc(768);
for (let y = 0; y < 8; y++) {
  for (let x = 0; x < 32; x++) {
    const idx = (y * 32 + x) * 3;
    rgbData[idx] = 0;           // R
    rgbData[idx + 1] = 0;       // G
    rgbData[idx + 2] = x * 8;   // B gradient
  }
}

const client = mqtt.connect('mqtts://your-cluster.s1.eu.hivemq.cloud:8883', {
  username: 'display',
  password: 'your_password'
});

client.on('connect', () => {
  client.publish('pixelwidget/pixel/image', rgbData.toString('base64'));
  client.end();
});
```

### Using Python (paho-mqtt)

```python
import ssl
import paho.mqtt.client as mqtt

# Connect to HiveMQ Cloud
client = mqtt.Client()
client.username_pw_set("display", "your_password")
client.tls_set(cert_reqs=ssl.CERT_REQUIRED, tls_version=ssl.PROTOCOL_TLS)
client.connect("your-cluster.s1.eu.hivemq.cloud", 8883, 60)

# Set pixel at (10, 3) to green
client.publish("pixelwidget/pixel/set", "10,3,0,255,0")

# Draw a red diagonal line
client.publish("pixelwidget/pixel/line", "0,0,31,7,255,0,0")

# Fill with purple
client.publish("pixelwidget/pixel/fill", "128,0,128")

client.disconnect()
```

### Using Node.js

```javascript
const mqtt = require('mqtt');

const client = mqtt.connect('mqtts://your-cluster.s1.eu.hivemq.cloud:8883', {
  username: 'display',
  password: 'your_password'
});

client.on('connect', () => {
  // Set multiple pixels for a pattern
  for (let x = 0; x < 32; x++) {
    const color = x % 2 === 0 ? '255,0,0' : '0,255,0';
    client.publish('pixelwidget/pixel/set', `${x},0,${color}`);
  }
});
```

## Coordinate System

```
      X →
    0 1 2 3 ... 31
  ┌─────────────────┐
0 │ ● ● ● ● ... ● │  Y
1 │ ● ● ● ● ... ● │  ↓
2 │ ● ● ● ● ... ● │
3 │ ● ● ● ● ... ● │
4 │ ● ● ● ● ... ● │
5 │ ● ● ● ● ... ● │
6 │ ● ● ● ● ... ● │
7 │ ● ● ● ● ... ● │
  └─────────────────┘
```

- Origin (0,0) is top-left
- X increases to the right (0-31)
- Y increases downward (0-7)
- Colors are RGB values 0-255

## Files

- `pixelwidget.yaml` - Main ESPHome configuration with MQTT pixel control
- `secrets.yaml` - WiFi and MQTT credentials (do not commit)

## License

MIT
