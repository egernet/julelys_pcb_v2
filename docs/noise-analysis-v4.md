# Noise analysis - v4 (feature/v4, commit 26fc348)

Date: 2026-10-14. The numbers come from the board geometry (track lengths,
layers, placement) plus estimates. Nothing was simulated. Re-run this analysis
on the test board (bare ESP32-C3 + native USB, `feature/v4-esp32-chip`) and
compare.

## Summary

At the board's signal speeds there are no critical signal-integrity problems:
LED data 800 kHz, SPI 2.5 MHz, UART 115 kbit/s. The real noise risks are the
**LED supply current** (up to 26 A) and the **LED outputs** that drive long
cables.

## Findings (most important first)

| # | Finding | Why it is a noise risk | Proposed fix |
|---|---|---|---|
| 1 | **No pull-downs on the LED outputs.** The 74HC4051 leaves the 7 unselected channels open. CH1-CH8 only have 330 Ω series resistors. | The data input of the first LED on an unselected string floats on a long cable. It picks up noise from the other strings and from the supply current, so the LEDs flicker or show wrong colours. This is the most likely visible fault. | 10-47 kΩ pull-down on each CH line at the JST connector (8 x 0603). |
| 2 | **The Pi's 5V shares the LED supply with no isolation.** C15 (470 µF) sits bottom-left, about 45 mm from J1. J1 itself only has C16 (100 nF). | When all LEDs step from dark to white, the current can jump up to 26 A. The supply and copper drop is seen directly by the Pi. A Pi 5 flags undervoltage below about 4.63 V. | Add 100-220 µF bulk right at J1 pins 2/4. Ramp or limit brightness in firmware. Set the PSU to 5.1-5.2 V. |
| 3 | **Copper for the full LED current.** Each 2 mm power track (1 oz, about 0.25 mΩ/mm) carries up to about 13 A for 4 strings. The In2 plane (0.5 oz by default at JLC) shares the load. | Full white gives about 100-150 mV from terminal to connector, plus the JST contacts. IPC estimates about 40-60 °C rise for 2 mm / 1 oz at 13 A, less with the plane in parallel. The GND drop also acts as noise for the logic. | Limit total current in firmware (for example 8-10 A). Order 1 oz inner layers. Feed long strings from both ends. |
| 4 | **Bottom-layer signals reference the +5V plane.** The stackup is B.Cu, prepreg, In2 (+5V). Long SPI/A0-A2/IO9 runs sit on the bottom (CLK 32 mm, CS 31 mm, A0 33 mm, IO9 49 mm). | Their return current flows in the +5V plane and has to cross to GND through the nearest capacitor. That plane also carries the noisy LED current, so the noise couples into these signals. The effect is moderate at these speeds. | Keep sensitive signals on top, over the In1 GND plane. Spread a few 100 nF +5V-to-GND caps over the board. |
| 5 | **No ESD/surge protection on the LED outputs or V_IN.** Only USB has ESD (D7). | Outdoor Christmas lights with metres of cable pick up ESD and transients. These hit the 74HC4051 directly, which is rated about ±2 kV HBM. V_IN has no reverse-polarity or TVS protection. | TVS arrays on CH1-8 (2 x 4-channel). TVS (and/or fuse) on V_IN. |
| 6 | **No 499 Ω series resistor on U0TXD** (Espressif recommends it). TXD is 78 mm long with 8 vias. | Fast edges on a long track radiate and cost some Wi-Fi sensitivity. | 499 Ω at the ESP32 TX pin. |
| 7 | **Long EN/BOOT nets.** CHIP_UP is 90 mm and IO9 is 74 mm, spread over buttons, auto-reset, Pi and test points. | They can pick up noise and cause false resets. The risk is low because C1/C6 (1 µF / 100 nF) sit 3-6 mm from the EN pin. | Keep the caps at the pin. Shorter on the test board, where U4 goes away. |
| 8 | **USB D+ is 27 mm with 2 vias, D- is 13 mm with none.** They are not routed as a pair. | USB full-speed tolerates this (about 0.1 ns skew). | Route as a differential pair. |
| 9 | **SW1/SW2 sit right below the antenna keep-out.** | Their metal frames can detune the antenna slightly. This only matters in dev-board mode. | Move them away on the test board. |

## What is fine

- R14-R21 (330 Ω) plus the mux on-resistance (about 70 Ω) give a source
  impedance of about 400 Ω. That slows the cable edges and damps reflections.
  Every on-board track is electrically short (λ/10 at about 100 MHz is about
  15 cm).
- The LED current loop (terminals -> JST -> LEDs -> return) stays in the left
  half of the board, away from the ESP32 and the mux.
- The AMS1117 has high PSRR at the LED PWM frequency (about 1 kHz). The ESP32
  decoupling is close: C3 at 2.3 mm, C2 at 3.0 mm.
- GND stitching: 172 GND vias and an unbroken In1 GND plane with no signal
  tracks in it.

## Stackup note

The board file's stackup says 0.1 mm prepreg. JLC's standard 4-layer stackup
(JLC04161H-7628) has **0.21 mm** prepreg between L1 and L2 (Dk about 4.4), with
a 1.065 mm core. Fix this before sizing any 50 Ω RF line. 50 Ω CPWG comes out
at about 0.32-0.34 mm wide with a 0.20-0.30 mm gap.

## Measurements once boards exist

1. **Flicker:** drive one string only and check whether the others flicker
   (finding 1).
2. **Pi supply:** step all strings from black to full white, scope 5V at J1,
   and run `vcgencmd get_throttled` on the Pi (finding 2).
3. **Heat:** run full white for 5 minutes and measure the power tracks near
   V_IN and the JSTs (finding 3).
