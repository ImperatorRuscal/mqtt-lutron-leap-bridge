# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm start                             # Run the application
docker build -t mqtt-lutron-leap-bridge .  # Build Docker image
```

There are no test or lint scripts defined in `package.json`.

## Architecture

This is an ES6 module (`"type": "module"`) Node.js bridge that translates between the Lutron LEAP protocol and MQTT.

### Two core files

**`lutron.js`** — `LutronLeap` class wrapping `@terafin2/lutron-leap`. Handles TLS certificate-based connection to the Lutron bridge, discovers the device hierarchy (areas, control stations, devices, LEDs), and emits typed events: `zone-status`, `area-status`, `led-status`, `button-status`, `shade-status`, `device-status`, `scene-status`, `sensor-status`. Exposes command methods: `sendZoneLevelCommand()`, `sendZoneOnOffCommand()`, `sendButtonPress()`.

**`mqtt-lutron-leap-bridge.js`** — Main entry point. Validates required env vars, instantiates both connections, and wires up bidirectional message translation between MQTT topics and Lutron events.

### Data flow

```
Lutron Bridge (TLS/LEAP) ↔ lutron.js (EventEmitter) ↔ mqtt-lutron-leap-bridge.js ↔ MQTT broker
```

A 10-second interval pings the Lutron bridge for health; the process exits on disconnection (relies on container restart).

## Configuration

All configuration is via environment variables:

| Variable | Description |
|---|---|
| `BRIDGE_IP` | Lutron LEAP bridge IP address |
| `BRIDGE_CA_PATH` | Path to CA certificate file |
| `BRIDGE_CERT_PATH` | Path to client certificate file |
| `BRIDGE_KEY_PATH` | Path to client private key file |
| `MQTT_HOST` | MQTT broker hostname/IP |
| `TOPIC_PREFIX` | MQTT topic root (defaults to `/leap/`) |

## MQTT Topic Structure

**Published (Lutron → MQTT):**
- `{TOPIC_PREFIX}/zone/{ID}/on_off` — 0 or 1
- `{TOPIC_PREFIX}/zone/{ID}/level` — brightness level
- `{TOPIC_PREFIX}/areas/{ID}/occupancy` — 0 or 1
- `{TOPIC_PREFIX}/led/{ID}` — 0 or 1
- `development/{TOPIC_PREFIX}/button|shade|device|scene|sensor/{ID}` — experimental, format TBD

**Subscribed (MQTT → Lutron):**
- `{TOPIC_PREFIX}/zone/{ID}/on_off/set`
- `{TOPIC_PREFIX}/zone/{ID}/level/set`
- `{TOPIC_PREFIX}/button/{ID}/press`

## Key Dependencies

- `@terafin2/lutron-leap` — low-level LEAP protocol client
- `mqtt` — MQTT client
- `homeautomation-js-lib` — logging, MQTT helpers, health check utilities
- `lodash`, `interval-promise`
