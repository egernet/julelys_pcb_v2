# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Julelys is a complete smart Christmas light system consisting of three components:
1. **julelys_pcb** (this repo) - KiCad HAT PCB for Orange Pi Zero 2
2. **julelys_manager** - Swift control software (runs on Orange Pi/Mac)
3. **julelys_esp32** - ESP32-C3 firmware (SPI slave)

The system drives a **440 SK6812 RGBW LED matrix (8×55)** and supports AI control via Model Context Protocol (MCP).

## System Architecture

```
Claude AI ──► Mac (julelys_manager) ──► Orange Pi Zero 2
                    MCP/SSH                    │
                                               │ SPI @ 2.5 Mbps
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
                                                 8×55 = 440 LEDs
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
- `power.kicad_sch` - AMS1117-3.3V regulator, auto-reset circuit, status LED
- `untitled.kicad_sch` - USB-C, CP2102N, JST connectors

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

## SPI Interface (Orange Pi ↔ ESP32)

| Signal | ESP32 GPIO | Function |
|--------|------------|----------|
| CS | GPIO 10 | Chip select |
| CLK | GPIO 3 | SPI clock |
| MOSI | GPIO 6 | Data in (frames from manager) |
| MISO | GPIO 2 | Data out |
| LED Data | GPIO 7 | RMT output to multiplexer |

**Frame format:** 1760 bytes = 440 LEDs × 4 bytes (RGBW)

## Connectors

| Reference | Type | Function |
|-----------|------|----------|
| J1 | 2×13 Pin Socket | Orange Pi Zero 2 GPIO header |
| J2 | 1×13 Pin Socket | GPIO breakout |
| J3 | 2×6 Pin Header | Debug/programming |
| J4-J11 | JST B3B-XH-A (×8) | LED strip outputs (5V, Data, GND) |
| J12 | 1×3 Pin Header | Auxiliary connector |
| P1 | USB-C | Power + programming |
| V_IN1, V_IN2 | Screw Terminal (×2) | External 5V input (parallel, ~20A total) |

## Power

- **Input:** 5V via USB-C or dual screw terminals (V_IN1 + V_IN2 in parallel)
- **LED current:** 8 strings × 55 LEDs × 60mA = 26.4A max (use adequate PSU)
- **Regulator:** AMS1117-3.3V (1A) for ESP32 and logic ICs
- **LED power:** Direct 5V passthrough to JST connectors (no reverse polarity protection)

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
