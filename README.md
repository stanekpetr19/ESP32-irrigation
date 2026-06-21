# Smart Irrigation ESPHome

ESPHome-based ESP32 irrigation/relay controller for Home Assistant.

## Current status

Initial board bring-up firmware:

- Wi-Fi
- Home Assistant native API
- OTA updates
- local ESPHome web server
- 8 relay outputs
- safe boot with relays off
- GPIO23 rescue jumper input
- ESPHome safe mode enabled

## Hardware

Relay GPIOs:

| Relay | GPIO |
|---|---:|
| Relay 1 | GPIO32 |
| Relay 2 | GPIO33 |
| Relay 3 | GPIO25 |
| Relay 4 | GPIO26 |
| Relay 5 | GPIO27 |
| Relay 6 | GPIO14 |
| Relay 7 | GPIO12 |
| Relay 8 | GPIO13 |

Rescue jumper:

| Function | GPIO |
|---|---:|
| Rescue jumper to GND | GPIO23 |

## Development

Install dependencies:

```bash
uv sync