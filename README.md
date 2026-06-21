# ESP32 Smart Irrigation Controller

ESPHome-based ESP32 controller for managing irrigation and tank-refill valve routing in Home Assistant.

This project is a practical hobby embedded/home-automation build. The goal is to create a reliable, understandable, and shareable irrigation controller using ESPHome, Home Assistant, OTA updates, relay interlocks, and safe valve-state handling.

Current firmware version: **v0.2.0**

---

## Project goals

- Control irrigation-related valves using an ESP32 relay board.
- Integrate cleanly with Home Assistant using ESPHome native API.
- Support OTA firmware updates after the first serial flash.
- Expose only safe high-level modes to Home Assistant, not raw relay controls.
- Prevent mutually exclusive valve relays from being energized at the same time.
- Keep the design simple enough to maintain, but structured enough to be a useful embedded-learning project.

---

## Current system overview

The system uses an ESP32 relay board to control two motorized valves.

A separate Zigbee relay controls the 230 V pump. The pump is intentionally kept outside the ESP32 relay board so that mains voltage remains isolated from the ESP32/12 V valve-control side.

```text
Home Assistant
   |
   | ESPHome native API
   v
ESP32 relay controller
   |
   +-- Relay 3: Valve 1 -> tank source
   +-- Relay 4: Valve 1 -> well source
   +-- Relay 5: Valve 2 -> irrigation outlet
   +-- Relay 6: Valve 2 -> tank-fill outlet
   |
   +-- GPIO23 rescue jumper
   +-- Wi-Fi / OTA / local web UI
   |
   v
Home Assistant readiness sensors
   |
   v
Separate Zigbee relay controls pump
```

---

## Hydraulic concept

The system supports a well-pump-tank-pump-irrigation workflow.

The well has slow and inconsistent flow, which is not ideal for direct irrigation. The tank acts as a buffer. The pump can either draw from the tank for irrigation or draw from the well for refilling the tank.

### Valve 1: pump inlet/source

| Relay | Function | Result |
|---:|---|---|
| Relay 3 | Valve 1 tank direction | Pump draws preferentially from water tank |
| Relay 4 | Valve 1 well direction | Pump draws from well |

### Valve 2: pump outlet/destination

| Relay | Function | Result |
|---:|---|---|
| Relay 5 | Valve 2 irrigation direction | Pump outlet goes to irrigation system |
| Relay 6 | Valve 2 tank-fill direction | Pump outlet goes to tank-refill hose |

The valve relays are used as **momentary drive outputs**. They are energized for approximately 15 seconds, then turned off.

---

## Supported modes in v0.2.0

| Mode | Valve 1 source | Valve 2 outlet | Relay routine | Pump |
|---|---|---|---|---|
| OFF / standby | Tank | Tank-fill | Relay 3 + Relay 6 | Off |
| Irrigation | Tank | Irrigation | Relay 3 + Relay 5 | Allowed after ready |
| Tank refill | Well | Tank-fill | Relay 4 + Relay 6 | Allowed after ready |
| Well to irrigation / pool-fill | Well | Irrigation | Relay 4 + Relay 5 | Allowed after ready |

Only these high-level modes should be exposed to Home Assistant.

Raw relay controls for relays 3–6 are intentionally hidden from normal Home Assistant use.

---

## Safety rules

The firmware is designed around these rules:

1. Relays 3 and 4 must never be energized at the same time.
2. Relays 5 and 6 must never be energized at the same time.
3. Raw valve relays are not exposed as ordinary Home Assistant switches.
4. Home Assistant should only start the pump after the ESP32 reports that the requested valve mode is ready.
5. After pump stop, Home Assistant should request the ESP32 to return the valves to OFF / standby mode.
6. On ESP32 boot, valve state is treated as unknown until a fresh prepare routine is run.

---

## GPIO mapping

### Relay outputs

| Relay | GPIO | v0.2.0 role | Status |
|---:|---:|---|---|
| Relay 1 | GPIO32 | Reserved for v0.4 | Internal/off |
| Relay 2 | GPIO33 | Reserved for v0.4 | Internal/off |
| Relay 3 | GPIO25 | Valve 1 -> tank source | Active |
| Relay 4 | GPIO26 | Valve 1 -> well source | Active |
| Relay 5 | GPIO27 | Valve 2 -> irrigation outlet | Active |
| Relay 6 | GPIO12 | Valve 2 -> tank-fill outlet | Active |
| Relay 7 | GPIO13 | Reserved | Not used |
| Relay 8 | GPIO14 | Reserved | Not used |

### Inputs

| GPIO | Function |
|---:|---|
| GPIO0 | ESP32 boot/programming button |
| EN | ESP32 reset button |

