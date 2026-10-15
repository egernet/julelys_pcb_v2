# Noise analysis - test board (feature/v4-esp32-chip, commit 8bf60ad)

Date: 2026-10-14. Same method as `docs/noise-analysis-v4.md`: numbers come
from the board geometry (track lengths, layers, placement) plus estimates.
Nothing was simulated. The QFN fan-out (11 nets plus R41) is not routed yet,
so the nets that end at U1 are measured without their last 1-3 mm. Re-check
those after the hand routing.

## Summary

The test board fixes most of the v4 findings. The control nets are much
shorter, the CH outputs have pull-downs, TXD has its 499 Ω, and the GND
stitching is denser. Three fixes are only half done, because the parts are on
the board but **in the wrong place**: the Pi bulk cap C36, the TVS arrays on
the LED outputs, and the V_IN TVS D10. The USB pair is now the weakest signal:
about 58 mm, not routed as a pair, and crossing the LED area.

## Findings compared with v4

| # | v4 finding | Test board | Status |
|---|---|---|---|
| 1 | No pull-downs on CH1-CH8 | R43-R50 10 kΩ on every CH line | **Fixed** |
| 2 | Pi 5V: bulk cap C15 is 49 mm from J1 | C36 100 µF was added, but it sits **39 mm** from J1 pin 2/4 (C16 100 nF is still 5 mm) | **Not fixed**: move C36 next to J1 |
| 3 | Copper for the full LED current | Power tracks unchanged | Unchanged (firmware limit, 1 oz inner) |
| 4 | B.Cu signals reference the +5V plane | B.Cu signals 704 -> 406 mm, CLK 42 -> 21 mm, CS 33 -> 19 mm, A0 48 -> 29 mm. But In2 now carries **377 mm** of signals (v4: 27 mm), and the +5V plane shrank from 2569 to 2288 mm² (still one piece). TXD runs 51 mm on B.Cu. | **Better**, not solved |
| 5 | No ESD on LED outputs / V_IN | U11/U12 SRV05-4 and D10 SMBJ5.0A added, but the TVS pins are **9-31 mm** from the JST data pins and D10 is **28-35 mm** from V_IN | **Half fixed**: see below |
| 6 | No 499 Ω on TXD; TXD 78 mm, 8 vias | R42 499 Ω; TXD 52 mm, 1 via | **Fixed** (U_TX stub pending fan-out) |
| 7 | Long EN/BOOT nets | CHIP_UP 90 -> 49 mm, IO9 74 -> 46 mm. U4 auto-reset is gone. | **Fixed** |
| 8 | USB D+ 27 mm / D- 13 mm, not a pair | Native USB to the ESP32: D+ **57 mm** (F/B, 2 vias), D- **59 mm** (F/In2, 2 vias). Lengths match within 2 mm, but the lines are on different layers and run through the JST/LED area. | **Worse**: see below |
| 9 | SW1/SW2 below the antenna | SW1 22 mm, SW2 31 mm from the chip antenna; antenna keep-out is clear on all layers | **Fixed** |

## New findings

| # | Finding | Why it matters | Proposed fix |
|---|---|---|---|
| 10 | **TVS far from the connectors.** The ESD current runs 10-30 mm across the board before the TVS clamps it. That is roughly 10-30 nH, or 100-300 V of extra overshoot on a 1 ns, 10 A ESD edge. | The 330 Ω series resistors (R14-R21) still protect the mux behind them, so this is not a failure. But the TVS is most effective at the connector, before the ESD current enters the board. | Move each SRV05-4 to its JST column (one TVS per 4 connectors, at the data pins). Put D10 at the V_IN terminals. Keep the pull-down row where it is. |
| 11 | **USB pair through the LED area.** D+ and D- are routed separately, about 58 mm each, from USB-C at the left edge to the ESP32 at the top right. They cross the JST block, where up to 26 A flows. | Full-speed USB (12 Mbit/s) tolerates the skew and the length. The risk is common-mode noise from the LED current, which shows up as USB drop-outs while the LEDs run at high brightness. | Route D+/D- as a pair on F.Cu over the In1 GND plane, with no layer changes, and keep it away from the JST pins. If that does not fit, route around the top edge under J1. |
| 12 | **CH lines got longer**: 13-47 mm -> 27-60 mm, with 2-6 vias, mostly on In2/B.Cu. The protection row sits between the mux and the JSTs. | At 800 kHz with 330 Ω sources there is no signal-integrity problem, and the new pull-downs hold unselected lines low. Longer lines on In2 make In2 more crowded (finding 4). | Moving the TVS to the JSTs (finding 10) shortens these lines again. |
| 13 | **RF and crystal**: RF_IN 1.3 mm, XTAL_N 2.2 mm, F.Cu over In1 GND. The antenna keep-out is clean. | Looks right, but cannot be judged until the fan-out is done. | Re-check after the hand routing: RF line straight with no via, crystal traces short and symmetric, with no signals under Y1 on any layer. |

