# Wiring — Office Status Light

One unit = XIAO ESP32-C6 + 24-LED WS2812B ring + two passives, powered by a
single 5 V USB-C wall wart into the XIAO.

![Wiring diagram](wiring-diagram.svg)

## Connections

| From | To | Notes |
|---|---|---|
| USB-C wall wart (5 V ≥2 A) | XIAO USB-C port | Single supply for everything |
| XIAO `5V` pin | Ring `5V` / `VCC` | VBUS passthrough; see power budget below |
| XIAO `GND` pin | Ring `GND` | Common ground — required for the data line to work |
| XIAO `D1` (GPIO1) | 330 Ω resistor → Ring `DIN` | Resistor as close to the ring's DIN pad as possible |
| Ring `5V` ↔ Ring `GND` | 1000 µF 6.3 V electrolytic | Across the rail **at the ring**; stripe/short leg (−) to GND |

The ring's `DOUT` pad is unused (single ring, no chaining).

## Power budget

24 × WS2812B at full white ≈ **1.4 A** — more than the XIAO's 5 V pin should
pass continuously. Two acceptable configurations:

1. **As drawn (default):** ring fed from the XIAO 5 V pin, with the firmware's
   `Max Brightness Limit` at ≤35 % (~0.5 A worst case). Fine for a status
   light — ISA-101 says it's dim or dark almost all the time anyway.
2. **Full brightness needed:** split a USB cable / use a screw-terminal USB
   breakout and feed the ring's 5 V/GND directly from the supply, with the
   XIAO's 5 V pin tied to the same rail (it accepts 5 V input there). Grounds
   common, data unchanged.

## 3.3 V logic vs 5 V ring

The C6 drives DIN at 3.3 V while the ring runs at 5 V — marginal per the
WS2812B datasheet (VIH = 0.7 × VDD = 3.5 V) but reliable in practice with
short leads and the 330 Ω resistor. If you see flicker or glitching:

- **Option A (preferred):** 74AHCT125 buffer between GPIO1 and DIN (powered
  from 5 V, shifts 3.3 V → 5 V cleanly), or
- **Option B (zero extra parts class):** drop the ring's supply through a
  1N4007 diode (~4.3 V at the ring) so 3.3 V clears the VIH threshold. Only
  valid in configuration 2 — don't diode-drop the XIAO's rail.

## Assembly notes

- Keep the GPIO1→DIN lead short (<10 cm); it's the only signal that cares.
- Solder the cap and resistor to the ring's pads, not mid-cable — the ring
  then presents a clean 3-wire pigtail (5V/GND/DIN) to the XIAO.
- The XIAO and pigtail tuck behind the ring inside the 3D-printed
  diffuser/stand (design TBD); USB-C port faces the base cutout.
- First flash over USB (`esphome run firmware/status-light-don-office.yaml`
  from the RAZORCREST build env), OTA after that.
