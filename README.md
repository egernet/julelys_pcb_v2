# Julelys PCB v3

HAT (Hardware Attached on Top) til Orange Pi Zero 2 til styring af adresserbare SK6812 RGBW LED-strips.

## System Oversigt

```
Orange Pi Zero 2 ──► Julelys HAT ──► 8×55 SK6812 LED Matrix (440 LEDs)
        │                 │
        │ SPI @ 2.5 Mbps  │
        ▼                 ▼
  julelys_manager    ESP32-C3 (SPI slave)
                          │
                          ▼
                    74HC4051 MUX + Level Shifters
                          │
                          ▼
                    8x JST outputs
```

## Relaterede Repositories

| Repository | Beskrivelse |
|------------|-------------|
| [julelys_manager](https://github.com/egernet/julelys_manager) | Swift CLI + MCP server til LED-styring |
| [julelys_esp32](https://github.com/egernet/julelys_esp32/tree/feature/Hat) | ESP-IDF firmware (feature/Hat branch) |

## Features

- **ESP32-C3-WROOM-02** som SPI slave med RMT LED driver
- **8 LED-kanaler** via 74HC4051 multiplexer
- **Level shifting** (3.3V → 5V) via 74AHCT1G125
- **USB-C** til strøm og programmering (CP2102N)
- **Auto-reset** kredsløb til nem flashing
- **AI-styring** via Model Context Protocol (MCP)

## Nøglekomponenter

| Komponent | Funktion |
|-----------|----------|
| ESP32-C3-WROOM-02 | Hoved-MCU, SPI slave |
| CP2102N | USB-UART bridge |
| AMS1117-3.3 | 5V → 3.3V LDO |
| 74HC4051 | 8-kanal analog multiplexer |
| 74AHCT1G125 ×4 | Level shifters |

## Connectors

| Reference | Type | Funktion |
|-----------|------|----------|
| J1 | 2×13 Pin Socket | Orange Pi GPIO header |
| J4-J11 | JST B3B-XH-A ×8 | LED strip outputs |
| P1 | USB-C | Strøm + programmering |
| V_IN1 | Skrueterminal | Ekstern 5V |

## KiCad Filer

- `Julelys.kicad_pcb` - PCB layout
- `Julelys.kicad_sch` - Hovedskema
- `esp32.kicad_sch` - ESP32 subskema
- `power.kicad_sch` - Strømforsyning
- `untitled.kicad_sch` - USB-C og connectors

## Produktion

Produktionsfiler genereres til `production/`:
- Gerber filer (zipped)
- `bom.csv` - Stykliste
- `positions.csv` - Pick and place

## Licens

MIT
