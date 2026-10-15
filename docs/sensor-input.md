# Sensor input on the LED outputs (test board)

Added 2026-10-15 on `feature/v4-esp32-chip`. **For dev-board use only**, when
the board is not driving Julelys lights. Then the 8 JST connectors (J4-J11)
can read up to 8 analog sensors instead.

In Julelys mode nothing changes: R51 keeps the LED data shifter U6 enabled,
IO1 is never driven, and the Julelys firmware does not need to know about
the sensor parts. The only cost is a 25k load (R52+R53) on the LED data line,
which the shifter drives without trouble.

## How it works

The 74HC4051 (U2) is an analog multiplexer, so it passes signals both ways.
Today the ESP32 sends LED data through the level shifter U6 into the mux
common pin (`/Data`), and A0-A2 pick the channel. To read a sensor, the same
path runs backwards:

```
JST data pin -> 330R (R14-R21) -> U2 channel -> /Data -> R52 15k -> IO0 (ADC1_CH0)
                                                                  -> R53 10k -> GND
                                                                  -> C40 10nF -> GND
U6 /OE (pin 1) <- IO1, with R51 10k pull-down (U6 on at boot)
```

| Part | Value | Job |
|---|---|---|
| R51 | 10k | Pull-down on U6 /OE, so the LED data shifter is on at boot and whenever IO1 is not driven |
| R52 / R53 | 15k / 10k | Divider: 0-5 V on the JST data pin becomes 0-2 V at the ADC |
| C40 | 10 nF | ADC filter (settles in about 0.3 ms) |

## Read sequence (dev firmware)

1. At start-up, drive IO1 high. U6's output goes high-impedance and stays
   that way; no LED data is sent in this mode.
2. Set A0-A2 to the channel to read.
3. Wait about 0.5 ms for the divider and C40 to settle.
4. Read IO0 with ADC1 channel 0, 12 dB attenuation. Sensor voltage =
   ADC voltage x 2.5.
5. Repeat from step 2 for the next channel. All 8 take about 4 ms.

ADC1 works while Wi-Fi is on, so the readings can be streamed over the
network.

## Limits

- **Do not connect LED strings in sensor mode.** The JST pins carry the
  board's +5V, so only sensors belong on them then.
- **Sensor output 0-5 V.** The 10k pull-down on every channel (R43-R50) loads
  the sensor, so it needs a low-impedance voltage output (most powered analog
  sensors are). Resistive sensors (NTC, LDR) work as the upper half of a
  divider: connect them between +5V and data on the JST. The lower half is
  the 10k pull-down in parallel with R52+R53, about 7.2k in total.
- **Supply on the JST is +5V** (the LED supply). 3.3 V sensors need their
  own regulator.
- **Accuracy** is that of the ESP32-C3 ADC, about +-2 % after calibration.
  The divider resistors are 1 %.
- **Pi fitted:** IO0/IO1 also go to J1 pins 37/36 (Pi GPIO26/16) through 1k.
  Leave those Pi pins as inputs, or the Pi fights the ADC/OE lines.
