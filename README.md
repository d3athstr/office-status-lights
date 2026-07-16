# Office Status Lights

DIY ESPHome RGB status lights for Don's and DeAnna's offices — replacing the
off-the-shelf **Third Reality 3RSNL02043Z Zigbee night lights** ("Network
Indicator" / "Network Indicator 2" in HA) currently driven by office-occupancy
automations to show network/internet state.

PartsBin project: https://parts.empire12.net (Office Status Lights, project 3)

## Why replace

- Native ESPHome/HA API instead of Zigbee — richer control (segments,
  animations, brightness curves) and one less Zigbee pairing surface
- A 24-LED addressable ring can show **several states at once** (WAN, VPN,
  backups, alerts) instead of one color

## Design

| Piece | Choice | Why |
|---|---|---|
| MCU | Seeed XIAO ESP32-C6 (on hand ×9) | Compact for in-diffuser mounting, WiFi, ESPHome-supported (ESP-IDF) |
| LEDs | 24-LED WS2812B ring, 5050 (on hand ×10) | Addressable — partition into per-state segments |
| Data conditioning | 330 Ω series resistor on DIN | Standard WS2812 hygiene, damps reflections |
| Supply hygiene | 1000 µF electrolytic across 5 V at the ring | Absorbs inrush / switching transients |
| Power | 5 V ≥2 A USB wall wart (not yet in inventory) | Single supply feeds XIAO + ring |
| Enclosure | 3D-printed diffuser/stand | Design TBD |

## ISA-101 / HP-HMI rule (standing)

The light is **dim/neutral when everything is OK** — color and brightness only
on exception (WAN down, VPN down, backup stale, unacked alert). **No always-on
green.** State logic lives in HA automations; the firmware just exposes the
ring and four 6-LED segments.

Proposed segment map (quadrants, clockwise from top):

| Segment | LEDs | State | Exception color |
|---|---|---|---|
| `wan` | 0–5 | WAN / internet | red |
| `vpn` | 6–11 | VPN tunnels | amber |
| `backup` | 12–17 | Backup freshness | amber |
| `alert` | 18–23 | Unacked Wazuh/HA alerts | magenta |

All-OK idle: everything off, or a barely-visible warm white (≤5 %) if a
"powered and healthy" heartbeat is wanted.

## Firmware

- `firmware/status-light-common.yaml` — shared package (ring + 4 partitions,
  WiFi, API, OTA, software power limit)
- `firmware/status-light-don-office.yaml` / `status-light-deanna-office.yaml`
  — thin per-device configs (name + data pin only)

Build env on RAZORCREST (see memory gotchas); OTA after first USB flash; WiFi
creds via `!secret`. Note: ESPHome sensor publishes log at *debug* level.

## Wiring

See [`docs/wiring.md`](docs/wiring.md) and the diagram:

![Wiring diagram](docs/wiring-diagram.svg)

## Open questions

- [ ] Final state list and priority order when several are active
- [ ] Ring vs. short strip segment-mapped to fixed states — **ring assumed**
- [ ] Diffuser/stand design (3D print)
- [ ] Retire the Third Reality units or redeploy them elsewhere?

> Historical note: the Network Project HA doc called the current indicators
> "ESPHome-based" — wrong; they are Zigbee night lights. Doc corrected
> 2026-07-13. Two more of the same Zigbee units run as Garage Door Indicators —
> NOT in scope here.
