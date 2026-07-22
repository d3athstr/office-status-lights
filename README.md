# Office Status Lights

DIY **Zigbee** status lights for two home offices — XIAO ESP32-C6 + 24-LED
WS2812B ring, replacing off-the-shelf Zigbee night lights ("Network
Indicator" entities in Home Assistant).

## Behavior (rev 3)

The whole ring is **one color-controllable light in HA** (ZHA,
`COLOR_DIMMABLE_LIGHT` endpoint, entity `light.don_office_busy_light`).
HA automations drive it:

- **Green** = free, **red** = busy — state lives in
  `input_boolean.office_status_light_alert` (ON = red 70%, OFF = green 30%)
- A **Sonoff SNZB-01P button** ("Busy Light Button") on the desk toggles
  the boolean — single press flips red ↔ green
- Lit **24/7 whenever powered** (schedule bound removed 2026-07-21). On
  battery the ring is dark by hardware — ring VCC is USB-VBUS only on the
  as-built unit — so this is effectively "on when plugged in" with zero
  HA-side power detection needed
- Colors live in HA — firmware is just a Zigbee color light

> Deliberate exception to the house ISA-101 dark-when-OK rule: owner spec
> 2026-07-16; originally bounded by an office-hours schedule, now bounded
> only by USB power (owner spec 2026-07-21). (Rev 2's four-quadrant
> multi-state scheme is in git history if we ever want it back.)

## Design

| Piece | Choice | Why |
|---|---|---|
| MCU | Seeed XIAO ESP32-C6 | Native 802.15.4 radio (Zigbee 3.0), onboard 1S LiPo charge management, tiny |
| Radio | Zigbee 3.0 end device via [zigbee_esphome](https://github.com/luar123/zigbee_esphome) | Pairs with ZHA; component is experimental — pinned by full commit SHA (short SHAs aren't fetchable) |
| LEDs | 24-LED WS2812B ring (5050) | On hand; single color endpoint drives all 24 |
| Power | See wiring variants below | |
| Ring gating | AO3400 N-FET on the ring's GND return (optional), gate GPIO2 + 100 k pulldown | Dark ring = 0 mA instead of ~1 mA/LED; firmware follows the ring state, no-op if unfitted |
| Data | GPIO1 → 330 Ω → DIN, 1000 µF across the ring rails | Standard WS2812 hygiene |
| Enclosure | 3D-printed diffuser/stand | Design TBD |

## Wiring variants

Full details, conversion punch list and diagram: [`docs/wiring.md`](docs/wiring.md)

1. **As-built (unit 1, rev 1):** ring VCC on the XIAO **5V pin**, LiPo on
   the BAT pads. Fully functional on USB power; on battery the MCU + radio
   stay online but the **ring is dark** (the 5V pin is USB-VBUS only).
2. **Rev 4 target — diode-OR:** ring VCC fed through two Schottky diodes
   (SS34/1N5822) from the 5V pin *and* BAT+ — USB carries the ring when
   present (charger free to charge the cell), the LiPo carries it during
   outages. AO3400 + 100 k in the ring's GND return so a dark ring costs
   0 mA. (Rev 2's plain BAT+ feed is superseded: with the ring lit 24/7 it
   out-draws the ~100 mA onboard charger and the cell would discharge even
   on USB.)

![Wiring diagram](docs/wiring-diagram.svg)

## Firmware

- `firmware/status-light-common.yaml` — shared package: one
  `COLOR_DIMMABLE_LIGHT` Zigbee endpoint (ON_OFF/LEVEL/COLOR_CONTROL xy),
  RF-switch init, optional FET gate follows ring state, BOOT-button
  (5–20 s) Zigbee factory reset
- `firmware/status-light-{don,deanna}-office.yaml` — thin per-device configs

Requires **ESPHome ≥ 2026.6**. No `wifi:`/`api:`/`ota:` — reflash over
USB-C only. Build env: RAZORCREST, `C:\Users\donja\esphome-status\`.

### Hard-won gotchas (all cost real debugging time)

- **XIAO C6 RF switch:** GPIO3 low = switch power, GPIO14 low = internal
  antenna — neither is on the header, nothing sets them on esp-idf, and
  without them the radio hears *nothing* (steering loops on NO_NETWORK).
- **Never attach a serial monitor while testing Zigbee** — opening the
  C6's USB-Serial/JTAG port hard-resets the chip (`rst:0x15`) and drops it
  off the network. It also fakes a perfect "crash loop" if your reader
  auto-reconnects.
- **Stale flash** makes the stack boot "non-factory-new" and loop on
  `BDB Device Reboot failed (0x03)` without ever scanning: fix with
  `esptool erase-region 0x370000 0x70000` (nvs) + `0x3E0000 0x1000`
  (zb_fct). Zigbee NVRAM lives in the shared `nvs` partition.
- **Joining:** after 10 fast retries the component steers only once per
  10 min — open `zha.permit` (254 s) and reset the device into a fresh
  retry cycle. Changing the endpoint layout requires removing the device
  from ZHA and re-pairing (interview is cached).

## Battery reality

Always-on end device baseline ≈ 25 mA; ring green @ 30 % 24/7 adds
~100–130 mA. On the rev 4 rewire that means **~2–2.5 days per charge** if
the ring stays lit on battery, or **~10–13 days** with a dark-when-OK
policy (needs the phase-2 USB-present detect). Onboard charging is ~100 mA
(3+ days for a full 8 Ah after a deep drain) — fine as continuous top-off,
slow after an outage; charge externally or add a TP4056 if in a hurry.
Future: sleepy-poll firmware (sub-mA baseline) needs bench proof that
parent-queued ZHA commands survive the poll window.

## Open items

- [x] HA schedule/status automations (schedule + busy-button toggle, 2026-07-21)
- [ ] **Order Schottky diodes** (SS34 or 1N5822, ~20) — not in parts stock
- [ ] Unit 1: meter the LiPo + BAT joints (board has died on every USB
      loss: 2026-07-16 21:12 and overnight 2026-07-21 — battery path has
      never actually carried it)
- [ ] Unit 1: replace the USB cable (2026-07-21 outage was a full USB
      de-enumeration; cable was flapping the week before)
- [ ] Unit 1: rev 4 rewire (diode-OR + AO3400) per docs/wiring.md punch list
- [ ] Phase 2: USB-present detect (D3) + battery % (A0 divider or MAX17048)
      → dark-when-OK on battery
- [ ] Second 8000 mAh LiPo + build for DeAnna's office (needs 2 more Schottkys)
- [ ] Diffuser/stand design (3D print)
- [ ] Retire or redeploy the old Zigbee night lights
