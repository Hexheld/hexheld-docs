# Hexheld Power Consumption Estimate

This report is a synthesis of research and ballpark estimates performed by Kagamiin~ to determine/guess nominal power consumption and typical expected battery life for the Hexheld.

The majority of the analysis will go into the backlight, since it is the most power-consuming component of the system.

## Backlight facts

The backlight in the Hexheld consists in 3 parallel arrays of the following types of LEDs:

- Red and green: GaP (gallium phosphide) diodes
- Blue: InGaN (indium gallium nitride) diodes

Each array is driven by its own constant current source, which gets modulated via PWM. The drive current is calibrated such that when the red channel is driven at half the duty cycle of the green and blue channels, the output light matches the CIE D50 Illuminant standard - which should match the color of sunlight at noon.

Some research shows the following facts:

- Blue LEDs from 1995 were roughly 10% as efficient as modern equivalents (which have a quantum efficiency of roughly 60%)
- Green and red LEDs from 1995 were, surprisingly, around 45% as efficient as modern equivalents (modern-day red LEDs have quantum efficiency of roughly 45%)
- In terms of gallium phosphide diodes and raw spectral flux, green LEDs are less efficient than red LEDs. From my research I'll just assume they're 50% as efficient for my purposes.

## Backlight drive currents

From all the data gathered, my guesstimates for the calibration currents in terms of modern-day LED efficiencies are:

Red: 75 mA
Green: 135 mA
Blue: 40 mA

Note that the scale is completely arbitrary.

By applying the efficiency difference, this translates to the following current draws using state-of-the-art 1995 LED tech:

- Red: 166 mA
- Green: 300 mA
- Blue: 400 mA

Now, since we are using constant current sources to drive the LEDs, I'm going to assume power losses at the driver are negligible (which is practically so with proper MOSFET current source drivers). With typical voltages of 2.0, 2.1 and 3.6 for each of the LEDs we get the following power draw:

- Red: 333 mW
- Green: 630 mW
- Blue: 1440 mW (yikes!)

So, with the three arrays being driven at 100% duty cycle, the backlight would be using 2403 mW, so basically 2.4 watts. That's, uhh, a lot. That's for full drive strength though, which produces a reddish-tinted output.

By halving the duty cycle of the red channel to get calibrated white light output, we get a power consumption at max brightness of 2236 mW, which is just slightly better. Ouch.

(fun fact: a compact fluorescent light could theoretically use less power than that to produce the same amount of white light, but efficiently boosting a few volts from the AAs up to over 100 volts to drive the CFL is a whole another story lol not to mention the space taken up by the bulb and optics)

## The HiveCraft SoC

The HiveCraft SoC is a rather powerful machine when compared to the hardware of its main competitor, the Game Boy. Boasting an 8 MHz CPU with a 3-stage pipelined 16-bit CISC architecture, it can run circles around the competition. The PPU doesn't stay much behind either, with a bitplane bandwidth of 4 MB/s, support for multiple layers, can display fucktons of sprites per scanline etc etc. Oh and let's not forget about its custom PowerNoise sound chip with unique sound generation. Oh, and those two DMA controllers, those are beefy. What does it take to run this thing?

Good question. The original DMG Game Boy had a nominal power draw of roughly 120 mA at 5 volts (in practice, usually about half of that depending on the game). The HiveCraft, being a more modern design, is based on 3.3V CMOS technology rather than 5V, and is built on a smaller, more efficient CMOS lithography process.

Given all those facts, my guess is 300 mA @ 3.3V for the HiveCraft. Inefficiencies from power regulation apart, that's pretty much 1 W. That's the nominal power requirement, however - actual power draw will be lower depending on the game code, CPU speed setting and PPU features being used, as well as use of DMA transfers and such. Given that the HexHeld has more advanced power management features than the Game Boy, I'd guesstimate that the average power draw during gameplay would range from a quarter to half of that.

## Powering the system

With the HiveCraft SoC being a 3.3 volt part, powering it from two AA batteries would be a breeze. The voltage regulator would do a good and efficient work of stabilizing the unregulated 1.6-3.3 V of the alkaline batteries into a stable 3.3 volt source.

