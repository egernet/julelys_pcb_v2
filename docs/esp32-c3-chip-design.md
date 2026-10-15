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
| U0TXD (28) | 499 Ω series (fitted: 510 Ω, the nearest JLC Basic part) |
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
- 22 Ω 0402 C25092 (Basic). TX series resistor: 510 Ω 0402 C25123 (Basic) instead of 499 Ω C4125 (Extended).

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

## Layout status (2026-10-15)

Floorplan v3: the ESP32-C3FH4 (U1) is top-right under J1, with pin 1 (RF)
facing the Walsin antenna at the right edge (6 x 6 mm keep-out, feed track
allowed, no pour). The parts around the QFN were re-placed so that every pin
can escape:
- RF chain runs straight out of pin 1 on F.Cu: C32 (shunt) at the pin, L3
  (series), C33 (shunt), then a 45-degree line up to L4/C34 and the antenna.
  RF_IN 2.3 mm, RF_LINE 5.4 mm, ANT 3.6 mm, all 0.33 mm wide, no vias.
- VDD3P3: pins 2/3 run below the IO escapes to C24 (1 uF), then L1 (2 nH),
  then +3.3V. C24 is 2.6 mm from pin 2.
- Crystal Y1 sits below pins 29/30. XTAL_P goes through L2 (24 nH) to Y1
  pin 1, XTAL_N runs around the Y1 GND pad to pin 3. All on F.Cu, no vias.
  XTAL_N is 8.1 mm because it has to go around that pad. Load caps C30/C31
  sit at the crystal pins.
- Decoupling at the pins: VDDA C25/C26 bottom-right (1.5/2.3 mm), RTC C27
  above pin 11 (2.8 mm), CPU C28 (1.5 mm), VDD_SPI C29 (1.7 mm).
- USB series resistors R40 (D-) and R39 (D+) left of pins 25/26. R41 (DNP)
  and R42 (TX) next to them.
- Moved out of the fan-out area: C1/C6 (EN caps) to the right next to R4,
  R23/R36 (A2/IO7 pull-ups) next to the level shifters U9/U6, R30 a little
  to the left, TP5/TP9 (bottom side) onto their nets.
- Level shifters U6-U9 in one row (input pin towards the ESP32, output towards
  the mux), each with its 100 nF (C18-C21) right below VCC. C8 is next to U2
  pin 16.
- LED output pull-downs in one row below the JST connectors: R47-R50,
  R43-R46. Their GND pads face down and are tied with a short GND bus.
- TVS arrays near the connectors: U11 (CH1-4) right of the J4-J7 column,
  U12 (CH5-8) below the V_IN terminals. D10 (+5V TVS) near USB-C/V_IN2.
- C36 (Pi bulk, 100 µF 1206) next to J1 pins 2/4, with C16.
- USB D+/D- run below the JST block and do not cross the LED current area.
  R40 sits next to R39 under U1 pins 25/26.

Routing: F.Cu, **In2.Cu (mixed: signals plus the +5V pour)** and B.Cu. In1.Cu
is a solid GND plane, the reference for the signals and the RF line.
Stackup: JLC04161H-7628 (0.21 mm prepreg). RF net class: 0.33 mm width.
The zones join SMD pads solid; through-hole pads use thermal reliefs, except
J1, which is solid.

**Fully routed: 0 DRC errors, 0 unconnected, no dangling tracks or vias.**

QFN fan-out: every pin leaves on F.Cu in a short stub. The right and top pins
then drop to In2/B.Cu through staggered vias. Signal tracks at the QFN are
0.15 mm. A rule area named `QFN_FAN` around U1 lowers the clearance to
0.15 mm (rule "QFN fan-out" in `Julelys.kicad_dru`), which is still well
inside JLC's 0.1 mm limit. Outside that area the 0.2 mm net class clearances
apply.