GPIO12 is an ESP32 strapping pin. It works on this relay board but should be treated cautiously during boot testing.

---

## Home Assistant entities

Exact entity IDs may differ depending on Home Assistant naming.

### Buttons

| Button | Purpose |
|---|---|
| Prepare OFF Mode | Set valves to tank source + tank-fill outlet |
| Prepare Irrigation Mode | Set valves to tank source + irrigation outlet |
| Prepare Tank Refill Mode | Set valves to well source + tank-fill outlet |
| Prepare Well To Irrigation Mode | Set valves to well source + irrigation outlet |
| Force Valve Relays Off | Diagnostic/manual relay-off command |
| Restart | Restart ESP32 |
| Restart in Safe Mode | Boot ESPHome safe mode |

### Binary sensors

| Binary sensor | Meaning |
|---|---|
| Irrigation Ready | Valves are set for tank -> irrigation |
| Tank Refill Ready | Valves are set for well -> tank-fill |
| Off Mode Ready | Valves are set for standby/off |
| Well To Irrigation Ready | Valves are set for well -> irrigation |
| Valves Changing | Valve movement routine is currently running |

### Text sensors

| Text sensor | Meaning |
|---|---|
| Valve Mode | Current assumed valve mode |
| Firmware Version | Firmware version string |
| ESPHome Version | ESPHome build/runtime version |
| IP Address | Current device IP |
| Connected SSID | Connected Wi-Fi network |

---

## Recommended Home Assistant control flow

The pump should not be switched directly from a normal dashboard button.

Instead, use Home Assistant scripts.

### Irrigation start

```yaml
alias: Irrigation - Start
mode: single

sequence:
  - service: button.press
    target:
      entity_id: button.irrigation_controller_prepare_irrigation_mode

  - wait_template: >
      {{ is_state('binary_sensor.irrigation_controller_irrigation_ready', 'on') }}
    timeout: "00:00:45"
    continue_on_timeout: false

  - service: switch.turn_on
    target:
      entity_id: switch.your_zigbee_pump_relay
```

Replace `switch.your_zigbee_pump_relay` with the actual pump relay entity ID.

---

## Development setup

This project uses:

- ESPHome
- ESP32 with ESP-IDF framework backend
- uv for Python dependency management
- VS Code / VS Code Remote WSL
- Git + GitHub

### Install dependencies

```bash
uv sync
```

### Validate ESPHome config

```bash
uv run esphome config esphome/irrigation-relay-test.yaml
```

### Compile

```bash
uv run esphome compile esphome/irrigation-relay-test.yaml
```

### Upload by serial

Example using WSL USB passthrough:

```bash
uv run esphome upload esphome/irrigation-relay-test.yaml --device /dev/ttyUSB0
```

### Upload by OTA

```bash
uv run esphome upload esphome/irrigation-relay-test.yaml --device irrigation-relay-test.local
```

or by IP:

```bash
uv run esphome upload esphome/irrigation-relay-test.yaml --device 192.168.x.y
```

### View logs

```bash
uv run esphome logs esphome/irrigation-relay-test.yaml --device irrigation-relay-test.local
```

---

## Recovery strategy

### ESPHome safe mode

ESPHome safe mode is enabled. If the device repeatedly fails to boot, ESPHome can start with minimal functionality so OTA recovery remains possible.

## Version roadmap

### v0.1.0

- First serial flash
- Wi-Fi
- OTA
- ESPHome API
- local web server
- relay test controls

### v0.2.0

- Internal valve relay handling
- Relay 3/4 and 5/6 interlocks
- High-level valve modes
- Readiness sensors for Home Assistant
- Button-based Home Assistant workflow
- Pump gating through Home Assistant scripts

### v0.3.0 planned

- Add tank-level sensor
- Prevent refill when tank is full
- Prevent irrigation when tank is empty
- Add refill timeout protection
- Add basic fault states

### v0.4.0 planned

- Implement relays 1–2 for small inside 12V pumps for winter-garden plant irrigation from the tank
- Add zone-level irrigation modes
- Add per-zone safety checks
- Add automatic irrigation preparation logic

### v0.5.0 planned

- Implement soil moisture sensor outside
- Implement soil moisture sensor inside
- Implement temperature and weather information

### Future ideas

- Custom ESPHome external C++ component
- Valve-position feedback from limit switches
- Flow sensor
- Zigbee pump switch HW control via ESP32. 
- Dashboard cards for Home Assistant
- Hardware interlock for pump enable
- PCB or DIN-rail enclosure design

---

## Disclaimer

This is a hobby automation project controlling pumps and valves. It should not be considered a certified safety system.

Do not expose the ESPHome web interface to the public internet. Keep mains-voltage switching electrically isolated and follow local electrical safety regulations.
