# Wiring — Office Status Light (Zigbee)

One unit = XIAO ESP32-C6 + 24-LED WS2812B ring + AO3400 low-side FET, with a
1S LiPo (8000 mAh) on the XIAO's BAT pads and the ring fed through a
**Schottky diode-OR** (USB when present, battery otherwise). The radio is
Zigbee 3.0 (ZHA); there is no WiFi and no OTA — reflash over USB-C.

## Design history

- **Rev 1 / as-built (unit 1, 2026-07-16):** ring VCC on the XIAO **5V pin**
  (USB-VBUS passthrough), ring GND straight to GND (no FET), LiPo on
  BAT+/−. Works on USB; on battery the MCU + radio stay up but the **ring
  is dark** — the 5V pin is dead without USB.
- **Rev 2 (paper only):** ring VCC moved to BAT+, AO3400 in the GND return.
  **Superseded before build** — with the ring now lit 24/7 (owner spec
  2026-07-21) it has a fatal flaw: the ring's ~100–130 mA continuous draw
  would come from the battery *even on USB*, and the XIAO's ~100 mA onboard
  charger can't keep up. The cell would slowly discharge while plugged in.
- **Rev 4 (this document):** diode-OR. USB carries the ring whenever
  present, so the charger's full ~100 mA goes to the cell; the battery
  carries the ring only during an outage.

## Rev 4 wiring

![Wiring diagram](wiring-diagram.svg)

| From | To | Notes |
|---|---|---|
| LiPo + | XIAO `BAT+` pad (underside) | 1S 3.7 V **protected** cell; no reverse-polarity protection on the pads |
| LiPo − | XIAO `BAT−` pad (underside) | |
| XIAO `5V` pin | Schottky **D_usb** anode | SS34 / 1N5817–1N5822 class (3 A rating is overkill headroom, fine) |
| XIAO `BAT+` pad | Schottky **D_bat** anode | Second Schottky, same part |
| D_usb + D_bat cathodes | **VLED node** → Ring `VCC` | USB (~4.65 V) wins over battery (≤3.9 V) automatically |
| XIAO `D1` (GPIO1) | 330 Ω → Ring `DIN` | Resistor at the ring's DIN pad; keep lead < 10 cm |
| Ring `GND` | AO3400 **drain** | Ring's ground return is switched, not tied to GND |
| AO3400 **source** | GND | |
| AO3400 **gate** | XIAO `D2` (GPIO2) | 100 kΩ from gate to GND so the ring stays off during boot/flash |
| Ring `VCC` ↔ Ring `GND` | 1000 µF 6.3 V electrolytic | At the ring; stripe/short leg (−) to the ring-GND side |

`DOUT` unused. USB-C is for power, charging and flashing.

### Optional (phase 2) — power telemetry

Both dashed on the diagram; neither blocks the rev 4 rewire:

- **USB-present detect:** `5V` pin → 100 k → XIAO `D3` (GPIO21) → 100 k →
  GND. Divider puts ~2.5 V on the pin when VBUS is live — a plain GPIO
  binary sensor. Lets firmware/HA switch to **dark-when-OK on battery**
  (ISA-101 behavior returns during outages, ~5× the runtime).
- **Battery voltage:** `BAT+` → 200 k (2 × 100 k in series — no 200 k in
  stock) → XIAO `A0` (GPIO0) → 100 k → GND, then enable the commented
  `sensor:` block in the firmware.

## Why the FET

WS2812Bs draw ~1 mA per LED even when displaying black — ~20 mA continuous
for a 24-LED ring. The AO3400 opens the ring's ground return whenever the
ring is off, so a dark ring costs 0 mA. Firmware already drives the gate
(`ring_gate` follows the light state); with the light lit 24/7 the FET only
earns its keep when the ring is commanded off, but it costs nothing and
restores the 0 mA dark state the moment any schedule/policy comes back.

## Voltages

- **On USB:** ring sees ~4.6–4.7 V (5.0 minus one Schottky drop). Bonus:
  data-line margin *improves* vs the rev 1 direct-5V feed — VIH = 0.7 × 4.65
  ≈ 3.26 V, which the C6's 3.3 V data line now clears in spec.
- **On battery:** ring sees ~3.0–3.9 V (cell 3.4–4.2 V minus the drop).
  Slightly dim, dims further as the cell drains; red (the alert color) has
  the lowest forward voltage and survives longest. Colors stay recognizable.
- Never feed the ring from both 5V and BAT+ *without* the diodes — that
  back-powers VBUS from the cell and fights the charger.

## Power budget (honest numbers, rev 3 firmware behavior)

- C6 always-on Zigbee end device (radio RX on): **~25 mA** baseline
- Ring green @ 30 % 24/7: **~100–130 mA** (24 LEDs, one channel, plus idle)
- **On USB:** irrelevant — wall power carries it, charger tops off the cell
- **On battery, ring lit green:** ~130–155 mA total → **~2–2.5 days** on
  8000 mAh. Fine for its actual job: riding out power blips and outages.
- **On battery, dark-when-OK** (needs the phase-2 USB-detect + an HA or
  firmware policy): ~25 mA → **~10–13 days**
- Charging reality: onboard charger is ~100 mA — a full 8 Ah recharge takes
  3+ days of USB time. Top off is continuous in normal use (USB always
  attached); after a long outage, expect a slow refill or charge externally
  / add a TP4056-class 1 A board.

## Unit 1 conversion punch list (from as-built rev 1)

1. **Meter the LiPo first** — the board died when USB was pulled on
   2026-07-16 and again in the 2026-07-21 power loss, so the battery path
   has never actually worked: bad cell, bad BAT-pad joints, or simply left
   disconnected after the USB-flap debugging pass. A healthy 1S reads
   3.7–4.2 V at the connector. Fix this before anything else or the diodes
   are pointless.
2. Replace the USB cable (the 2026-07-21 overnight failure was a full USB
   de-enumeration — RAZORCREST saw no device at all; prime suspect is the
   same cable that was flapping the week before).
3. Lift ring VCC off the XIAO 5V pin. Fit **D_usb** (5V pin → anode) and
   **D_bat** (BAT+ → anode), join cathodes, run the joined node to ring VCC.
4. Lift ring GND from GND and route it through the AO3400: drain = ring
   GND, source = GND, gate = D2/GPIO2 with the 100 k gate-to-GND pulldown.
5. Verify the 1000 µF cap and 330 Ω survived the surgery.
6. No firmware change required — the gate logic is already in the flashed
   rev 3 image and is a no-op until the FET exists.

**Parts:** AO3400 (PartsBin ×100), 100 k (×70), 1000 µF 6.3 V (×10), 330 Ω
(plenty) — all on hand. **Schottky diodes are NOT in PartsBin** — order
SS34 or 1N5822 (2 per unit; get ~20, DeAnna's unit needs 2 more).

## Assembly notes

- Solder the cap and 330 Ω at the ring's pads; FET + 100 k + both diodes on
  a small perf square near the ring's GND pad. Everything tucks behind the
  ring in the 3D-printed diffuser/stand (design TBD).
- First flash over USB (`esphome run firmware/status-light-don-office.yaml`
  from the RAZORCREST build env, `esphome-status` dir). Pairing: open ZHA
  "Add device" — the light joins automatically when unjoined. Hold the BOOT
  button 5–20 s to factory-reset the Zigbee join.
