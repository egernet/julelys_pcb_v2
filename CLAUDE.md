# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Julelys is a complete smart Christmas light system consisting of three components:
1. **julelys_pcb** (this repo) - KiCad HAT PCB for Raspberry Pi 3/4/5
2. **julelys_manager** - Swift control software (runs on Raspberry Pi/Mac)
3. **julelys_esp32** - ESP32-C3 firmware (SPI slave)

The system drives a **440 SK6812 RGBW LED matrix (8x55)** and supports AI control via Model Context Protocol (MCP).

## System Architecture

```
Claude AI ──► Mac (julelys_manager) ──► Raspberry Pi 3/4/5
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
- `power.kicad_sch` - AMS1117-3.3V regulator, auto-reset circuit, status LED, RPi power protection
- `untitled.kicad_sch` - USB-C, CP2102N, JST connectors, RPi GPIO header

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
| SP0503BAHT | D7 | USB ESD protection |
| SS34 | D6 | Schottky diode for RPi power isolation |

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

Standard 40-pin Raspberry Pi header compatible with RPi 3/4/5:

| Function | RPi Pins |
|----------|----------|
| 5V_OPI (isolated) | 2, 4 |
| 3.3V | 1, 17 |
| GND | 6, 9, 14, 20, 25, 30, 34, 39 |
| SPI0_MOSI | 19 |
| SPI0_MISO | 21 |
| SPI0_SCLK | 23 |
| SPI0_CE0 | 24 |

## Connectors

| Reference | Type | Function |
|-----------|------|----------|
| J1 | 2x20 Pin Socket | Raspberry Pi 40-pin GPIO header |
| J2 | 1x13 Pin Socket | GPIO breakout |
| J3 | 2x6 Pin Header | Debug/programming |
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

### Raspberry Pi Power Protection

The RPi receives isolated power via a dedicated protection circuit:

```
+5V (main) ────────────────────────► LED strips, ESP32, etc.
     │
     └──►|── D6 (SS34) ──┬── C15 (470uF) ──► 5V_OPI ──► J1 (RPi)
                         │
                         └── C16 (100nF)
                         │
                         └── GND
```

**Purpose:** When LEDs draw high current (up to 26A), voltage can dip momentarily. The Schottky diode prevents the LEDs from draining the bulk capacitor, keeping the Raspberry Pi stable during current spikes.

**Components:**
| Reference | Value | Function |
|-----------|-------|----------|
| D6 | SS34 (3A Schottky) | Prevents reverse current flow |
| C15 | 470uF/10V | Bulk capacitor for RPi holdup |
| C16 | 100nF | High-frequency bypass |

**Note:** The diode causes ~0.5V drop, so RPi receives ~4.5V. This is within spec (RPi operates down to ~4.0V).

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

## Migration from Orange Pi Zero 2

This PCB was originally designed for Orange Pi Zero 2 (26-pin header). Key changes for Raspberry Pi:

1. **J1 upgraded** from 2x13 to 2x20 (40-pin) header
2. **SPI pins remapped** to RPi SPI0 (GPIO 8/9/10/11)
3. **Power protection added** (D6, C15, C16) for stable RPi operation
4. **5V_OPI net** created for isolated RPi power supply
