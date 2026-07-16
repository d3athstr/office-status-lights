# Office Status Lights

DIY **Zigbee** status lights for two home offices — battery-powered XIAO
ESP32-C6 + 24-LED WS2812B ring, replacing off-the-shelf Zigbee night lights
("Network Indicator" entities in Home Assistant) that could only show one
color at a time.

## Design

| Piece | Choice | Why |
|---|---|---|
| MCU | Seeed XIAO ESP32-C6 | Native 802.15.4 radio (Zigbee 3.0), onboard 1S LiPo charge management, tiny |
| Radio | Zigbee 3.0 end device via [zigbee_esphome](https://github.com/luar123/zigbee_esphome) | Battery operation ruled out WiFi (~80–100 mA); pairs with ZHA. Component is experimental — pinned by commit |
| LEDs | 24-LED WS2812B ring (5050) | Addressable — four quadrants show four states at once |
| Power | 1S LiPo 8000 mAh on the BAT pads | Protected cell; ring VCC fed from BAT+ (the 5V pin is dead on battery) |
| Ring gating | AO3400 N-FET on the ring's GND return, gate GPIO2 + 100 k pulldown | WS2812Bs idle at ~1 mA/LED — gating makes the all-OK dark state cost 0 mA |
| Data | GPIO1 → 330 Ω → DIN, 1000 µF across the ring rails | Standard WS2812 hygiene |
| Enclosure | 3D-printed diffuser/stand | Design TBD |

Full wiring: [`docs/wiring.md`](docs/wiring.md)

![Wiring diagram](docs/wiring-diagram.svg)

## ISA-101 / HP-HMI rule

The light is **dark when everything is OK** — color only on exception. No
always-on green. Each quadrant appears in HA as its own fixed-hue dimmable
light; automations only turn segments on/off and set brightness. Hues live
in firmware:

| Segment | LEDs | HA entity (endpoint) | State | Exception color |
|---|---|---|---|---|
| `wan` | 0–5 | endpoint 1 | WAN / internet | red |
| `vpn` | 6–11 | endpoint 2 | VPN tunnels | amber |
| `backup` | 12–17 | endpoint 3 | Backup freshness | amber |
| `alert` | 18–23 | endpoint 4 | Unacked alerts | magenta |

## Firmware

- `firmware/status-light-common.yaml` — shared package: Zigbee endpoints
  (4× `DIMMABLE_LIGHT` with `ON_OFF`/`LEVEL` clusters), ring + partitions,
  FET gate logic, BOOT-button (5–20 s) Zigbee factory reset
- `firmware/status-light-{don,deanna}-office.yaml` — thin per-device configs

Requires **ESPHome ≥ 2026.6** (the zigbee component's floor). No `wifi:`,
`api:`, or `ota:` — reflash over USB-C only, so get it right on the bench.
Pairing: open ZHA "Add device"; an unjoined light joins automatically.

## Battery reality (v1)

Always-on end device baseline is ~25 mA → **~10–14 days per charge** at
8000 mAh. Onboard charging is ~100 mA (3+ days for a full 8 Ah) — top off
overnight, charge externally, or add a TP4056. **v2 plan:** sleepy end
device with a short poll interval (sub-mA baseline, months per charge) —
needs bench validation that ZHA commands queued at the parent survive the
poll window.

## Open questions

- [ ] v2 sleepy-poll firmware (see above)
- [ ] Battery % to HA — needs a 200k/100k divider to A0 (commented block in firmware)
- [ ] Diffuser/stand design (3D print)
- [ ] Final state list + priority order; HA automations driving the segments
- [ ] Retire or redeploy the old Zigbee night lights
