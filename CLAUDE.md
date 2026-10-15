# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Julelys is a complete smart Christmas light system consisting of three components:
1. **julelys_pcb** (this repo) - KiCad HAT PCB for Raspberry Pi 4/5
2. **julelys_manager** - Swift control software (runs on Raspberry Pi/Mac)
3. **julelys_esp32** - ESP32-C3 firmware (SPI slave)

The system drives a **440 SK6812 RGBW LED matrix (8x55)** and supports AI control via Model Context Protocol (MCP).

## System Architecture

```
Claude AI ──► Mac (julelys_manager) ──► Raspberry Pi 4/5  
                    MCP/SSH                    │
                                               │ SPI0 @ 2.5 Mbps
                                               ▼
                                    ┌──────────────────┐
                                    │   JULELYS HAT    │
                                    │   (this PCB)     │
                                    │                  │
                                    │  ESP32-C3 ─────► 74HC4051 MUX
                                    │      │           + Level Shifters
                                    │      ▼                 │
                                    │  RMT Driver            │
                                    └──────────────────┬─────┘
                                                       │ 8 channels
                                                       ▼
                                              SK6812 LED Matrix
                                                 8x55 = 440 LEDs
```

## Related Repositories

| Repository | Description |
|------------|-------------|
| [julelys_manager](https://github.com/egernet/julelys_manager) | Swift CLI + MCP server for LED control |
| [julelys_esp32](https://github.com/egernet/julelys_esp32/tree/feature/Hat) | ESP-IDF firmware (feature/Hat branch) |

## PCB Files (KiCad 7+)

**Schematics:**
- `Julelys.kicad_sch` - Top-level (multiplexer, level shifters, connectors)
- `esp32.kicad_sch` - ESP32-C3-WROOM-02 and decoupling
- `power.kicad_sch` - AMS1117-3.3V regulator, status LED, RPi power
- `untitled.kicad_sch` - USB-C, CP2102N, auto-reset, JST connectors, RPi GPIO header

**Layout:**
- `Julelys.kicad_pcb` - Main PCB layout
- `Julelys LED/` - Separate LED panel PCB

**Libraries:**
- `CP2102N_A02_GQFN28/` - USB-UART bridge
- `BC847BDW1T1G/` - Dual transistor (auto-reset)
- `SS34_E3_57T/` - Schottky diode
- `ul_74HC4051D-BJ-/` - 8-channel multiplexer
- `B3B_XH_A/` - JST XH connector
- `USB4125_GF_A/` - USB-C connector
- `3D/` - 3D step models

## Key Components

| Component | Reference | Function |
|-----------|-----------|----------|
| ESP32-C3-WROOM-02 | U1 | Main MCU, SPI slave, RMT LED driver |
| CP2102N-A02-GQFN28 | U5 | USB to UART for programming |
| AMS1117-3.3 | U3 | 5V → 3.3V LDO regulator |
| 74HC4051 | U2 | 8-channel analog MUX for row selection |
| 74AHCT1G125 | U6-U9 | Level shifters (3.3V → 5V) |
| BC847BDW1T1G | U4 | Dual transistor for DTR/RTS auto-reset |
| 74LVC1G08 | U10 | ANDs CP2102N TX and Pi TX into ESP32 RX |
| AO3400A | Q1 | Fan low-side switch |
| SP0503BAHT | D7 | USB ESD protection |

## SPI Interface (Raspberry Pi ↔ ESP32)

**Raspberry Pi SPI0 pinout:**
| RPi Pin | RPi GPIO | Signal | ESP32 GPIO | Function |
|---------|----------|--------|------------|----------|
| 24 | GPIO8 | CS | GPIO 10 | Chip select (directly active) |
| 23 | GPIO11 | SCLK | GPIO 3 | SPI clock |
| 19 | GPIO10 | MOSI | GPIO 6 | Data in (frames from manager) |
| 21 | GPIO9 | MISO | GPIO 2 | Data out |
| - | - | LED Data | GPIO 7 | RMT output to multiplexer |

**Frame format:** 1760 bytes = 440 LEDs x 4 bytes (RGBW)

## Raspberry Pi GPIO Header (J1)

Standard 40-pin header for RPi 4/5. The first v4 draft kept the Orange Pi pin
numbers (pin 17 = Pi 3V3 on MISO, pin 30 = GND on ESP32 RX). Check J1 in the
netlist after every schematic edit. KiCad 7's CLI has no ERC, so run ERC in the
GUI:
`kicad-cli sch export netlist --format kicadsexpr -o /tmp/j.net Julelys.kicad_sch`.

All pins not listed are no-connect.

| RPi Pin | RPi GPIO | Net | ESP32 / HAT side |
|---------|----------|-----|------------------|
| 2, 4 | 5V | +5V | Pi powered from the HAT, no diode |
| 1, 17 | 3V3 | - | Leave unconnected (do not tie to the HAT's 3V3) |
| 6, 9, 14, 20, 25, 30, 34, 39 | GND | GND | |
| 19 | GPIO10 | MOSI | ESP32 GPIO 6 |
| 21 | GPIO9 | MISO | ESP32 GPIO 2 |
| 23 | GPIO11 | SCLK | ESP32 GPIO 3 |
| 24 | GPIO8 | CS | ESP32 GPIO 10 |
| 8 | GPIO14 (TXD) | PI_TX | U10 AND gate -> ESP32 RX (GPIO 20) |
| 10 | GPIO15 (RXD) | PI_RX | ESP32 TX (GPIO 21) |
| 29 | GPIO5 | PI_EN | ESP32 EN (CHIP_UP), via R28 1k |
| 31 | GPIO6 | PI_BOOT | ESP32 GPIO 9, via R29 1k |
| 37 | GPIO26 | PI_GPIO26 | ESP32 GPIO 0 via R37 1k. Test board: sensor ADC input (keep the Pi pin an input) |
| 36 | GPIO16 | PI_GPIO16 | ESP32 GPIO 1 via R38 1k. Test board: U6 /OE for sensor read-back (keep the Pi pin an input) |
| 27, 28 | ID_SD / ID_SC | - | Reserved for the HAT ID EEPROM, do not use |

Notes:
- GPIO5/6 default to input with pull-up, so the Pi never holds the ESP32 in reset
  or boot mode while it boots. Drive them low or leave them floating; never drive
  them high.
- ESP32 RX has two drivers, the Pi and the CP2102N (`USB_TX`). U10 (74LVC1G08)
  ANDs them, with R26/R27 10k pull-ups, so either one can talk to the ESP32.
- IO9 now has an external pull-up (R30 10k). It had none before.
- The spare lines (IO0/IO1 to Pi GPIO26/16) are free for interrupt or handshake
  use. The 1k resistors protect against both sides driving at once. IO0/IO1 are
  not strapping pins.
- **Test board (`feature/v4-esp32-chip`):** IO0/IO1 are used for sensor
  read-back on the 8 JST outputs, see `docs/sensor-input.md`. IO1 high
  tri-states the LED data shifter U6, the mux selects a channel, and IO0
  (ADC1_CH0) reads it through R52/R53. R51 keeps U6 enabled at boot. If a Pi
  is fitted, leave its GPIO26/16 as inputs.
- **Dev board use without a Pi:** J1 (the socket on the bottom) doubles as the
  ESP32 header, used with male dupont wires. It carries SPI (IO6/2/3/10 on pins
  19/21/23/24), TXD (pin 10), RXD (pin 8, through U10), EN/BOOT (29/31, 1k),
  IO0/IO1 (37/36, 1k), 5V (2/4) and GND. IO4/5/8 (mux select) and IO7 (LED data)
  are used on the board and can be probed on the test points. This replaces the
  v3.1 debug header J3, which was removed in v4.
- Pi 4: GPIO14/15 is the Linux console by default. Disable the console and set
  `enable_uart=1`. Pi 5 has a separate debug UART, so GPIO14/15 is free.

## Connectors

| Reference | Type | Function |
|-----------|------|----------|
| J1 | 2x20 Pin Socket | Raspberry Pi 40-pin GPIO header |
| J4-J11 | JST B3B-XH-A (x8) | LED strip outputs (5V, Data, GND) |
| J12 | 1x3 Pin Header | Auxiliary connector |
| P1 | USB-C | Power + programming |
| V_IN1, V_IN2 | Screw Terminal (x2) | External 5V input (parallel, ~20A total) |

## Power

### Main Power
- **Input:** 5V via USB-C or dual screw terminals (V_IN1 + V_IN2 in parallel)
- **LED current:** 8 strings x 55 LEDs x 60mA = 26.4A max (use adequate PSU)
- **Regulator:** AMS1117-3.3V (1A) for ESP32 and logic ICs
- **LED power:** Direct 5V passthrough to JST connectors

### Raspberry Pi Power (v4)

The Pi is powered from the HAT: +5V goes straight to J1 pins 2/4. **There is no
diode.** The SS34 from the earlier draft (D6, 3A, 0.5V drop) is too weak for a
Pi 5 (5A) and drops too much voltage. Parallel Schottkys do not share current
well. If isolation is needed later, use a MOSFET ideal diode.

- Keep the bulk capacitor (470uF) plus 100nF at the header. Without a diode,
  surviving LED current spikes depends on the PSU and on thick copper.
- **Never plug in the Pi's own USB-C while the HAT powers it.** The two supplies
  would fight each other.
- Pi 5 needs `PSU_MAX_CURRENT=5000` in its EEPROM config (`rpi-eeprom-config -e`).
  Without it, USB is limited to 600mA and the Pi warns about the supply.

## Production Output

Generated to `production/` with timestamped subfolders:
- Gerber files (zipped for fab upload)
- `bom.csv` - Bill of materials
- `positions.csv` - Pick and place coordinates
- `netlist.ipc` - IPC netlist

## Software Features (via julelys_manager)

- 12 built-in animations (Rainbow, Stars, Fireworks, Matrix, etc.)
- Custom JavaScript sequences
- Persistence of active sequences
- MCP integration for Claude AI control
- Three modes: Real (hardware), App (macOS GUI), Console (ANSI simulation)

## ESP32 Firmware Features

- SPI slave receiving frames at 2.5 Mbps
- Double buffering with mutex protection (no tearing)
- Automatic rainbow animation on idle (>5 seconds)
- FreeRTOS task-based architecture
- UART console for diagnostics

## v4 Requirements (feature/v4)

v4 is a HAT for **Raspberry Pi 4/5**. It does everything v3.1 does (ESP32-C3,
mux, level shifters, CP2102N/USB-C, 8 JST outputs, screw terminals), plus:

- **Board:** 65 x 56 mm HAT outline with M3 holes 58 x 49 mm apart. The layout
  restarts from the outline; the v3.1 layout is in git history (`0ffda38`) and
  `Julelys-backups/`.
- **Pi powered from the HAT**, with no diode (see Power).
- **No fan control:** the Pi 5 drives its own Active Cooler. The J13/Q1 fan
  circuit from the first v4 draft was removed, and Pi GPIO18/24 are free.
- **J3 removed:** J1 doubles as the dev header (see the J1 table).
- **Pi can flash the ESP32** over UART plus EN/BOOT (see the J1 table), so no
  buttons are needed. CP2102N/USB-C stays for debugging.
- **Auto-reset fixed.** On v3.1 the U4 collectors were swapped, so EN went to
  IO9 and IO9 to EN. Now the transistor with base <- DTR and emitter -> RTS
  drives EN (CHIP_UP), as in the Espressif reference. This is why esptool's
  reset failed on v3.1.
- **Mux LEDs D1-D3 moved off GPIO8.** GPIO8 is a strapping pin and must be high
  for download mode, but the LED and 270R pulled it towards about 1.8V. D1-D3
  now hang off the 5V side (`AS0`-`AS2`, after the level shifters), with
  R1-R3 = 1k.
- **Test points** (footprint `TestPoint_Pad_1.0x1.0mm`), almost all on the
  **bottom side** to free space on top. Probe them with the HAT off the Pi.
  TP9 (IO9) and TP10 (RXD) stay on top.
  - Power: +5V (TP1), 3V3 (TP3), VBUS (TP18), GND (TP2, TP19-21).
  - SPI: TP4-7.
  - ESP32: EN (TP8), BOOT/IO9 (TP9), RXD (TP10), TXD (TP11), LED data at GPIO7
    (TP12), PI_TX (TP13), USB_TX (TP14).
  - LED path: mux data input (TP22), mux select AS0-AS2 (TP23-25), and the
    string outputs CH1-CH8 (TP26-33).
- **Power LEDs:** D5 on 5V (existing) and D9 on 3V3 (with R34 1k).

## Design notes

- `docs/noise-analysis-v4.md`: noise/EMI analysis of v4 (26fc348) with
  findings and fixes.
- `docs/noise-analysis-test-board.md`: the same analysis on the test board,
  compared with v4, plus re-measurements after moving C36, U11/U12, D10 and
  the USB route.
- `docs/sensor-input.md`: reading up to 8 analog sensors through the LED
  JST outputs (test board).
- `docs/esp32-c3-chip-design.md`: bare ESP32-C3FH4 + chip antenna + native USB
  (test board, branch `feature/v4-esp32-chip`): reference values, parts and
  layout status. The test board is fully routed (0 DRC errors, 0
  unconnected). The QFN area uses 0.15 mm clearance via the `QFN_FAN` rule
  area.

## v4 Layout (first draft)

- **HAT mechanical spec** (raspberrypi/hats `hat-board-mechanical.pdf`): the
  board is 65 x 56.0 mm, because 56.5 mm applies only to an SMT GPIO header. It
  has 3 mm corner radii. The mounting holes are 2.75 mm NPTH, 58 x 49 mm apart
  and 3.5 mm from the edges. Each hole has a keepout zone of 6.4 mm diameter on
  all copper layers (spec: land at least 6.2 mm, not connected to GND). The
  optional camera and display flex cutouts are left out, because the ESP32 and
  the power input sit there.
- **Stackup:** 4 layers. F.Cu and B.Cu carry signals with a GND pour, In1.Cu is
  a solid GND plane, and In2.Cu is a solid +5V plane. All free copper is GND,
  stitched with vias. 2 layers was tried and dropped: routing did not complete,
  and the +5V pour got chopped up by signal traces.
- **Placement (floorplan v2):**
  - J1 sits on the bottom side at the Pi's position.
  - V_IN1/V_IN2 and USB-C are on the left edge, with the 8 JST outputs in a 2x4
    block next to them.
  - **The ESP32 (U1) sits top-right, right under J1, with its antenna at the
    right board edge.** The keepout has no copper on any layer. The antenna is
    only used in dev-board / stream mode without a Pi. On the Pi the link is
    SPI, so the Pi's USB/Ethernet metal next to the antenna does not matter.
    SPI, UART, EN/BOOT and the spare lines to J1 are therefore short.
  - The ESP32 caps (C2/C3/C6/C7, R30, R35-R38) sit in a row between J1 and the
    module.
  - The mux U2 is bottom-centre, with U6-U9, R14-R21 and C18-C21 around it. The
    address LEDs D1-D3 are in one row under it, and J12 is beside them.
  - The regulator U3 (C4/C5, D9/R34) and the Pi bulk cap C15 are bottom-left,
    next to the CP2102N U5. The power LED D5 and R5 sit at the left edge.
  - SW1/SW2 are on the right edge, under the antenna.
  - All test points except TP9/TP10 are on the bottom side.
- **High current:** locked 2 mm tracks, +5V on F.Cu and GND on B.Cu, run from
  the terminals to the JST pins. The Pi's 5V (J1 pins 2/4) is fed only through
  the In2 +5V plane. A track around the left end of J1 would cross the
  mounting-hole land. 2 oz outer copper is recommended.
- **Status:** fully routed. 0 DRC errors (JLCPCB rules), 0 unconnected,
  0 ERC errors, schematic parity clean. Freerouting did most of it; the last
  connections were routed by a script, using In2 for a few short crossings.
  Left to do by hand:
  - Clean up the front silkscreen (overlapping references).
  - Review the routing.
- **Decoupling distances** (cap pad to IC power pin): C3 2.3, C2 3.0, C6 3.0,
  C7 2.2, C8 2.2, C9/C10 2.0, C4/C5 2.5, C17 1.7, C18 2.0, C19 3.2, C20 1.7,
  C21 1.7 mm. C16 (Pi 5V) is 5.0 mm from J1 pin 4, because the J1 pins need a
  clear area around them. A 74AHCT125 quad buffer (TSSOP-14) was tried in place
  of U6-U9 but did not fit.
- **Schematic review fixes (Codex review):**
  - R35 10k pulls GPIO2 (a strapping pin) up. The Pi's MISO has a default
    pull-down.
  - R36 10k pulls IO7 (LED data) down, so the LEDs do not flash while the ESP32
    boots.
  - C5 is 22uF for AMS1117 stability.
  - C7 on IO9 is 1nF. Espressif advises against large caps on GPIO9.
  - C18-C21 are 100nF decoupling caps for the level shifters U6-U9.
  Known and accepted: JST-XH is rated 3 A per pin against 3.3 A for a full-white
  string, with no per-string fuse (limit brightness in firmware). USB VBUS feeds
  +5V through D4 (1 A), so program over USB without LED load.
- **Fab rules:** JLCPCB 4-layer, 1 oz outer / 1 oz inner
  (https://jlcpcb.com/capabilities/pcb-capabilities). The limits are in
  `Julelys.kicad_pro` and `Julelys.kicad_dru`:
  - min track and space 0.1 mm
  - via annular ring at least 0.15 mm (0.6/0.3 vias; 0.7/0.3 gives the
    recommended 0.2)
  - PTH hole to track 0.28 mm, pad hole to hole 0.45 mm
  - copper to edge 0.3 mm
  - silkscreen text at least 1.0 mm high and 0.15 mm thick, 0.15 mm from pads
  2 oz outer copper would need 0.8 mm vias (annular ring 0.254), which do not
  fit around U5/U1.
- **Cooling and mechanics:** a Pi 5 uses the official Active Cooler under the
  HAT, driven by the Pi itself. Mount the HAT with a GPIO stacking extension and
  about 16-18 mm standoffs. The standard 8.5 mm socket height does not clear the
  cooler, and the through-hole pins of the JST block sit above it. There is no
  fan control on the HAT.
- **Screw terminals:** V_IN1/V_IN2 point their 45-degree wire entry at the left
  board edge. The footprint is rotated 270 degrees and the symbol is mirrored,
  so pin 1 = +5V and pin 2 = GND (the opposite of v3.1). A 2-pole terminal is
  symmetric, so this only changes which screw is which.
- **3D models:** every physical part has one. Library parts use
  `${KICAD10_3DMODEL_DIR}`. HRO USB-C, ESP32, CP2102N, BC847 and JST models live
  in the project (`3D/` and the part folders). Test points and mounting holes
  have none.
- **Tools:** KiCad 10 (`kicad-cli sch erc`, `kicad-cli pcb drc`). The board
  file is in KiCad 10 format. The schematics are still in KiCad 7 format until
  they are saved from KiCad 10.

## Migration from Orange Pi Zero 2

This PCB was originally designed for Orange Pi Zero 2 (26-pin header). Changes for Raspberry Pi:

1. **J1 upgraded** from 2x13 to 2x20 (40-pin) header, and remapped to the Pi pinout.
2. **J2 removed** (Orange Pi breakout), together with R24/R25 (22R on USB D+/D- to J2).
3. **Auto-reset** (U4, R12, R13) moved from `power.kicad_sch` to `untitled.kicad_sch`.
4. **D6/C15/C16 and the `5V_OPI` net** from the first draft are dropped in favour of direct power (see Power).
