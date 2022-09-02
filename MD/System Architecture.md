
# System Architecture

Hexheld is a fantasy handheld console created by jvsTSX, [The Beesh-Spweesh](https://twitter.com/StinkerB06), and a couple other people. It uses custom underlying hardware, presenting its own unique challenges for programmers and developers.

The heart of the Hexheld system is the **HiveCraft**—a custom [SoC](https://en.wikipedia.org/wiki/System_on_a_chip) embedding most of the Hexheld's internal hardware, including the CPU, graphics and sound processors, and working memory (WRAM). Some hardware is external from this chip, notably the VRAM.



## Specifications

```
Display         168×224 hexagonal (shape and pixel cells) transflective LCD screen with 8 grayshades, RGB backlight, optional overlays
Aux. Display    8-character 7-segment display, battery level indicator
Connectivity    Detachable controller, link cable, infrared

Memory      64 KB RAM (32K WRAM + 32K VRAM)
Bus         24-bit addresses, 16-bit data bus with some 8-bit accesses, 8 MHz
CPU         Pilot - 8-bit and 16-bit operations, pipeline stages, up to 8 MHz
PPU         Radar - 2 tile layers + 128 sprites, window
Sound       Power Noise - 3× noise, 1× slope, 1× PCM, expansion audio, stereo
Hexridge    Up to 16 MB of mask ROM with optional SRAM and/or custom hardware
```



## Memory

The HiveCraft chip has a 16 MB memory address space, which means it can see up to 16,777,216 unique bytes via a 24-bit address bus value.

It has the following memory layout:

| Address Value | Component | Data Path | Size |
| :-: | :-: | :-: | :-: |
| `$000000` to `$007FFF` | WRAM | 16-bit | 32 KB |
| `$008000` to `$00FFFF` | VRAM | 8-bit | 32 KB |
| `$010000` to `$FFFFFF` | Hexridge | 8-bit or 16-bit | ~16 MB |

Only the CPU and DMA units can access the address space.

The PPU actively reads bytes from VRAM when enabled, so the hardware provides access slots for the CPU/DMA to access it. Consequently, each access may become delayed by a cycle. It is effectively 2 to 4 times slower than WRAM.

Accesses to Hexridge ROM may be delayed by an arbitrary amount of cycles to adapt to different ROM access latencies.


### Multi-Byte Values

As Hexheld is a [little endian](https://en.wikipedia.org/wiki/Endianness) system, the byte representing the lowest bit quantities is found at the base address, and the higher quantities are found at subsequent addresses.

16-bit word values in memory are intended to be aligned on 2-byte boundaries, that is, the address value must be even. Bit 0 of the 24-bit address is dropped by the hardware when doing a 16-bit access, so the low byte is always at an even address and the high byte at an odd address.

Some components use an 8-bit data bus width. If a 16-bit access is performed on such component, the access is automatically broken down into two individual 8-bit accesses with the access for the low byte happening first. As a result, more cycles are spent.


### RAM

Hexheld provides 64 KB of RAM. However, the 32 KB halves are separate memory modules:
- WRAM is embedded inside the HiveCraft chip. It is general-purpose memory and may be organized in any way. Generally, this is the fastest memory type available on Hexheld.
- VRAM is external from the HiveCraft and is slower than WRAM by a margin.



## I/O

In addition to the memory bus, the HiveCraft chip has a secondary, internal bus called the [I/O](https://en.wikipedia.org/wiki/Input/output) bus.

There are 256 unique I/O locations, addressed via an 8-bit *port number*. Each location has 8-bit contents.

The I/O map contains all Hexheld hardware registers:

| Port Number | Component |
| :-: | :-: |
| `$00` to `$1F` | LCD / PPU - Radar |
| `$20` to `$3F` | Sound - Power Noise |
| `$40` to `$4F` | Miscellaneous |
| `$50` to `$5F` | Interrupt Registers |
| `$60` to `$6F` | DMA Registers |
| `$70` to `$73` | Backlight |
| `$74` to `$7F` | - |
| `$80` to `$82` | System Registers |
| `$83` to `$FF` | - |

It is not recommended to access ports with an undefined purpose, as future revisions and successors to Hexheld may utilize them for their own use. All of these ports are read as the value `$00`.

When accessing I/O with Pilot CPU instructions, the port number is designated in the `C` register.

When I/O is the destination of a DMA transfer, the upper 2 bits of the port numbers are masked out, limiting the port numbers to range from `$00` to `$3F`. As such, only Radar and Power Noise registers can be targeted.


### HiveCraft Version Number

It is possible to detect the current version of the Hexheld hardware:

| Port Number | Name | Meaning |
| :-: | :- | :- |
| `$82` | `VER` | HiveCraft Version Number |

This register is read-only. The latest version as of this revision of this document is `$00`. The internal boot program will not start the software on the Hexridge if the `VER` value is less than the version number stored in the ROM header.


## Main display

The main display is a dot-matrix TFT LCD made up of hexagonal pixels arranged in a staggered hexagonal array. The screen profile is hexagon-shaped, with pixels outside the boundary omitted.

### Screen geometry and profile

Both the screen profile and the pixel cell profile are shaped like perfect hexagons, with equilateral sides and 120º internal corner angles, and vertically-oriented (corner tips facing up and down, flat sides facing left and right).

The screen resolution is measured in 168 pixels horizontally (edge-to-edge) and 224 pixels vertically down the center (tip-to-tip). Odd pixel lines are shifted to the right by half a pixel and the protruding corners from each line's pixels interlock between the adjacent lines' pixels.

The PPU drives the screen as if it were an ordinary 168x224 dot-matrix LCD with rectangularly-distributed dots. The software is responsible for providing graphics data that takes the screen geometry into consideration (particularly the fact that odd lines are offset by half a pixel).

### Backlight

The screen backlight is comprised of 3 individually driven channels of red, green and blue LED arrays. Each channel is driven by the HiveCraft's integrated backlight controller and can be individually toggled on/off or dimmed using PWM to generate different color combinations.

The red and green LED arrays are comprised of high-brightness gallium phosphide diodes while the blue array is comprised of contemporary state-of-the-art high-efficiency gallium nitride diodes. The drive current of the backlight LEDs is calibrated such that when all channels are driven simultaneously at full duty cycle, the resulting white light's chromaticity is correlated to the CIE D50 Illuminant standard.

Special consideration should be taken regarding the green channel of the backlight due to its output hue which is limited by the contemporarily available technology. Rather than a lime-green or emerald-green hue as commonly seen on CRT phosphors, the gallium phosphide green LEDs in the backlight emit a yellow-green hue.


## HiveCraft Components


### CPU - Pilot16

A custom 16-bit processor. It can be thought of as a sibling of the [Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80) and various other designs influenced by that.

Feature set:
- 9 general-purpose 8-bit registers, usable as 4 16-bit registers
- Shadow registers (values exchangeable with main registers)
- Segmentation scheme with 16-bit segment and 16-bit offset values
- Banking scheme with 8-bit bank and 16-bit offset values
- Repetition prefix to reduce code size
- 3-stage pipeline, prefetch queue, "always-take" branch prediction

Instructions:
- Size varying from 1 to 3 16-bit words
- Common arithmetic, logic, shift, and bit operations
- Multiplication and division
- Memory-to-memory transfer operations
- Support for both short and far jumps/calls


### PPU - Radar

An LCD controller and picture processor.

Feature set:
- 1BPP to 4BPP planar characters (8×8 size) and 4BPP packed bitmap graphics
- 2 character layers with a selection of operating modes and layer mixing settings
- 128 objects on-screen, size from 8×8 to 8×32, selectable color depths from 1BPP to 4BPP per object, automatic object attribute DMA
- Internal memory for character and object attribute, external memory access for pixel data
- Programmable rectangular layer/object clipping region
- 16-entry palette with 3 bits (8 possible grayshades) per entry
- Integrated dot-matrix LCD driver for the main display, integrated driver for the aux (segment/battery) displays
- Line counter compare function


### Sound - Power Noise

A 5-channel programmable sound generator.

Feature set:
- 3 "noise" channels (programmable LFSR synthesis)
- 1 "slope" channel, can modulate the amplitude of one or more of the "noise" channels
- 1 DAC channel, can be DMA-driven for PCM sample playback
- Stereo mixing/balancing
- Expansion audio mixing from Hexridge


### DMA Controller

This component provides 2 flexible [Direct Memory Access (DMA)](https://en.wikipedia.org/wiki/Direct_memory_access) channels.


### Timers

These are 2 13-bit up-counters, lettered A and B, which may generate periodic interrupts among other roles.


### Serial interfaces

These components send out and receive in bytes bit-wise over the connected link cable, and control the infrared light transmitter/receiver on the Hexheld unit.


### Hexridge Interface

This component controls the Hexridge interface and ROM wait state generation.


### Interrupt Controller

This component is responsible for sending interrupt requests to the CPU, and supplies the necessary information to allow the CPU to run the appropriate interrupt handlers.


### Backlight Controller

This component controls the color and brightness of the screen's RGB backlight, and is capable of using [PWM](https://en.wikipedia.org/wiki/Pulse-width_modulation) to dim the separate red, green and blue channels of the backlight in order to achieve varying hues and intensities.

Feature set:
- Software control over backlight state (off/on/PWM) for each individual red/green/blue channel
- Per-channel PWM duty cycle control with 5-bit intensity per channel
- 33 individually-assignably intensities per red/green/blue channel (0-31 PWM + "32" fully-on)
- Approximately 36000 different backlight setting combinations
- 1024 Hz PWM cycle rate (32768 Hz master cycle rate)


### Battery ADC

This component measures the incoming battery voltage, and accordingly updates the battery level indicator on the Hexheld unit.

Feature set:
- Software-triggered battery level sampling
- 8-level state-of-charge estimation using internal look-up tables
- Allows software to read out the determined battery level
