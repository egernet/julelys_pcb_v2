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
| 12 | GPIO18 (PWM0) | FAN_PWM | Fan MOSFET gate |
| 18 | GPIO24 | FAN_TACH | Fan tach, pull-up to 3V3 |
| 27, 28 | ID_SD / ID_SC | - | Reserved for the HAT ID EEPROM, do not use |

Notes:
- GPIO5/6 default to input with pull-up, so the Pi never holds the ESP32 in reset
  or boot mode while it boots. Drive them low or leave them floating; never drive
  them high.
- ESP32 RX has two drivers, the Pi and the CP2102N (`USB_TX`). U10 (74LVC1G08)
  ANDs them, with R26/R27 10k pull-ups, so either one can talk to the ESP32.
- IO9 now has an external pull-up (R30 10k). It had none before.
- Pi 4: GPIO14/15 is the Linux console by default. Disable the console and set
  `enable_uart=1`. Pi 5 has a separate debug UART, so GPIO14/15 is free.

## Connectors

| Reference | Type | Function |
|-----------|------|----------|
| J1 | 2x20 Pin Socket | Raspberry Pi 40-pin GPIO header |
| J3 | 2x6 Pin Header | Debug/programming |
| J4-J11 | JST B3B-XH-A (x8) | LED strip outputs (5V, Data, GND) |
| J12 | 1x3 Pin Header | Auxiliary connector |
| J13 | 1x3 Pin Header | Fan (FAN-, +5V, tach) |
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
- **Fan control** (`power.kicad_sch`): J13 3-pin header (1 = FAN-, 2 = +5V,
  3 = tach). Q1 AO3400A switches the low side. R31 100R is the gate resistor
  from FAN_PWM (GPIO18), R32 100k the gate pull-down, D8 the flyback diode, and
  R33 10k pulls tach up to 3V3. Use the `pwm-fan`/`gpio-fan` overlay on the Pi.
  Tach readings are unreliable while PWM is switching. The HAT covers the Pi 5's
  own fan connector.
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
- **Test points** (TP1-TP33, footprint `TestPoint_Pad_1.0x1.0mm`):
  - Power: +5V (TP1), 3V3 (TP3), VBUS (TP18), GND (TP2, TP19-21).
  - SPI: TP4-7.
  - ESP32: EN (TP8), BOOT/IO9 (TP9), RXD (TP10), TXD (TP11), LED data at GPIO7
    (TP12), PI_TX (TP13), USB_TX (TP14).
  - Fan: FAN_PWM (TP15), FAN_TACH (TP16), FAN- (TP17).
  - LED path: mux data input (TP22), mux select AS0-AS2 (TP23-25), and all 8
    string outputs CH1-CH8 (TP26-33).
- **Power LEDs:** D5 on 5V (existing) and D9 on 3V3 (with R34 1k).

## v4 Layout (first draft)

- **Stackup:** 4 layers. F.Cu and B.Cu carry signals with a GND pour, In1.Cu is
  a solid GND plane, and In2.Cu is a solid +5V plane. All free copper is GND,
  stitched with vias. 2 layers was tried and dropped: routing did not complete,
  and the +5V pour got chopped up by signal traces.
- **Placement:**
  - J1 sits on the bottom side at the Pi's position.
  - V_IN1/V_IN2 and USB-C are on the left edge.
  - The 8 JST outputs are a 2x4 block next to the power input.
  - The ESP32 sits on the bottom edge with its antenna out over the edge, and
    no copper under the antenna on any layer.
  - The address LEDs D1-D3 are in one row above U2. The power LEDs D5/D9 are
    next to the power input.
- **High current:** locked 2 mm tracks, +5V on F.Cu and GND on B.Cu, run from
  the terminals to the JST pins. A 1 mm +5V track on both layers runs around the
  left end of J1 to Pi pins 2/4. 2 oz outer copper is recommended.
- **Status:** autorouted with Freerouting. Before ordering:
  - Route the 5 remaining connections by hand in KiCad, with the interactive
    router in shove mode: TXD and IO6 at the ESP32 (U1 pins 12 and 5), USB_TX
    (U5 pin 26 to R26), and VBUS at P1 A4/A9.
  - Move 3 FAN_N tracks away from the right board edge.
  - Clean up the silkscreen.
- **Cooling and mechanics:** a Pi 5 uses the official Active Cooler under the
  HAT, driven by the Pi itself. Mount the HAT with a GPIO stacking extension and
  about 16-18 mm standoffs. The standard 8.5 mm socket height does not clear the
  cooler, and the through-hole pins of the JST block sit above it. J13 stays for
  a case fan. There is no room for a fan on the HAT itself.
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