There's just one problem: the backlight. Especially those damn blue LEDs. Assuming the operating voltage of 3.6 volts and with a small safety margin for decoupling, we'd need at least a stable 4 volt supply to drive that backlight. And damn, is it gonna draw power. The boost regulator is gonna have to work hard to boost an expected 1.6-2.4 volts up to 4 volts. Just from the blue LEDs alone we might be looking at current draws of 800 mA average, peaking at over 1200 mA near the end of the batteries' poor, short lives. We might not even get half an hour out of it before the batteries go poof.

So we have to bump up our requirement to four AA batteries. This already gives us enough operating margin that a boost regulator isn't even needed for the HiveCraft SoC - an efficient buck regulator will do just fine. The same goes for the red and green channels of the backlight. The only remaining need for a boost converter lies in the blue channel of the backlight, and in practice it'll only need to operate towards the end of the battery's capacity. That's much better.

I'll assume a constant 95% conversion efficiency for the buck regulator that provides the 3.3V rail for the HiveCraft. At the nominal power requirement, that'd be 1053 mW. For real-world estimates I'll just assume a range of 280-550 mW. As for the backlight, I'll assume 100% efficiency for the red and green channels since their current source is powered via the unregulated battery output, and I'll assume 97% average efficiency for the blue channel since the boost regulator will only be active for a relatively small portion of runtime, yielding an average power draw of 1485 mW at 100% duty cycle.

## The results so far

To obtain the expected battery life, I'll be referencing a datasheet for Energizer alkaline AA batteries (standard type), and measuring values in the constant power to effective capacity graph - using my favorite swiss army knife for messing with anything visual (GIMP).

### Backlight off

- Nominal power requirement: 1053 mW (SoC only)
  - Power draw per cell: 263 mW
  - Attainable capacity per cell: ~2000 mWh
  - Minimum battery life (one continuous run): **~7h 36m**
  - Guessed typical battery life (2h-long sessions, 10h-long recovery): **~8h 45m**

- Typical power requirement (high estimate): 550 mW (SoC only)
  - Power draw per cell: 138 mW
  - Attainable capacity per cell: ~2720 mWh
  - Minimum battery life (one continuous run): **~19h 43m**
  - Guessed typical battery life (2h-long sessions, 10h-long recovery): **~20h 30m**

- Typical power draw requirement (low estimate): 280 mW (SoC only)
  - Power draw per cell: 70 mW
  - Attainable capacity per cell: ~3080 mWh
  - Minimum battery life (one continuous run): **~44h 0m**
  - Guessed typical battery life (2h-long sessions, 10h-long recovery): **~44h 30m**

### Backlight on

The duty cycles are specified in units of 1/32. Unless otherwise specified, the power draw from the SoC is taken from the high estimate.

Minimum battery life represents a full continuous run; guessed typical battery life represents 2h-long sessions with 10h-long recovery inbetween.

| Scenario                                    | Backlight setting |          Average power draw          | Power per cell | Attainable capacity per cell | Time (minimum) | (guessed typical) |
|:--------------------------------------------|:-----------------:|:------------------------------------:|:--------------:|:----------------------------:|:--------------:|:-----------------:|
| White, full bright                          |      16-32-32     | 166 + 630 + 1485 + 550 = **2831 mW** |     708 mW     |          ~1170 mWh           |  **~1h 39m**   |     _(yikes)_     |
| Avg all combinations, "full" bright         |      16-16-16     | 166 + 315 + 743 + 550 = **1774 mW**  |     444 mW     |          ~1570 mWh           |  **~3h 32m**   |    **~3h 50m**    |
| White, 1/4 bright                           |       4-8-8       |  42 + 158 + 371 + 550 = **1121 mW**  |     280 mW     |          ~1923 mWh           |  **~6h 52m**   |    **~8h 0m**     |
| Avg all combinations, "1/4" bright          |       4-4-4       |   42 + 79 + 186 + 550 = **857 mW**   |     214 mW     |          ~2174 mWh           |  **~10h 10m**  |   **~11h 15m**    |
| Avg all comb, "1/4" bright, low SoC pwr est |       4-4-4       |   42 + 79 + 186 + 280 = **587 mW**   |     147 mW     |          ~2562 mWh           |  **~17h 26m**  |   **~18h 15m**    |

