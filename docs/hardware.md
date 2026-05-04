# Hardware

Component-level breakdown of the MAS-8710 main PCB. Sourced from a vision-pass analysis of disassembled-unit photographs (see [`PCB-Labeller-Demo`](https://github.com/danielrosehill/PCB-Labeller-Demo)) plus close-up bench inspection of the MCU markings. Annotated photographs of the board are bundled in the companion deployment repo at [`LAN-Clocks/internals/`](https://github.com/danielrosehill/LAN-Clocks/tree/main/internals).

## Block diagram (logical)

```
USB-C 5 V ─► reverse-polarity protection ─► AMS1117-3.3 LDO ─► 3.3 V rail
                                                                 │
                                                                 ├─► ESP8285 (MCU + WiFi + 1 MB internal flash)
                                                                 │       │
                                                                 │       ├── PCB antenna (meander trace)
                                                                 │       ├── 26 MHz crystal
                                                                 │       ├── 4 × tactile buttons (SET / + / − / alarm)
                                                                 │       ├── 2 × status LEDs (WIFI, 同步 sync)
                                                                 │       ├── piezo buzzer (alarm + key feedback)
                                                                 │       └── LCD-driver bus (proprietary, undecoded)
                                                                 │
                                                                 └─► CR1220 backup battery → RTC retention
```

## Main components

| Ref | Part | Role |
|---|---|---|
| **U6** | **ESP8285** (Espressif, QFN-32) | Main MCU. ESP8266 core with **1 MB embedded flash** — there is no separate SPI flash chip. Handles WiFi, NTP, button input, alarm logic, and drives the LCD controller. **Tasmota/ESPHome compatible** — flash mode `dout`, size `1MB`. |
| **U5** | Unidentified LCD/segment driver (SOIC) | Drives the front-panel segment LCD. Bus protocol between ESP and U5 is unlabelled — likely SPI or I²C, possibly a bit-banged segment protocol. **Not yet decoded.** This is the primary obstacle to writing a custom-firmware build that drives the display. |
| **U?** | AMS1117-3.3 LDO | 5 V → 3.3 V regulation for the MCU and peripherals. Standard part, dropout ~1 V, nominal load ≤ 1 A. |
| | SS34 schottky | Reverse-polarity / inrush protection on the USB-C 5 V input. |
| | 4R7 inductor | Filtering on the regulator input/output (not a switching topology — purely LC filter on the LDO). |
| | 26 MHz crystal | ESP8285 system clock reference. |
| **B1** | CR1220 coin cell | RTC backup. Maintains time across mains-power loss. Removing it does **not** brick the unit; on next power-up the clock re-syncs via NTP. |

## Connectors and headers

| Ref | Description |
|---|---|
| USB-C (front edge) | **Power-only.** No USB-serial bridge IC anywhere on the board (no CH340 / CP2102 / FT232 / FT2232) — D+/D− are not connected to anything that can flash the ESP. To reflash, you must use the UART header. |
| **5-pin header (bottom edge, primary)** | UART flashing interface. Carries `GND`, `3V3`, `TX` (ESP GPIO1), `RX` (ESP GPIO3), and `GPIO0`. Pin order is **not** marked on the silkscreen — verify with continuity before connecting. See [`docs/osd-buttons.md`](osd-buttons.md) for the user-side, and the LAN-Clocks [`flashing-guide.md`](https://github.com/danielrosehill/LAN-Clocks/blob/main/docs/flashing-guide.md) for the procedure. |
| 5-pin header (top edge / secondary) | Likely runs to the front-panel LCD assembly (segment data, not UART). Not useful for flashing. Worth probing if you're reverse-engineering the LCD protocol. |

## User interface elements

| Element | Notes |
|---|---|
| 4 × tactile buttons | `SET`, `+`, `−`, `alarm`. Wired straight to ESP GPIOs (specific pin mapping not yet identified — would need probing or `ESPHome` GPIO discovery). |
| 2 × LEDs (rear) | `WIFI` (link / association status), `同步` (NTP sync indicator). Likely on dedicated GPIOs separate from the button pins. |
| Piezo buzzer | Alarm tone and key-press feedback. Drive line not yet identified. |
| Front-panel LCD | 4-digit 7-segment with date and ambient temperature regions. Driven by U5 over the proprietary bus. |
| LDR (some units) | Ambient-light sensor for the `AL:1` auto-brightness mode. Not present on every unit. |

## What's not on the board

The vision-pass analysis flagged several footprints that were investigated and ruled out:

- **No external SPI flash chip.** The ESP8285's 1 MB internal flash is the only program store. This is the key reason the chip is ESP8285 rather than ESP8266EX.
- **No external RTC IC** (the earlier draft analysis suggested an RX8025 SOIC; bench inspection on the ESP8285-based units doesn't confirm one). The CR1220 backs up the ESP's internal RTC across power cycles; NTP re-syncs on boot anyway.
- **No USB-serial bridge.** USB-C is power-only.
- **An unpopulated PCB antenna footprint** is visible on the front PCB. Likely an alternative WiFi-module footprint or an unused 433/315 MHz / WWVB / DCF77 receiver pad — speculative, and not load-bearing for the current firmware.
- **An unpopulated secondary 5-pin header.** Function unknown — likely factory-test or display-debug. Not required for normal operation or flashing.

## Open questions

These remain unresolved and would require bench work to answer:

1. **U5 part number.** A clear macro photograph of the SOIC top markings is the fastest path to identification. Likely candidates: HT16K33, MAX7219, TM1638, or a 74HC595 shift-register chain. Without this, no off-the-shelf ESPHome display component will work for a custom-firmware build.
2. **Bus protocol between ESP8285 and U5.** SPI, I²C, or bit-banged segment data — needs logic-analyser capture during normal operation.
3. **GPIO mapping of the four buttons and two LEDs.** Probe each pin during a normal boot, or fuzz-discover via ESPHome's binary-sensor scan.
4. **Whether all three deployed units share the same controller and revision.** The two `MAS8710BN*` (older firmware) units and the one `ESP-*` (newer firmware) unit may use different ESP variants or different LCD-driver revisions. Open both case styles and compare before assuming a single procedure fits all.

## Caveats on the source data

The component IDs above came from a two-pass vision-LLM analysis on photographs (see [`PCB-Labeller-Demo`](https://github.com/danielrosehill/PCB-Labeller-Demo)). The model was **wrong about the MCU variant** — it labelled it `ESP8266EX`, but a close-up of the chip die markings shows `ESP8285`. Treat all other LLM-derived part guesses (notably U5) as hypotheses pending bench confirmation.
