**This document is in "Work In Progress" (WIP) state, hence that it's incomplete, and that new edits will be released in the future.**

---

# CPU - Pilot16

Pilot is a fantasy CPU architecture created by [The Beesh-Spweesh](https://twitter.com/StinkerB06) under proposals made by jvsTSX. Hexheld is the primary application for this CPU, where it is embedded inside the HiveCraft SoC and can be clocked at up to 8 MHz.

The architecture is 16-bit, influenced by the 8-bit [Zilog Z80](https://en.wikipedia.org/wiki/Zilog_Z80) and likes, but is incompatible with those designs.

The CPU incorporates a 3-stage [pipeline](https://en.wikipedia.org/wiki/Instruction_pipelining) and instruction prefetch unit to improve performance of running code.



## Memory Access

Although the CPU has a 24-bit address bus, its architecture does not use linear 24-bit addresses. Instead, addresses are comprised of two values which get combined automatically to form the linear address.

The CPU takes in either an 8-bit *bank* value or a 16-bit *segment* value, then applies a 16-bit unsigned *offset* value to it. The bank or segment represents the *window* of 64 KB of memory that is to be used.


### Banking Scheme

The bank and offset values are concatenated together to form the 24-bit linear address, with the bank representing the upper 8 bits and the offset representing the lower 16 bits. The bank value may be implicit, come from the `K` register (when fetching instructions), or come from the `D` register while the `D` flag is set.

Absolute linear addresses are internally split up into a bank and offset.


### Segmentation Scheme

This scheme allows the memory window to be positioned on any 256-byte boundary.

To produce the 24-bit linear address, the segment value is shifted left by 8 bits, then the offset value is added to this shifted segment value. If the linear address exceeds `$FFFFFF`, it will wrap back to `$000000` and continue up from there.

Since the ALU is used to combine the segment and offset, an additional cycle is spent per memory access. Segmentation is thus slower than banking.

This scheme is only used in "`DS:`" addressing modes while the `D` flag is clear, with the `DS` register pair serving as the segment value. If the flag is set, the banking scheme is used with the `D` register serving as the bank value.


### Segment Adjust

In order to ease managements of 24-bit linear addresses, the processor provides a *segment adjust* feature consisting of the `A` flag and the `SAO` and `SAU` instructions.

The `A` flag is modified whenever an auto-index operand using "`DS:`" is present in an instruction, or is left alone otherwise. The flag is set when the offset register overflows past `$FFFF` or underflows below `$0000`, or is cleared otherwise.
- If both operands are as such, the `A` flag is set to an unpredictable state.

Some instructions do not modify the `A` flag according to the presence of such operand. Those either modify it according to the instruction operation or leave the flag alone.


### 24-bit Memory Reads

The `JPF` and `CALLF` instructions with an RM operand allow 24-bit data to be read from memory.

Technically, these are 32-bit reads, but the upper 8 bits of the 32-bit value are not observed. The 16-bit word halves of the value are read independently with the low word read first. As Hexheld uses a [little endian](https://en.wikipedia.org/wiki/Endianness) model for multi-byte values, the low word is found at the base address, and the high word follows consecutively.

CAUTION: The processor gets the address of the high word by increasing *only* the offset portion. If the effective offset of the RM operand is `$FFFE` (or `$FFFF` when the word-alignment behavior is considered), the words will not be read consecutively.


### I/O Ports

To access the I/O map, the 8-bit port number must be loaded into the `C` register. There is no way to specify the port number directly within an instruction.



## Registers


### General-Purpose Registers

| High | Low | 8-bit Role | 16-bit Role |
| -: | :- | :-: | :-: |
| `A` | `B` | Accumulators | Accumulator |
| `H` | `L` | | Accumulator |
| `I` | `X` | | Index |
| `D` | `S` | Data Bank | Data Segment |
| | `C` | I/O Port Number | |

The high and low registers may be paired together to form a 16-bit register. These pairs are named `AB`, `HL`, `IX`, and `DS`.

The accumulators are the registers that many arithmetic and logic operations rely on.

`AB` and `HL` may be combined to form the 32-bit register combination `ABHL`. This stores the product of the `MULSW` instruction and represents the dividend for the `DIVSW` instruction.


### Flags' Register (`F`)

This 8-bit register stores various condition code and operating flags.

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| `S` | `Z` | `I` | `H` | `A` | `V` | `D` | `C` |

`C` - Set if an unsigned addition produced a carry or subtraction borrowed, cleared otherwise. Also used by shift and rotate instructions as a bit input/output.

`D` - Represents the data segment mode. For more information, see **Memory Access**.

`V` - (See below.)

`A` - Serves as the conditional execution flag for the `SAO` and `SAU` instructions.

`H` - Similar to the `C` flag, but is only significant for the lower 4 bits (8-bit size) or 12 bits (16-bit size) of the operands. Serves as an input for the decimal adjust algorithm.

`I` - If clear, maskable interrupts are disabled.

`Z` - Set if the result of an operation is zero, cleared if not.

`S` - Set if the result of an operation is negative, cleared if not. This is a direct copy of the highest bit of the result.

#### `V` Flag

This flag has 2 purposes:

**Overflow** - Set if the result of an addition or subtraction is outside the signed range of the operation size, cleared otherwise.

**Parity** - Set if the amount of set bits in the result of an operation is even, cleared if odd.


### Shadow Registers

All general-purpose registers as well as the `F` register have duplicates called *shadow registers*.

The values of shadow registers may be exchanged with main registers using the `EXS` instruction. They can also be saved to and restored from the stack using the `PUSH ALL` and `POP ALL` instructions.

When referencing shadow registers, their names are the same as the main registers but have an apostrophe (`'`) suffix.

NOTE: `F'` does not include the `I` flag and is instead hardwired to zero. Exchanging `F` and `F'` will exchange all flags except for `I`.


### Stack Pointer (`SP`)

This is the 16-bit register that points to the most recently-pushed word on the [stack](https://en.wikipedia.org/wiki/Call_stack) contained in bank `$00` of memory.

Push operations decrease this register by 2 before storing the word to memory, and pop operations increase this register by 2 after reading the word from memory. That is, the "full-descending" convention is used.

Due to the requirement of making 16-bit word values in memory word-aligned, bit 0 of this register is hardwired to zero.


### Program Counter (`PC`)

This is the 16-bit register that holds the offset portion of the address of the next instruction in-line.

Due to the requirement of making 16-bit word values in memory word-aligned, bit 0 of this register is hardwired to zero.


### Program Bank (`K`)

This is the 8-bit register that holds the bank portion of the address of the next instruction in-line.

This register is combined with the `F` register to form the 16-bit register pair `KF`.

NOTE: The processor does not perform any adjustment to this register's value to handle crosses of bank boundaries. It is only changed by "far" flow control.


### Interrupt Enable Mask (`E`)

This 8-bit register stores a [bitmask](https://en.wikipedia.org/wiki/Mask_(computing)) specifying which maskable interrupts should be enabled. Bit number corresponds to interrupt number. If a bit is set, its corresponding interrupt is enabled.


### Z Registers (`Z0`-`Z15`)

These 16-bit "registers" reside in the first 32 bytes (16 words) of memory. They are used as scratch storage and can hold offsets for memory accesses.

The high or low 8-bit half of a Z register can be accessed by suffixing "`H`" or "`L`" to the register's name.

Z registers are mapped in memory as follows:

| Linear Address | 16-bit Name | 8-bit Name |
| :-: | :-: | :-: |
| `$000000` | `Z0` | `Z0L` |
| `$000001` | | `Z0H` |
| `$000002` | `Z1` | `Z1L` |
| ... | | |
| `$00001E` | `Z15` | `Z15L` |
| `$00001F` | | `Z15H` |



## Addressing Modes

There are several different types of operands for instructions. These types are collectively the *addressing modes* supported by the CPU.


### Register Direct

The operand specifies a register. When writing the operand, it is simply the register's name without anything preceding or following it.

When a Z register is specified, it is technically an absolute memory operand.


### Extension Register

The operand specifies a 16-bit extension of an 8-bit register. Extension registers can be a source operand, but cannot be a read-modify-write or destination operand.

The following extension registers are available:

- `AZX` - `A` zero-extended to 16 bits.
- `ASX` - `A` sign-extended to 16 bits.
- `BSX` - `B` sign-extended to 16 bits.


### Memory - Absolute

The operand specifies a memory access using a literal linear address or offset.

The following variations of absolute addressing are available:

| Variation | Syntax |
| :-: | :- |
| Bank `$00` | `[linear]` |
| Bank `$01` | `[linear]` |
| Data Segment | `DS:[offset]` |


### Memory - Register Offset

The operand specifies a memory access using a register's value as the offset portion. The memory access may use bank `$00` or the data segment.

The offset register may be `AB`, `HL`, `IX`, `DS`, or a Z register. For the latter, an additional memory access is performed to get the Z register's value.

When writing the operand, the register's name is surrounded by square brackets. When using the data segment, "`DS:`" immediately precedes the left square bracket.

NOTE: The processor does not support `DS:[DS]`.


### Memory - Automatic Index

The operand specifies a memory access using a register's value as the offset portion, with the register being modified by an amount after the access. The memory access may use bank `$00` or the data segment.

The offset register may be `HL`, `IX`, or a Z register. For the latter, two additional memory accesses are performed before and after to get the Z register's value and write the auto-index modification, respectively.

The register value may be either increased or decreased. The amount is implicit and depends on the operation size: 1 for 8-bit or 2 for 16-bit. The index occurs immediately after the operand has been read from (source) or written to (read-modify-write or destination).

When the data segment is used and if the instruction allows, the `A` flag reports whether the register value wrapped and that a segment adjustment is required. For more information, see **Segment Adjust**.

The operand is written the same way as a Register Offset operand, but the register name within the square brackets is immediately followed by a "`+`" or "`-`" symbol to specify auto-indexing. For example, an auto-increase of `IX` with the data segment is written as "`DS:[IX+]`".


### Memory - Displacement Index

The operand specifies a memory access where the offset portion is the calculated sum of two values. If the sum exceeds `$FFFF`, it will wrap back to `$0000` and continue up from there.

The following variations of indexed addressing are available:

| Variation | Syntax |
| :-: | :- |
| Bank `$00` | `[IX+imm]`, `[SP+imm]`, `[Zn+IX]` |
| Data Segment | `DS:[IX+imm]`, `DS:[Zn+IX]` |

- `imm` is a literal value.
- The Z register variations perform an additional memory access to get the Z register's value.

Since the ALU is used to compute the offset, an additional cycle is spent.


### I/O Addressing

The operand specifies an I/O access. The following variations of I/O addressing are available:

- `[C]` - The port specified by `C` is accessed.
- `[C+]` - The port specified by `C` is accessed, then `C` is incremented by 1.
- `[C-]` - The port specified by `C` is accessed, then `C` is decremented by 1.


### Immediate

The operand is a literal value. Immediate values can be a source operand, but cannot be a read-modify-write or destination operand.



## Instruction Encoding

Every instruction found in memory begins with a 16-bit *opcode word*. Depending on the operands, up to 2 additional words may follow the opcode word. All of these words together make up the instruction's *machine code*. The processor spends a cycle fetching each word, so longer instructions take longer to run through the pipeline.

Certain fields pertaining to the operands of the instruction may be present in the opcode word:


### RM - Register/Memory Operand

A type of operand that can be used with most instructions.

Each RM operand uses a 5-bit field of the opcode word, and sometimes an additional instruction word. The 5-bit field specifies what the RM operand is.

The following are the available RM memory and immediate operands:

| | Syntax | | Syntax | | Syntax |
| :-: | :- | :-: | :- | :-: | :- |
| `0` | `imm` | `1` | `[HL+]` | `2` | `[AB]` |
| `4` | `[$010000+imm]` | `5` | `DS:[HL+]` | `6` | `DS:[AB]` |
| `8` | `[imm]` | `9` | `[HL-]` | `10` | `[HL]` |
| `12` | `DS:[imm]` | `13` | `DS:[HL-]` | `14` | `DS:[HL]` |
| `16` | `[IX+imm]` | `17` | `[IX+]` | `18` | `[IX]` |
| `20` | `DS:[IX+imm]` | `21` | `DS:[IX+]` | `22` | `DS:[IX]` |
| `24` | `[SP+imm]` | `25` | `[IX-]` | `26` | `[DS]` |
| `28` | Invalid | `29` | `DS:[IX-]` | `30` | Invalid |

- `imm` is a 16-bit unsigned literal value from an additional instruction word. If the operation size is 8-bit, only the lower 8 bits of the literal value are significant.

The following are the available RM register operands by operation size:

| | 8-bit | 16-bit |
| :-: | :-: | :-: |
| `3` | `A` | `AB` |
| `7` | `B` | `ASX` |
| `11` | `H` | `HL` |
| `15` | `L` | `BSX` |
| `19` | `I` | `IX` |
| `23` | `X` | `AZX` |
| `27` | `D` | `DS` |
| `31` | `S` | `SP` |

NOTES:
1. Instructions usually accept only one RM operand, but `LD` instructions can use two. If both operands use an additional instruction word, the word pertaining to the source operand precedes the word pertaining to the destination operand.
2. Immediate, auto-indexed memory, and register addressing modes are invalid for 24-bit RM operands.

Throughout this document, the following notations for the 5-bit field values are used:

- `sssss` - Source operand.
- `mmmmm` - Read-modify-write operand.
- `ddddd` - Destination operand.


### ZM - Z Register/Memory Operand

A type of operand that relies on Z registers, and can also specify displacement-indexed addressing modes that use a short-form literal value. ZM operands can only be used with `LD` instructions involving accumulators.

The ZM operand is encoded in the lowest 8 bits of the opcode word:

| | Syntax |
| :-: | :- |
| `000nnnn0` | `[Zn+]` |
| `000nnnn1` | `DS:[Zn+]` |
| `001nnnn0` | `[Zn-]` |
| `001nnnn1` | `DS:[Zn-]` |
| `010nnnn0` | `[Zn+IX]` |
| `010nnnn1` | `DS:[Zn+IX]` |
| `011nnnn0` | `[Zn]` |
| `011nnnn1` | `DS:[Zn]` |
| `100iiiii` | `[IX+imm]` |
| `101iiiii` | `[SP+imm]` |
| `110iiiii` | `DS:[IX+imm]` |
| `111nnnnx` | `[imm]` / `Zn` (16-bit) |
| `111nnnn0` | `[imm]` / `ZnL` (8-bit) |
| `111nnnn1` | `[imm]` / `ZnH` (8-bit) |

- `n` is the Z register's number from `0` to `15`.
- `imm` is a 5-bit literal value which is zero-extended to 16 bits.


### Accumulator Register

Some instructions operate using accumulator registers. Bits 8 and 10 of the opcode word specify the exact accumulator:

| Bit 10 | Bit 8 | Accumulator |
| -: | :- | :-: |
| `0` | `0` | `A` |
| `0` | `1` | `B` |
| `1` | `0` | `AB` |
| `1` | `1` | `HL` |

The size of the accumulator directly influences the operation size.



## Instruction Syntax

When writing an instruction, its *mnemonic* is written first. If operands are present, they follow the mnemonic and are separated from the mnemonic by a space. If multiple operands are present, they are separated by commas "`,`".


### Operand Types

- **Source** - The operand is only read from. It is the right-most operand.
- [**Read-Modify-Write (RMW)**](https://en.wikipedia.org/wiki/Read%E2%80%93modify%E2%80%93write)
- **Destination** - The operand is only written to.
- **Bit Number** - The operand specifies bit number of the following operand. It is the left-most operand.
- **Condition Code** - The operand represents the conditional execution code. It is the left-most operand.



## Operation Size

CPU operations may be performed on either 8-bit or 16-bit data types. The size is either specified by one of the bits in the opcode word, or is implied in the instruction.

When writing instructions, size can be specified by suffixing a letter to the instruction's mnemonic: "`B`" for 8-bit or "`W`" for 16-bit. For example, 16-bit `INC` is written as "`INCW`".

NOTE: Operation size is evident if a register operand is present (the register's name implies its size), so the suffix is optional. In some cases (such as a `LD` instruction having both operands be memory accesses), it is not evident. The default operation size is 8-bit.