Okay, so those full brightness runtimes are clearly awful. It's clear that at whatever calibration current I picked (completely at random, mind you), the full brightness output should not be used continuously, being reserved only for short special effects. But still, even without being sure of how bright the backlight will be, let's be honest - 1/4 brightness is still too inefficient.

If we look at our power draw for each backlight channel, you can clearly see how the ineficciencies of the green and blue channels are such a pain. Let's see what power-saving benefits we can get if we orange-shift our output then. I don't feel like doing CIE xy-chromaticity calculations, and besides the backlight PWM is only 32-steps, so I'll just propose two arbitrary "power saving" modes that work by multiplying the colors by some constants.

- Mode 1: 100% (4/4), 75%   (3/4), 50% (2/4)

| Scenario                                    | Backlight setting |          Average power draw          | Power per cell | Attainable capacity per cell | Time (minimum) | Improvement | (guessed typical) |
|:--------------------------------------------|:-----------------:|:------------------------------------:|:--------------:|:----------------------------:|:--------------:|:-----------:|:-----------------:|
| White, full bright                          |      16-24-16     | 166 + 473 + 743 + 550 = **1932 mW**  |     483 mW     |          ~1511 mWh           |   **~3h 8m**   |    +90%     |    **~3h 30m**    |
| Avg all combinations, "full" bright         |      16-12-8      | 166 + 236 + 371 + 550 = **1323 mW**  |     331 mW     |          ~1800 mWh           |  **~5h 26m**   |    +54%     |    **~6h 45m**    |
| White, 1/4 bright                           |       4-6-4       |  42 + 118 + 186 + 550 = **896 mW**   |     224 mW     |          ~2125 mWh           |  **~9h 29m**   |    +38%     |   **~10h 30m**    |
| Avg all combinations, "1/4" bright          |       4-3-2       |   42 + 59 + 93 + 550 = **744 mW**    |     186 mW     |          ~2327 mWh           |  **~12h 31m**  |    +23%     |   **~13h 20m**    |
| Avg all comb, "1/4" bright, low SoC pwr est |       4-3-2       |   42 + 59 + 93 + 280 = **474 mW**    |     119 mW     |          ~2765 mWh           |  **~23h 14m**  |    +33%     |    **~24h 0m**    |

Now that's better. And do note that we're talking about linear color mixing, so this power-saving mode will only introduce a slight orange tinge. Let's try the next one though:

- Mode 2: 100% (8/8), 62.5% (5/8), 25% (2/8)
  - NOTE: for the "average of all combinations" scenario, since we're averaging the output light over time, the average value can go down to non-integer values.

| Scenario                                    | Backlight setting |          Average power draw          | Power per cell | Attainable capacity per cell | Time (minimum) | Improvement | (guessed typical) |
|:--------------------------------------------|:-----------------:|:------------------------------------:|:--------------:|:----------------------------:|:--------------:|:-----------:|:-----------------:|
| White, full bright                          |      16-20-8      | 166 + 394 + 371 + 550 = **1481 mW**  |     370 mW     |          ~1710 mWh           |  **~4h 37m**   |    +180%    |    **~5h 10m**    |
| Avg all combinations, "full" bright         |      16-10-4      | 166 + 197 + 186 + 550 = **1099 mW**  |     275 mW     |          ~1950 mWh           |   **~7h 5m**   |    +100%    |    **~8h 20m**    |
| White, 1/4 bright                           |       4-5-2       |   42 + 98 + 93 + 550 = **783 mW**    |     196 mW     |          ~2272 mWh           |  **~11h 35m**  |    +69%     |   **~12h 30m**    |
| Avg all combinations, "1/4" bright          |      4-2.5-1      |   42 + 49 + 46 + 550 = **687 mW**    |     172 mW     |          ~2410 mWh           |  **~14h 1m**   |    +38%     |   **~14h 50m**    |
| Avg all comb, "1/4" bright, low SoC pwr est |      4-2.5-1      |   42 + 49 + 46 + 280 = **417 mW**    |     105 mW     |          ~2858 mWh           |  **~27h 13m**  |    +56%     |   **~27h 45m**    |

Now look at that time improvement! You can definitely see how this is roughly equivalent to calibrating the LEDs at the same base current. It does distort the colors a lot more, though.

_OOC: applied to a color image, it really reminds me of those "blue light reduction filters" that are becoming more common in phones and computers_
