# Julelys v4 test board - JLCPCB production files

Generated 2026-10-15 from `feature/v4-esp32-chip` (commit 542077f), KiCad 10.
DRC: 0 errors, 0 unconnected. ERC: 0 errors.

## Files

| File | Upload as |
|---|---|
| `Julelys_v4-test_gerber.zip` | PCB "Add gerber file" (all copper, mask, silk, paste, edge, PTH/NPTH drill) |
| `BOM_JLCPCB.csv` | Assembly BOM |
| `CPL_JLCPCB.csv` | Assembly CPL / pick and place |
| `gerber/` | The same files unzipped, for review |

## PCB order settings

- Layers: **4**, size 65 x 56 mm, thickness 1.6 mm
- Stackup: **JLC04161H-7628** (the RF line is sized for its 0.21 mm prepreg)
- Impedance control: **yes** (RF line 0.33 mm on F.Cu over In1 GND, about 50 ohm)
- Copper: 1 oz outer / 0.5 oz inner (what JLC04161H-7628 uses; the RF line is
  calculated with it). "Specify Stackup" must be **Yes** to pick JLC04161H-7628.
- Min track/space: 0.15 mm at the ESP32 QFN, 0.2 mm elsewhere. Vias 0.6/0.3 mm.
- Via covering: tented
- Surface finish: **ENIG** recommended (0.5 mm pitch QFN); lead-free HASL also works
- Remove order number: specify a location, or "No" if the option costs extra

## Assembly notes

- **SMD parts only.** JLC places only the SMD parts, all on the top side.
  P1 (USB-C) is an SMD connector with four shield legs; JLC assembles it with
  the SMD parts.
- **Solder by hand (not in BOM/CPL):** J1 (2x20 Pi socket, bottom side),
  J4-J11 (JST XH), J12 (1x3 header), V_IN1/V_IN2 (screw terminals).
- **Not placed (DNP):** C34, C35 (antenna tuning), R41 (USB D+ pull-up). They
  are left out of the BOM and CPL.
- Test points and mounting holes have no parts.
- **LCSC numbers:** every BOM line is filled in. All were checked against
  JLC's parts library on 2026-10-15 (value, package, stock). Basic parts are
  used wherever one exists. About 15 lines are Extended parts (U1, U2, U6-U9,
  U11/U12, Y1, AE1, L1-L4, P1, D7, D10, C15, SW1/SW2). R42 (510 ohm) and D4
  (B5819W) were switched to Basic parts to save two setup fees; JLC charges a
  setup fee per Extended part type, except for "preferred" ones (C1552, D10,
  U11/U12).
- Capacitors on +5V are rated 10 V or more, except C36 (100 uF 1206, 6.3 V,
  the largest Basic part). At 5 V it gives roughly 30-40 uF.
- **Rotations and centres are already corrected for JLC** in
  `CPL_JLCPCB.csv`. Each part's JLC footprint (JLCEDA/EasyEDA official library,
  https://lceda.cn/ / https://easyeda.com) was fitted onto the board's own pad
  positions by pin function, not just pad number. All fits are within the pad
  size (worst 0.26 mm, from slightly different pad sizes).
  `CPL_KiCad_uncorrected.csv` is the raw KiCad export, for comparison.

  | Part | Change vs. KiCad |
  |---|---|
  | U2 (SOIC-16) | +270 deg |
  | U6-U9 (SC-70-5), U11/U12 (SOT-23-6) | +270 deg |
  | U3 (SOT-223), D7 (SOT-143), C15 (electrolytic, + pad) | +180 deg |
  | P1 (USB-C) | centre moved 1.57 mm (different footprint origin) |
  | AE1 (antenna) | centre moved 1.76 mm (custom Walsin land pattern); feed = pin 1 in both |
  | D5/D9 (green LED) | none: JLC numbers its cathode pad 2, so it was matched by function |

  Still glance at JLC's placement preview before paying; it is the final word.
- Firmware for native USB: `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG`.
