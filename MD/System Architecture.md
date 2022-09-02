
# System Architecture

Hexheld is a fantasy handheld console created by jvsTSX, [The Beesh-Spweesh](https://twitter.com/StinkerB06), and a couple other people. It uses custom underlying hardware, presenting its own unique challenges for programmers and developers.

The heart of the Hexheld system is the **HiveCraft**—a custom [SoC](https://en.wikipedia.org/wiki/System_on_a_chip) embedding most of the Hexheld's internal hardware, including the CPU, graphics and sound processors, and working memory (WRAM). Some hardware is external from this chip, notably the VRAM.



## Specifications

```
Display         168×224 hexagon (shape and pixel cells) screen with 8 grayshades, RGB backlight, optional overlays
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
- VRAM is external from the HiveCraft, and is slower than WRAM by a margin.



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



## HiveCraft Components


### CPU - Pilot16

A custom 16-bit processor. It can be thought of as a sibling of the [Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80) and various other designs influenced by that.

Feature set:
- 9 general-purpose 8-bit registers, usable as 4 16-bit registers
- Shadow registers (values exchangeable with main registers)
- Segmentation scheme with 16-bit segment and 16-bit offset values
- Banking scheme with 8-bit bank and 16-bit offset values
- Repetition feature to reduce code size
- Pipelined execution

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
- 2 tile layers with a selection of operating modes
- 128 sprites on-screen, size from 8×8 to 8×32
- Programmable rectangular layer/sprite clipping region
- 8 grayshades with 16 palette entries
- Line counter compare function


### Sound - Power Noise

A 5-channel programmable sound generator.

Feature set:
- 3 noise channels (LFSR synthesis)
- 1 "slope" channel
- 1 PCM channel
- Stereo volumes/balancing
- Expansion audio mixing from Hexridge


### DMA Controller

This component provides 2 flexible [Direct Memory Access (DMA)](https://en.wikipedia.org/wiki/Direct_memory_access) channels.


### Timers

These are 2 13-bit up-counters, lettered A and B, which may generate periodic interrupts among other roles.


### Communication

These components send out and receive in bytes bit-wise over the connected link cable, and control the infrared light transmitter/receiver on the Hexheld unit.


### Hexridge Interface

This component controls the Hexridge interface and ROM wait generation.


### Interrupt Controller

This component is responsible for sending interrupt requests to the CPU, and supplies the necessary information to allow the CPU to run the appropriate interrupt handlers.


### Backlight Controller

This component controls the RGB backlight, and may use [PWM](https://en.wikipedia.org/wiki/Pulse-width_modulation) to achieve varying intensities.


### Battery ADC

This component measures the incoming battery voltage, and accordingly updates the battery level indicator on the Hexheld unit. The software may read the battery level.
