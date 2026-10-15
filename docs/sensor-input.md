# Sensor input on the LED outputs (test board)

Added 2026-10-15 on `feature/v4-esp32-chip`. The board can read up to 8
analog sensors through the same JST connectors (J4-J11) that drive the LED
strings. Lights stay the main job: the LED path is unchanged, and LED data
still comes from GPIO7 (RMT).

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

## Read sequence (firmware)

1. Finish sending the LED frame.
2. Drive IO1 high. U6's output goes high-impedance.
3. Set A0-A2 to the channel to read.
4. Wait about 0.5 ms for the divider and C40 to settle.
5. Read IO0 with ADC1 channel 0, 12 dB attenuation. Sensor voltage =
   ADC voltage x 2.5.
6. Drive IO1 low again before the next LED frame.

Eight channels take about 4 ms. At 26 fps there is about 38 ms between frames,
so reading all 8 between frames is fine. ADC1 works while Wi-Fi is on, so this
also works in stream mode.

## Limits

- **A channel is either a LED string or a sensor.** Reading a channel with a
  LED string attached just reads the string's idle data line.
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
