
# Sound - Power Noise

Power Noise is a fantasy sound chip designed by jvsTSX and [The Beesh-Spweesh](https://twitter.com/StinkerB06). Hexheld is the primary application for this chip, and is embedded inside the HiveCraft SoC.

There are 5 channels of sound:
1. Noise
2. Noise
3. Noise
4. Slope
5. [PCM](https://en.wikipedia.org/wiki/Pulse-code_modulation)

Noise channels use a [linear feedback shift register (LFSR)](https://en.wikipedia.org/wiki/Linear-feedback_shift_register) with programmable taps as the sound source. The exclusive OR function can be disabled, allowing the shift register to hold an arbitrary binary wave table.
- The name "Power Noise" comes from this flexibility and that this is the primary sound generation method of the chip.

The slope channel generates timbres with an accumulator and slope offsets. The slope waveform is divided into 2 portions each with a programmable length, offset, and flags. An amplitude modulation feature exists for the noise channels which uses the slope accumulator level.

The PCM channel is an 8-bit [DAC](https://en.wikipedia.org/wiki/Digital-to-analog_converter). The CPU or DMA controller must supply the samples.

There are external audio inputs which are mixed along with the on-board sound channels. These are signals coming from the inserted Hexridge.

The output is stereo. While channels are mono, volume controls are provided separately for the left and right sides, allowing for stereo balancing of channels. Apart from this design choice, there are 2 external audio inputs: one for left and one for right.



## Register Map

There are 32 8-bit registers which control the aspects of the sound chip. They occupy the I/O map:

| Port Number | Name | Meaning |
| :-: | :- | :- |
| `$20` | `ACR` | Audio Control Register |
| `$21` | `NO11` | Noise 1 Fine-Tune |
| `$22` | `NO12` | Noise 1 Octave |
| `$23` | `NO13`(R) | Noise 1 Shift Register (Low) |
| `$24` | `NO14`(R) | Noise 1 Shift Register (High) |
| `$25` | `NO15` | Noise 1 Tap Locations |
| `$26` | `NO16` | Noise 1 Volumes |
| `$27` | `NO17` | Noise 1 Control |
| `$28` | `LVL` | External Audio/PCM Volumes |
| `$29` | `NO21` | Noise 2 Fine-Tune |
| `$2A` | `NO22` | Noise 2 Octave |
| `$2B` | `NO23`(R) | Noise 2 Shift Register (Low) |
| `$2C` | `NO24`(R) | Noise 2 Shift Register (High) |
| `$2D` | `NO25` | Noise 2 Tap Locations |
| `$2E` | `NO26` | Noise 2 Volumes |
| `$2F` | `NO27` | Noise 2 Control |
| `$30` | `DAC` | PCM Sample Input |
| `$31` | `NO31` | Noise 3 Fine-Tune |
| `$32` | `NO32` | Noise 3 Octave |
| `$33` | `NO33`(R) | Noise 3 Shift Register (Low) |
| `$34` | `NO34`(R) | Noise 3 Shift Register (High) |
| `$35` | `NO35` | Noise 3 Tap Locations |
| `$36` | `NO36` | Noise 3 Volumes |
| `$37` | `NO37` | Noise 3 Control |
| `$38` | `SCNT`(R) | Slope Accumulator |
| `$39` | `SLO1` | Slope Fine-Tune |
| `$3A` | `SLO2` | Slope Octave |
| `$3B` | `SLO3` | Slope Portion A Duration |
| `$3C` | `SLO4` | Slope Portion B Duration |
| `$3D` | `SLO5` | Slope Offsets |
| `$3E` | `SLO6` | Slope Volumes |
| `$3F` | `SLO7` | Slope Control |

Registers marked with (R) can be read and written. The registers that are not marked are write-only and are read as the value `$FF`.

Since the DMA controller only allows ports `$00` to `$3F` to be used as the destinations of a transfer to I/O, any Power Noise register can be targeted.



## Audio Control

In order for sound to be output, flags in `ACR` must be enabled:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Mixers | | | PCM | Slope | Noise 3 | Noise 2 | Noise 1 |

Bits 0 to 4 correspond with the 5 sound channels. If a bit transitions from clear to set, the channel will start working and connect to the mixers. Clearing a bit will freeze the channel and disconnect it from the mixers. The registers of the channel can still be accessed while frozen.

Bits 5 and 6 have no function.

Bit 7 must be set for any sound to be output. If clear, the mixers are turned off, and neither the sound channels nor the external audio inputs will be present in the outputs. The sound chip is not fully shut down—channels are still functional in the background as long as the other bits are set.

Disabling parts reduces power consumption.



## Frequencies

The frequency to run a noise channel or the slope channel at is defined using 2 values: an octave setting, and a fine-tune value.

The octave setting is 3-bit, specified in the `NOx2` and `SLO2` registers. The upper 5 bits of these registers are unused. The setting specifies a base clock frequency, as follows:

| Octave Setting | Frequency | DIV | Range (Hz) |
| :-: | :-: | :-: | :-: |
| `0` | 2 MHz | 4 | 31,250 - 62,378 |
| `1` | 1 MHz | 8 | 15,625 - 31,189 |
| `2` | 500 KHz | 16 | 7,813 - 15,594 |
| `3` | 250 KHz | 32 | 3,906 - 7,797 |
| `4` | 125 KHz | 64 | 1,953 - 3,899 |
| `5` | 62.5 KHz | 128 | 977 - 1,949 |
| `6` | 31.25 KHz | 256 | 488 - 975 |
| `7` | 15.625 KHz | 512 | 244 - 487 |

The fine-tune value is 8-bit unsigned, specified in the `NOx1` and `SLO1` registers. The value covers the frequency range of an octave.


### Frequency Generation Algorithm

Each channel internally has a 14-bit *phase accumulator* register. At the frequency specified by the octave clock setting, 256 plus the fine-tune value is added to the accumulator.

When the accumulator value wraps around and continues up from `$0000`, the noise or slope algorithm will be performed once. Specifying a higher fine-tune value will cause the accumulator to wrap around more quickly, causing a higher frequency to be produced.

A pair of octave and fine-tune values may be converted into a real frequency with the following expression:
```
hz = clock_frequency[octave] / (16384 / (256 + finetune))
```

A wave generated by a channel has some period length in units of samples. `hz` is the sample frequency, not the frequency of the period. For example, the observed frequency of a 16-sample wave will be `hz` divided by 16. Thus, the frequency of such wave may range from 15.26 Hz to 3.9 KHz.



## Volumes

Volume represents how loud a sound source is. All sound channels as well as the external audio inputs have volume controls associated with them.

As the sound output is stereo, separate volume controls are provided for the left and right sides.


### Noise & Slope

Volume is specified for these channels in the `NOx6` and `SLO6` registers. They all have the same format:

| Bits 7-4 | Bits 3-0 |
| :-: | :-: |
| Left Volume | Right Volume |

Volume level scales linearly with value. The value `$0` represents silence, and the value `$F` represents the loudest volume.


### PCM & External Audio

The volumes of these sound sources are specified in the `LVL` register:

| Bits 7-6 | Bits 5-4 | Bits 3-2 | Bits 1-0 |
| :-: | :-: | :-: | :-: |
| External Left | External Right | PCM Left | PCM Right |

Volume level scales linearly with value. The value `0` represents silence, and the value `3` represents the loudest volume.

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

A value of `$0` specifies the least-significant bit, and a value of `$F` specifies the most-significant bit, of the shift register.


### Control

The `NOx7` registers manage the operation of the noise channels. They all have the same format:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Run | | | | | | AM | Enable Tap B |

Bit 0 specifies how many taps are used. If clear, only Tap A is used and the pseudo-random bit is the bit of the LFSR specified by the tap's position setting. If set, the pseudo-random bit is the exclusive OR of the bits of the LFSR specified by both tap position settings.

If bit 1 is set, amplitude modulation is enabled. This feature uses the upper 4 bits of the slope accumulator value as the amplitude.

Bits 2 to 6 have no function.

If bit 7 is clear, the frequency generator is paused, preventing the noise algorithm from working and the shift register from automatically updating.


### Noise Algorithm and Output Procedure

Every time the frequency phase accumulator wraps around, a pseudo-random bit is produced according to the specified tap locations and whether Tap B is enabled. The shift register contents is shifted left by one bit position, and the new bit 0 is the pseudo-random bit. The state of bit 15 is lost.

Bit 15 of the shift register contains the binary amplitude that is incorporated in the output. If clear, the amplitude is low. If set, the amplitude is high.

The wave amplitude value is 4-bit. The binary amplitude is stretched to 4 bits, such that low is `$0` and high is `$F`. If amplitude modulation is enabled, a bitwise AND of this 4-bit amplitude and the upper 4 bits of the slope accumulator value is produced, effectively causing the `$F` amplitude to be replaced with the slope amplitude.

The wave amplitude value is multiplied by the 4-bit volume value to produce the final 8-bit unsigned amplitude value for the channel. The maximum amplitude is `225`.



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


### Output Procedure

Every time the frequency phase accumulator wraps around, the slope accumulator is modified according to the algorithms.

The upper 4 bits of the slope accumulator value are used as the 4-bit wave amplitude value. This amplitude value is multiplied by the 4-bit volume value to produce the final 8-bit unsigned amplitude value for the channel. The maximum amplitude is `225`.



## Sound Channel 5 (PCM)

This channel outputs an arbitrary 8-bit unsigned amplitude written to the `DAC` register. The sound chip does not automatically modify `DAC` to produce a waveform, so samples must be provided by the CPU or DMA controller.

The 8-bit amplitude is multiplied by the 2-bit volume setting in `LVL` to produce the final 10-bit unsigned amplitude value for the channel. The maximum amplitude is `765`.


### PCM Playback with DMA

The Hexheld DMA controller is configurable enough to be able to deliver samples to `DAC` at the reload frequency of Timer A.

The timer is a 13-bit up-counter. It can use any one of the same 8 clock frequencies as the frequency generators. The counter will divide the clock frequency by an amount from 1 to 8,192 to become the frequency to step the DMA transfer at.

The size of a sound can vary from 1 to 65,536 samples (64 KB), according to `DxCNT`. After playing the last sample, it can either hang there or loop back to the first sample.

As the Hexheld memory address bus is 24 bits wide, up to 16 MB of PCM sound data can be addressed by pointing `DxSA` to the location of the first sample. Further samples follow consecutively at higher addresses.



## Mixers

All sound channels are mixed together digitally—by adding their final amplitude values together—to produce the final 11-bit unsigned mix. The highest possible sum for this mix is `1665`.

On loudest volumes, the noise and slope channels each span 1/8 of the mix, and the PCM channel spans 3/8 of the mix.

When a channel is disabled in `ACR`, its input amplitude value is `0`.

External audio inputs are analog, and are therefore not incorporated in the digital mixing. The digital mixing amplitudes are made analog before being mixed with the inputs.

NOTE: As the output is stereo, the same mixing algorithms apply for both sides, only that each side incorporates its own volume values.



## Reset

When the Hexheld system is reset, all Power Noise registers are initialized with `$00`.

Since the slope wave generation resets as well, it may be thought that `SLO7` is initialized with `$40`.