## What is fine

- In1.Cu GND is still one solid piece (2931 mm²).
- GND stitching vias went from 172 to 205. F.Cu GND fragments went from 34 to
  22, B.Cu from 20 to 14.
- The +5V plane on In2 is still one connected piece.
- The level shifters (U6-U9) sit in one row, each 100 nF right at VCC.
- The LED current loop still stays in the left half of the board, away from
  the ESP32.

## Recommended before ordering (most important first)

1. Move C36 to J1 pins 2/4 (finding 2).
2. Move U11/U12 to the JST columns and D10 to V_IN (findings 5, 10, 12).
3. Route USB as a pair on F.Cu, away from the JSTs (findings 8, 11).
4. Finish the QFN fan-out, then re-check RF/crystal (finding 13).

The measurements listed in `docs/noise-analysis-v4.md` still apply once boards
exist. For this board, also measure RSSI against a WROOM-02 board, and check
that USB stays connected at full LED brightness.

## Update: fixes applied (2026-10-14)

Recommendations 1-3 were applied. Re-measured on the new layout:

| Item | Before | After |
|---|---|---|
| C36 (Pi bulk) to J1 pin 2/4 | 39 mm | **5 mm**. C36 is now 100 µF 1206 (6.3 V X5R); a 1210 did not fit between J1, the mounting hole and V_IN1. C16 moved next to it. |
| U11 TVS pin to JST data pin (CH1-4) | 10-31 mm | **7-12 mm** (U11 to the right of the J4-J7 column) |
| U12 TVS pin to JST data pin (CH5-8) | 9-31 mm | **8-22 mm** (U12 below the V_IN terminals). The terminals sit left of the J8-J11 column, so there is no closer spot. |
| D10 to V_IN | 28-35 mm | **19-30 mm**. The terminal block leaves no room next to it. |
| USB through the JST block | crosses it | **0 mm inside** |
| USB D+ / D- | 57 / 59 mm, 2 vias each | 55 / 68 mm, 6 / 8 vias, mostly F.Cu |
| CH1-CH8 length | 27-60 mm | 33-54 mm |
| GND stitching vias | 205 | 216 |
| +5V plane on In2 | 2288 mm² | 2380 mm², J1 pins 2/4 in the main piece |

The USB pair has to cross the ESP32 -> shifter -> mux -> 330 Ω -> JST
chain, which splits the board, so a few short B.Cu hops are unavoidable
without moving that chain. For full-speed USB, 13 mm of length mismatch
(about 80 ps) and the vias do not matter. Keeping the pair out of the LED
current area was the point.

C36 at 1206 has less effective capacitance at 5 V than the 1210 (X5R DC bias).
Expect roughly 30-40 µF effective. That is still useful close to the pins,
together with the PSU and copper.


## Update: QFN fan-out done (2026-10-15)

The board is fully routed (0 DRC errors, 0 unconnected). The parts around U1
were re-placed first, so this re-checks finding 13:

| Item | Result |
|---|---|
| RF_IN (pin 1 to C32/L3) | 2.3 mm, 0.33 mm, F.Cu over In1 GND, no vias |
| RF_LINE (L3 to L4) | 5.4 mm, 0.33 mm, no vias. The chain runs straight out of pin 1, so no IO escape crosses it on F.Cu |
| ANT (L4 to antenna) | 3.6 mm, widened from 0.2 to 0.33 mm |
| XTAL_P / XTAL_PX | 1.5 / 3.8 mm, F.Cu, no vias, through L2 |
| XTAL_N | 8.1 mm, F.Cu, no vias. Longer than ideal, because it goes around the Y1 GND pad. At 40 MHz this only adds a little stray capacitance; trim the load caps with the TX tone test |
| Other nets under Y1 | A1, IO2, IO9 and PI_BOOT on B.Cu/In2, below the In1 GND plane. One IO9 via sits next to the Y1 GND pad (pin 4), not at the XTAL pins |
| Decoupling to pin | C24 2.6, C25 1.5, C26 2.3, C27 2.8, C28 1.5, C29 1.7 mm |
| USB D+ / D- | 62 / 73 mm, 6 / 8 vias. Still out of the LED current area. The ~11 mm mismatch (about 70 ps) does not matter at full speed |
| QFN fan-out | 0.15 mm tracks with 0.15 mm clearance inside the `QFN_FAN` rule area (JLC allows 0.1 mm) |

All four recommendations are now done. What is left is measuring real boards:
RSSI against a WROOM-02 board, crystal frequency with the TX tone test, and
USB stability at full LED brightness.
