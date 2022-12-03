
# Sound - Power Noise

Power Noise is a fantasy sound chip designed by jvsTSX and [The Beesh-Spweesh](https://twitter.com/StinkerB06). Hexheld is the primary application for this chip, where it is embedded inside the HiveCraft SoC.

There are 5 channels of sound:
1. Noise
2. Noise
3. Noise
4. Slope
5. [PCM](https://en.wikipedia.org/wiki/Pulse-code_modulation)

Noise channels use a [linear feedback shift register (LFSR)](https://en.wikipedia.org/wiki/Linear-feedback_shift_register) with programmable taps as the sound source. The exclusive OR function can be disabled, allowing the shift register to hold an arbitrary binary wave table.
- The name "Power Noise" comes from this flexibility and that this is the primary sound generation method of the chip.

The slope channel generates timbres with an accumulator and slope offsets. The slope waveform is divided into 2 portions each with a programmable length, offset, and flags. An amplitude modulation feature exists for this channel which uses the amplitude levels of one or more noise channels, and works by ANDing together.

The PCM channel is an 8-bit [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter). The CPU or DMA controller must supply the samples.

There are external audio inputs which are mixed along with the on-board sound channels. These are signals coming from the inserted Hexridge.

The output is stereo. While channels are mono, volume controls are provided separately for the left and right sides, allowing for stereo balancing of channels. Apart from this design choice, there are 2 external audio inputs: one for left and one for right.



## Register Map

There are 32 8-bit registers which control the aspects of the sound chip. They occupy the I/O map:

| Port Number | Name | Meaning |
| :-: | :- | :- |
| `$20` | `ACR`(W) | Audio Control Register |
| `$21` | `NO11`(W) | Noise 1 Frequency (Low) |
| `$22` | `NO12`(W) | Noise 1 Frequency (High) |
| `$23` | `NO13`(R/W) | Noise 1 Shift Register (Low) |
| `$24` | `NO14`(R/W) | Noise 1 Shift Register (High) |
| `$25` | `NO15`(W) | Noise 1 Tap Locations |
| `$26` | `NO16`(W) | Noise 1 Volumes |
| `$27` | `NO17`(W) | Noise 1 Control |
| `$28` | `LVL`(W) | Volume Level Settings |
| `$29` | `NO21`(W) | Noise 2 Frequency (Low) |
| `$2A` | `NO22`(W) | Noise 2 Frequency (High) |
| `$2B` | `NO23`(R/W) | Noise 2 Shift Register (Low) |
| `$2C` | `NO24`(R/W) | Noise 2 Shift Register (High) |
| `$2D` | `NO25`(W) | Noise 2 Tap Locations |
| `$2E` | `NO26`(W) | Noise 2 Volumes |
| `$2F` | `NO27`(W) | Noise 2 Control |
| `$30` | `DAC`(W) | PCM Sample Input |
| `$31` | `NO31`(W) | Noise 3 Frequency (Low) |
| `$32` | `NO32`(W) | Noise 3 Frequency (High) |
| `$33` | `NO33`(R/W) | Noise 3 Shift Register (Low) |
| `$34` | `NO34`(R/W) | Noise 3 Shift Register (High) |
| `$35` | `NO35`(W) | Noise 3 Tap Locations |
| `$36` | `NO36`(W) | Noise 3 Volumes |
| `$37` | `NO37`(W) | Noise 3 Control |
| `$38` | `SCNT`(R/W) | Slope Accumulator |
| `$39` | `SLO1`(W) | Slope Frequency (Low) |
| `$3A` | `SLO2`(W) | Slope Frequency (High) |
| `$3B` | `SLO3`(W) | Slope Portion A Duration |
| `$3C` | `SLO4`(W) | Slope Portion B Duration |
| `$3D` | `SLO5`(W) | Slope Offsets |
| `$3E` | `SLO6`(W) | Slope Volumes |
| `$3F` | `SLO7`(W) | Slope Control |

Registers marked with (R/W) can be read and written. Those that are marked with (W) are write-only and are read as the value `$FF`.

Since the DMA controller only allows ports `$00` to `$3F` to be used as the destinations of a transfer to I/O, any Power Noise register can be targeted.



## Audio Control

In order for sound to be output, flags in `ACR` must be enabled:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Master Enable | | External | PCM | Slope | Noise 3 | Noise 2 | Noise 1 |

Bits 0 to 4 correspond with the 5 sound channels. If a bit transitions from clear to set, the channel will start working and be audible. Clearing a bit will freeze the channel and cause it to always output an amplitude of `0`. The registers of the channel can still be accessed while frozen.

If bit 5 is set, the external audio signals coming from the Hexridge will be mixed into the audio outputs. If clear, external audio is muted.

Bit 6 has no function.

Bit 7 must be set for any sound to be output. If cleared, all mixers—digital and analog—are turned off, and so neither the sound channels nor the external audio inputs will be present in the outputs. The sound chip is not fully shut down—channels are still functional in the background as long as the other bits are set.

Disabling parts reduces power consumption.



## Frequencies

The frequency to run a noise channel or the slope channel at is defined as a 16-bit value. The lower bytes are written into the `NOx1` and `SLO1` registers, and the upper bytes are written into the `NOx2` and `SLO2` registers.

The value serves as the reload value for a 16-bit up-counter. The up-counter runs at a frequency of 8 MHz (DIV1), which will get divided by 65,536 minus the value. The noise or slope algorithm will be performed at this quotient frequency.

A wave generated by a channel has some period length in units of samples. The frequency specifies the sample frequency, not the frequency of the period. For example, the observed frequency of a 16-sample wave will be the specified frequency divided by 16.


### Frequency Conversion Formulas

```
hz    = 8000000 / (65536 - value)
value = 65536 - (8000000 / hz)
```



## Volumes

Volume represents how loud a sound source is. All 5 sound channels have volume controls associated with them. It is also possible to adjust the volume of the overall mixes of those channels.

As the sound output is stereo, separate volume controls are provided for the left and right sides for each channel.


### Noise & Slope

Volume is specified for these channels in the `NOx6` and `SLO6` registers. They all have the same format:

| Bits 7-4 | Bits 3-0 |
| :-: | :-: |
| Left Volume | Right Volume |

Volume level scales linearly with value. The value `$0` represents silence, and the value `$F` represents the loudest volume.

For the slope channel, the volume value is a 4-bit AND mask that is applied to the 4-bit amplitude value. To ensure a somewhat-proper reduction of volume, the amplitude value is shifted a certain number of bits to the right before being ANDed, depending on the 4-bit volume value:

- `$0` to `$1` - 3 bits
- `$2` to `$3` - 2 bits
- `$4` to `$7` - 1 bit
- `$8` to `$F` - No shift


### PCM & Overall Audio

The volume of the PCM sound channel, as well as the overall audio volume, are specified in the `LVL` register:

| Bit 7 | Bits 6-4 | Bits 3-2 | Bits 1-0 |
| :-: | :-: | :-: | :-: |
| | Overal Audio | PCM Left | PCM Right |

PCM volume level does not scale linearly with value. When incorporating the 8-bit `DAC` amplitude in the output, it is logically-shifted a certain number of bits to the right depending on the 2-bit volume value:

| Value | Shift Amount | Volume |
| :-: | :-: | :-: |
| `0` | | 0% (output always `$00`) |
| `1` | 2 | 25% |
| `2` | 1 | 50% |
| `3` | 0 | 100% |

Overall audio volume level scales linearly with value. The value `0` represents silence, and the value `7` represents the loudest volume. This setting applies to both the left and right sides, and affects only the internal sound channels.

The loudness of the PCM channel at full volume is approximately equivalent to the 3 noise channels all at loudest volume.



## Sound Channels 1,2,3 (Noise)

These channels generate binary waveforms using a 16-bit LFSR. It can have 1 or 2 taps, each with a specifiable bit position.
- A binary wave is made out of 2 amplitudes: *low* and *high*.

The wave is a pseudo-random sequence of binary samples. The length of the sequence is a result of the selected tap locations and the shift register state. Longer sequences result in sounds that more resemble noise and less of an audible pitch.

An arbitrary binary wave table can be output by disabling Tap B. The wave table may be any length from 1 to 16 samples, where the length is 1 plus the bit position of Tap A.

The shift register values can be read and written. The `NOx3` registers are the lower bytes, and the `NOx4` registers are the upper bytes, of shift register contents.


### Tap Locations

The `NOx5` registers specify the bit positions of the LFSR taps. They all have the same format:

| Bits 7-4 | Bits 3-0 |
| :-: | :-: |
| Tap A Bit Position | Tap B Bit Position |

A value of `$0` specifies the least-significant bit (`$0001`), and a value of `$F` specifies the most-significant bit (`$8000`), of the shift register.


### Control

The `NOx7` registers manage the operation of the noise channels. They all have the same format:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Run | | | | | | AM | Enable Tap B |

Bit 0 specifies how many taps are used. If clear, only Tap A is used and the pseudo-random bit is the bit of the LFSR specified by the tap's position setting. If set, the pseudo-random bit is the exclusive OR of the bits of the LFSR specified by both tap position settings.

If bit 1 is set, the slope channel's amplitude modulation feature is enabled to use this noise channel's binary amplitude.

Bits 2 to 6 have no function.

If bit 7 is clear, the frequency generator is paused, preventing the noise algorithm from working and the shift register from automatically updating.


### Noise Algorithm and Output Procedure

Every time the frequency phase accumulator wraps around, a pseudo-random bit is produced according to the specified tap locations and whether Tap B is enabled. The shift register contents is shifted left by one bit position, and the new bit 0 is the pseudo-random bit. The state of bit 15 is lost.

Bit 15 of the shift register contains the binary amplitude that is incorporated in the output as well as the amplitude modulation feature. If clear, the amplitude is low. If set, the amplitude is high.

A noise channel's output value is 4-bit. If the binary amplitude is low, `$0` is output. If the amplitude is high, the 4-bit volume value is output.



## Sound Channel 4 (Slope)

This channel consists of a 7-bit accumulator, which is modified by constant amounts to ramp the amplitude up and/or down.

A waveform generated by this channel is divided into 2 portions: A and B. Both portions each have the following attributes:
- Length (1 to 256 samples)
- Offset (0 to 15)
- Up/Down Flag
- Reset Flag
- Clip Flag

Portion A is the one to be played first, then Portion B comes after. Both portions are alternated between to produce a looping wave.


### Accumulator

The `SCNT` register is the accumulator value. It can be read and written.

Only the lower 7 bits are significant. The upper-most bit is unused and is always clear when read.

The upper 4 bits of the accumulator are used as the slope amplitude value. In essence, the accumulator is a 4.3 unsigned [fixed-point](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) value which gets rounded down.


### Portion Length

The lengths of the portions are defined as 8-bit values. The length of Portion A is specified in the `SLO3` register, and the length of Portion B is specified in the `SLO4` register. Add 1 to these values for the actual lengths in samples.

The total duration of a wave is the durations of both portions added together. Note that depending on slope flags, there may be an "attack" period before the repeating wave, or the actual duration of the wave may be longer than this sum.


### Offsets

An offset value represents how much is added to or subtracted from the accumulator on every sample of a portion. The offsets are specified in the `SLO5` register:

| Bits 7-4 | Bits 3-0 |
| :-: | :-: |
| Portion A Offset | Portion B Offset |

The maximum offset is `15`. If the accumulator value is interpreted as fixed-point, the maximum considered offset is `1.875`.


### Flags

Flags specify how to modify the accumulator on every sample of a portion. They are contained in the lower 6 bits of the `SLO7` register.

Bits 0 and 1 specify the modification direction of Portions B and A, respectively. If clear, offset is added to the accumulator. If set, offset is subtracted from the accumulator.

Bits 2 and 3 specify that the accumulator should reset on the first sample of Portions B and A, respectively. If clear, the accumulator will be modified by an offset as normal. If set, the accumulator will instead be directly set to `$00` for addition or `$7F` for subtraction.

Bits 4 and 5 specify how to handle accumulator overflows/underflows in Portions B and A, respectively. If clear, the accumulator value will wrap around and continue on the other end of the range. If set, the accumulator value will be clipped at `$7F` on overflow or `$00` on underflow.


### Control

The `SLO7` register manages the operation of the slope channel:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Run | Reset | A Clip | B Clip | A Reset | B Reset | A Up/Down | B Up/Down |

Bits 0 to 5 specify the slope flags. See above for information.

If bit 6 is set in the write, the slope wave generation resets to the first sample of Portion A. This reset is automatically performed on Hexheld system reset.

If bit 7 is clear, the frequency generator is paused, preventing the slope algorithms from working and the slope accumulator from automatically updating.


### Amplitude Modulation (AM)

When this feature is enabled from the `NOx7` registers, the slope amplitude level is effectively ANDed with the binary amplitudes of one or more noise channels, enabling complex sound effects and timbres to be generated.

If the binary amplitude of the noise channel is high, the slope output level is unmodified. However, if low, the slope output level is forced to be `$0`.

If AM is enabled from multiple noise channels, their binary amplitudes will be ANDed together. As such, if either of the amplitudes is low, the slope output level is forced to be `$0`.


### Output Procedure

Every time the frequency phase accumulator wraps around, the slope accumulator is modified according to the algorithms.

The upper 4 bits of the 7-bit slope accumulator value are used as the 4-bit wave amplitude value. This amplitude value goes through amplitude modulation if enabled, then goes through the shifting and masking logic for volume reduction.



## Sound Channel 5 (PCM)

This channel outputs an arbitrary 8-bit unsigned amplitude written to the `DAC` register. The sound chip does not automatically modify `DAC` to produce a waveform, so samples must be provided by the CPU or DMA controller.

The 8-bit amplitude is shifted right by a certain number of bits according to the 2-bit volume setting in `LVL` to produce the actual unsigned output level value for the channel.


### PCM Playback with DMA

The Hexheld DMA controller is configurable enough to be able to deliver samples to `DAC` at the reload frequency of general-purpose timer A.

The size of a sound can vary from 1 to 65,536 samples (64 KB), according to `DxCNT`. After playing the last sample, it can either hang there or loop back to the first sample.

As the Hexheld memory address bus is 24 bits wide, up to 16 MB of PCM sound data can be addressed by pointing `DxSA` to the location of the first sample. Further samples follow consecutively at higher addresses.



## Output Procedure

The noise and slope channels are mixed together digitally—by adding their 4-bit output values together—to produce a 6-bit unsigned linear mix. The highest possible sum for this mix is `60`. The PCM channel is mixed with these channels in an analog fashion. This analog mix then gets attenuated according to the 3-bit overall volume setting. Finally, external audio comes in.

NOTE: As the output is stereo, the same mixing algorithms apply for both sides, only that each side incorporates its own per-channel volume values.



## Reset

When the Hexheld system is reset, all Power Noise registers are initialized with `$00`.

Since the slope wave generation resets as well, it may be thought that `SLO7` is initialized with `$40`.
