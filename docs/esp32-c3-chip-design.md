# Bare ESP32-C3 + native USB (test board, feature/v4-esp32-chip)

Research notes from 2026-10-14 for replacing the ESP32-C3-WROOM-02 module
with a bare chip and a chip antenna (A), and the CP2102N with the ESP32-C3's
native USB Serial/JTAG (B). Values marked **[rec]** are our own calculations
or recommendations. All other values come from Espressif documents.

Sources:
- [ESP32-C3 schematic checklist](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32c3/schematic-checklist.html)
- [ESP32-C3 datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-c3_datasheet_en.pdf)
- [ESP32-C3-MINI-1 datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-c3-mini-1_datasheet_en.pdf), Fig. 8-1 module schematic
- [ESP32-C3-WROOM-02 datasheet](https://www.espressif.com/sites/default/files/documentation/esp32-c3-wroom-02_datasheet_en.pdf)

## Chip

- **ESP32-C3FH4**, 4 MB in-package flash, LCSC **C2858491**, about $2.17 each.
  With in-package flash the SPI flash pins cannot be used for anything else.
- The ESP32-C3 without flash (C2838500) would need an external flash. The only
  JLC Basic NOR flash is W25Q128JVSIQ (C97521).

## Core circuit (Espressif reference values)

| Pin(s) | Circuit |
|---|---|
| VDD3P3 (2, 3) | 3V3, then 10 µF + 0.1 µF, then **L 2.0 nH (≥500 mA, e.g. C18216 LQG15HS2N0S02D)**, then 1 µF at the pins |
| VDDA (31, 32) | 1 µF + 10 nF |
| VDD3P3_RTC (11) | 0.1 µF |
| VDD3P3_CPU (17) | 0.1 µF |
| VDD_SPI (18) | 1 µF (powers the in-package flash at 3.3 V) |
| CHIP_EN (7) | 10 kΩ pull-up + 1 µF (both existing) |
| XTAL_P (30) / XTAL_N (29) | 40 MHz ±10 ppm. **24 nH in series on XTAL_P.** Load caps per CL, then trim with TX tone. |
| LNA_IN (1) | CLC match close to the chip: shunt C 1.2-1.8 pF, series L 2.0-3.0 nH, shunt C 1.2-1.8 pF. 50 Ω line to the antenna. |
| U0TXD (28) | 499 Ω series |
| GPIO18/19 (25/26) | USB D-/D+, 22-33 Ω series near the chip. D+ pull-up optional (footprint DNP). |
| Strapping | GPIO2 pull-up (R35), GPIO8 pull-up (R23), GPIO9 pull-up (R30). No large cap on GPIO9. |

## Parts (JLC/LCSC)

- Crystal 40 MHz 3225: **YXC X322540MPB4SI, C9010** (CL 15 pF, about $0.11).
  **[rec]** load caps C1 = C2 ≈ 20-22 pF (Cstray about 4 pF).
- Chip antenna: **Walsin RFANT3216120A5T, C127629** (3.2 x 1.6 x 1.2 mm,
  about $0.11, about 10,900 in stock). Chosen 2026-10-14. Unmatched it
  resonates at about 2.9 GHz; the datasheet uses a **6.8 nH series inductor**
  to reach 2.45 GHz. It needs a 6 x 6 mm ground-free corner/edge on all layers.
  The datasheet reference board is 20 x 45 mm, 1.2 mm thick. Layout: a pi
  footprint at the antenna (shunt C, series L = 6.8 nH, shunt C), with the
  shunts DNP to start, for later tuning. The alternative is Johanson
  2450AT18A100E (C89334, about $1, better documented, eval-board match 1.0 pF,
  2.7 nH shunt, 3.9 nH series).
- RF 0402: 1.2/1.5/1.8 pF = C1551/C1552/C1553; 2.2/2.7/3.0 nH = C86061/C77108/C19898.
- 24 nH 0402: C74301.
- 22 Ω 0402 C25092 (Basic). 499 Ω 0402 C4125.

## RF line

The JLC04161H-7628 stackup has 0.21 mm prepreg (Dk about 4.4) between F.Cu
and In1. A 50 Ω CPWG on F.Cu over In1 GND is about **0.32-0.34 mm wide with a
0.20-0.30 mm gap**. Check it with the JLC impedance calculator and order
impedance control. The matching values are only starting points: tune them
on the real board with a VNA, or at least check RSSI against a module board.

## Native USB (B)

- USB-C D+/D- go through ESD (D7) and 22 Ω to GPIO19/GPIO18. The CP2102N, the
  DTR/RTS auto-reset (U4) and the UART AND gate (U10) are removed.
- USB Serial/JTAG gives flashing with automatic reset, a CDC console and JTAG
  debugging (OpenOCD/GDB).
- Firmware: set `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG` (julelys_esp32).
- Recovery if the firmware disables USB: hold BOOT (SW1) and press RESET (SW2).
- UART0 stays on J1 pins 8/10 for the Pi or an external USB-UART adapter.
