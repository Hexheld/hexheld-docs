**This document is in "Work In Progress" (WIP) state. It cannot be merged into `main` until it is considered ready.**

---

# CPU - Pilot16

Pilot is a fantasy CPU architecture created by [The Beesh-Spweesh](https://twitter.com/StinkerB06) based on some proposals made by jvsTSX. Hexheld is the primary application for this CPU, where it is embedded inside the HiveCraft SoC and can be clocked at up to 8 MHz.

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

To ease 24-bit pointer arithmetic, the processor provides a *segment adjust* feature. The upper 8 bits of the pointer are to be in the `D` register and the lower 16 bits in some other register.

The `A` flag reports an overflow/underflow condition from a calculation done to the 16-bit offset portion, and the `SAO`/`SAU` instruction modifies `D` to move the data segment window forwards/backwards by 64 KB if the condition occured.

For many instructions, segment adjustments work off of auto-indexed memory operands using the data segment:

#### Auto-Index Using Data Segment

- When this type of memory operand is present in the instruction, the `A` flag is set if the offset register overflowed past `$FFFF` or underflowed below `$0000`, or is cleared otherwise.
- When multiple of the operands are of this type, the `A` flag is set to an unpredictable state. 
- When this type is not present, the `A` flag is left alone.


### 24-bit Memory Reads

The `JPF` and `CALLF` instructions with an RM operand allow 24-bit data to be read from memory.

Technically, these are 32-bit reads, but the upper 8 bits of the 32-bit value are not observed. The 16-bit word halves of the value are read independently with the low word read first. As Hexheld uses a little-[endian](https://en.wikipedia.org/wiki/Endianness) model for multi-byte values, the low word is found at the base address, and the high word follows consecutively.

The processor gets the address of the high word by increasing the linear address by 2. When the effective offset of the RM operand is `$FFFE` (or `$FFFF` when the word-alignment behavior is considered), the high word is read outside the bank/segment window of the RM operand.


### I/O Ports

To access the I/O map, the 8-bit port number must be loaded into the `C` register. There is no way to specify the port number directly within an instruction.



## Registers


### General-Purpose Registers

The CPU has 9 8-bit registers organized as follows:

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

This 8-bit register stores various condition code and operating flags:

| Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| `S` | `Z` | `I` | `H` | `A` | `V` | `D` | `C` |

`C` - Set if an addition or subtraction produced a carry, cleared otherwise. Also used by shift and rotate instructions as a bit input/output.

`D` - Represents the data segment mode. For more information, see **Memory Access**.

`V` - (See below.)

`A` - Serves as the conditional execution flag for the `SAO` and `SAU` instructions.

`H` - Similar to the `C` flag, but is only significant for the lower 4 bits (8-bit size) or 12 bits (16-bit size) of the operands and result. Serves as an input for the decimal adjust algorithm.

`I` - If clear, maskable interrupts are disabled.

`Z` - Set if the result of an operation is zero, cleared if not.

`S` - Set if the result of an operation is negative, cleared if not. This is a direct copy of the highest bit of the result.

NOTE: The proper terminology for a subtraction carry is a *borrow*.

#### `V` Flag

This flag has 2 purposes:

**Overflow** - Set if the result of an addition or subtraction is outside the signed range of the operation size, cleared otherwise.

**Parity** - Set if the amount of set bits in the result of an operation is even, cleared if odd.


### Shadow Registers

All general-purpose registers as well as the `F` register have duplicates called *shadow registers*.

The values of shadow registers may be exchanged with main registers using the `EXS` instruction. They can also be saved to and restored from the stack using the `PUSH ALL` and `POP ALL` instructions.

When referencing shadow registers, their names are the same as the main registers but have an apostrophe (`'`) suffix.

NOTE: `F'` does not include the `I` flag and is instead hardwired to zero. If `F` is one of the selected registers in an `EXS` instruction, the `I` flag will not change.


### Stack Pointer (`SP`)

This is the 16-bit register that holds the offset portion of the address of the most recently-pushed word on the [stack](https://en.wikipedia.org/wiki/Call_stack) contained in memory. The bank portion is fixed to `$00`.

Push operations decrease this register by 2 before storing the word to memory, and pop operations increase this register by 2 after reading the word from memory. That is, the "full-descending" convention is used. Pop operations do not perform any clean-up: the stack memory remains the same after the pop operation completes.

Bit 0 of this register is hardwired to zero. That is, any odd value written into this register will be rounded down to the next lower even.


### Program Counter (`PC`)

This is the 16-bit register that holds the offset portion of the address of the next instruction in-line.

Bit 0 of this register is hardwired to zero. That is, any odd value written into this register will be rounded down to the next lower even.


### Program Bank (`K`)

This is the 8-bit register that holds the bank portion of the address of the next instruction in-line.

This register is combined with the `F` register to form the 16-bit register pair `KF`.

NOTE: The processor does not perform any adjustment to this register's value in any situation where `PC` wraps around. It is only changed by "far" flow control instructions and by interrupt handlers starting.


### Interrupt Enable Mask (`E`)

This 8-bit register stores a bitmask specifying which maskable interrupts should be enabled. Bit number corresponds to interrupt number. If a bit is set, its corresponding interrupt is enabled.


### Z Registers (`Z0`-`Z15`)

These 16-bit "registers" reside in the first 32 bytes (16 words) of memory. They are used as scratch storage and can hold offsets for memory accesses.

The high or low 8-bit half of a Z register can be specified by suffixing "`H`" or "`L`" to the register's name.

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

Instruction operands come in several varieties. These varieties are collectively the *addressing modes* supported by the CPU.


### Register Direct

The operand specifies a register. When writing the operand, it is simply the register's name without anything preceding or following it.

When a Z register is specified, it is technically an absolute memory operand.


### Extension Register

A subtype of Register Direct, where the operand specifies a 16-bit extension of an 8-bit register. Extension registers can be a source operand, but cannot be a read-modify-write or destination operand.

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

The register value may be either increased or decreased. The amount is implicit and depends on the operation size: 1 for 8-bit, 2 for 16-bit, or 4 for 24-bit. The index occurs immediately after the operand has been read from (source) or written to (read-modify-write or destination).

When the data segment is used and if the instruction allows, the `A` flag reports whether the register value wrapped and that a segment adjustment is required. For more information, see **Segment Adjust**.

The operand is written the same way as a Register Offset operand, but the register name has a "`+`" or "`-`" suffix. For example, an auto-increase of `IX` with the data segment is written as "`DS:[IX+]`".

> Editor's Note: Incorporate the cycle penalties for auto-indexing into this document.


### Memory - Displacement Index

The operand specifies a memory access where the offset portion is the calculated sum of two values. If the sum exceeds `$FFFF`, it will wrap back to `$0000` and continue up from there.

The following variations of displacement-indexed addressing are available:

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
- `[C+]` - The port specified by `C` is accessed, then `C` is increased by 1.
- `[C-]` - The port specified by `C` is accessed, then `C` is decreased by 1.


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

- `imm` is a 16-bit unsigned literal value from an additional instruction word. In the immediate addressing mode, if the operation size is 8-bit, only the lower 8 bits of the instruction word are regarded and its upper 8 bits may contain anything.

The following are the available RM register direct operands by operation size:

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
2. The immediate and register direct addressing modes are invalid for 24-bit RM operands.

Throughout this document, the following notations are used for RM operands:

| 5-bit Field Bits | Label | Description |
| :-: | :-: | :-: |
| `sssss` | **RM.SRC** | Source operand. |
| `mmmmm` | **RM.RMW** | Read-modify-write operand. |
| `ddddd` | **RM.DST** | Destination operand. |


### ZM - Z Register/Memory Operand

A type of operand that relies on Z registers, and can also specify displacement-indexed addressing modes that use a short-form literal value.

The ZM operand is encoded in the lower 8 bits of the opcode word:

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

The size of the accumulator directly determines the operation size.


### CFE Operand

A type of operand that relies on the `C`, `F`, and `E` registers. Bits 6 to 8 of the opcode word specify what this operand is:

| | Operand |
| :-: | :-: |
| `2` | `[C+]` |
| `3` | `F` |
| `4` | `[C]` |
| `5` | `C` |
| `6` | `[C-]` |
| `7` | `E` |



## Instruction Syntax

When writing an instruction, its *mnemonic* is written first. If operands are present, they follow the mnemonic and are separated from the mnemonic by a space. If multiple operands are present, they are separated by commas (`,`).


### Operand Types

- **Source** - The operand is only read from. It is the right-most operand.
- [**Read-Modify-Write (RMW)**](https://en.wikipedia.org/wiki/Read%E2%80%93modify%E2%80%93write)
- **Destination** - The operand is only written to.
- **Bit Number** - The operand specifies bit number of the following operand. It is the left-most operand.
- **Condition Code** - The operand represents the conditional execution code. It is the left-most operand.



## Operation Size

CPU operations may be performed on either 8-bit or 16-bit data types. The size is either specified by one of the bits in the opcode word, or is implied in the instruction. Size influences the bit width of computations and whether memory operand accesses will be 8-bit or 16-bit.

When writing instructions, size can be specified by suffixing a letter to the instruction's mnemonic: "`B`" for 8-bit or "`W`" for 16-bit. For example, 16-bit `INC` is written as "`INCW`".

NOTE: Operation size is evident if a register direct operand is present (the register's name implies its size), so the suffix is optional. In some cases (such as a `LD` instruction having both operands be memory accesses), it is not evident. The default operation size is 8-bit.



## Instruction Set - Data Transfer

Instructions in this category copy or exchange values around.


### `LD` - Load

Operation: `dst = src`

Loads the destination operand with a copy of the source operand.

Flags (unless the destination is `F`):

- `SZIH-VDC` - Not modified.
- `----A---` - **Auto-Index Using Data Segment**

The following encodings are available for `LD`:

| Opcode Word | Operation Size | Destination | Source |
| :-: | :-: | :- | :- |
| `0001 000c cc0d dddd` | 8-bit | RM.DST | CFE Operand |
| `0001 001c cc0s ssss` | 8-bit | CFE Operand | RM.SRC |
| `0101 0a0a zzzz zzzz` | 8-bit, 16-bit | ZM Operand | Accumulator |
| `0101 0a1a zzzz zzzz` | 8-bit, 16-bit | Accumulator | ZM Operand |
| `0110 00dd ddds ssss` | 8-bit | RM.DST | RM.SRC |
| `0110 01dd ddds ssss` | 16-bit | RM.DST | RM.SRC |
| `0111 0000 iiii iiii` | 8-bit | `A` | Immediate |
| `0111 0100 iiii iiii` | 8-bit | `B` | Immediate |
| `0111 0001 iiii iiii` | 8-bit | `H` | Immediate |
| `0111 0101 iiii iiii` | 8-bit | `L` | Immediate |
| `0111 0010 iiii iiii` | 8-bit | `I` | Immediate |
| `0111 0110 iiii iiii` | 8-bit | `X` | Immediate |
| `0111 0011 iiii iiii` | 8-bit | `D` | Immediate |
| `0111 0111 iiii iiii` | 8-bit | `S` | Immediate |
| `0100 0000 iiii iiii` | 8-bit | `C` | Immediate |

- The `i` bits represent the immediate value.

NOTE: If the destination is an auto-indexed memory access using the data segment and the source is the `F` register, the register's old value will be written to memory before the `A` flag gets modified.


### `CLR` - Clear

Operation: `dst = 0`

Loads the destination operand with a zero value. This is preferred over `LD ..., 0` as the machine code overhead of the immediate value is avoided.

Flags:

- `SZIH-VDC` - Not modified.
- `----A---` - **Auto-Index Using Data Segment**

The following encodings are available for `CLR`:

| Opcode Word | Operation Size | Destination |
| :-: | :-: | :- |
| `0010 0011 100d dddd` | 8-bit | RM.DST |
| `0010 0111 100d dddd` | 16-bit | RM.DST |


### `LDIXA`, `LDSPA`, `LDPCA` - Load Relative Address

Operations: `dst = IX + imm`, `dst = SP + imm`, `dst = PC + imm; D = K; DF = 1`

Adds the `IX`/`SP`/`PC` register and a 16-bit immediate value from an additional instruction word together, storing the sum to the destination operand.

`LDPCA` additionally performs the `SDS` instruction's operation afterwards to configure the data segment to be the same as the current program bank.

Flags:

- `SZIH-V-C` - Not modified.
- `----A---` - (`LDIXA`) Set if the sum of the unsigned `IX` value and signed immediate value is above `$FFFF` or below `$0000`, cleared otherwise.
- `----A---` - (`LDSPA`, `LDPCA`) **Auto-Index Using Data Segment**
- `------D-` - `LDPCA` sets this flag. `LDIXA` and `LDSPA` leave this flag alone.

These instructions have the following encodings:

| | Opcode Word | Operation Size | Destination | Source |
| :-: | :-: | :-: | :- | :- |
| `LDIXA` | `0001 0000 010d dddd` | 16-bit | RM.DST | Immediate |
| `LDSPA` | `0001 0000 011d dddd` | 16-bit | RM.DST | Immediate |
| `LDPCA` | `0001 0000 001d dddd` | 16-bit | RM.DST | Immediate |

NOTE: If the RM operand uses an additional instruction word, that word follows after the word for the immediate operand value.


### `SDS` - Set Data Segment to Current Program Bank

Operation: `D = K; DF = 1`

Sets the `D` register to the value of the `K` register, and sets the `D` flag.

No other flags besides `D` are affected.

The opcode word corresponding to `SDS` is `$100C`.


### `PUSH` - Push Data to Stack

Operation: `temp = src; SP -= 2; (word)[SP] = temp`

Pushes the source operand's value onto the stack.

Flags:

- `SZIH-VDC` - Not modified.
- `----A---` - **Auto-Index Using Data Segment**

The following encodings are available for `PUSH`:

| Opcode Word | Operation Size | Source |
| :-: | :-: | :- |
| `0100 0011 100s ssss` | 16-bit | RM.SRC |
| `0100 0011 1001 1111` | 16-bit | `KF` |
| `0100 0011 1011 1111` | 16-bit | `GPR` |
| `0100 0011 1111 1111` | 16-bit | `ALL` |

- It is not possible for the RM operand to specify `SP` in register direct addressing, as the resulting opcode word coincides with `PUSH KF`.

`PUSH GPR` performs multiple push operations in one instruction to push all general-purpose registers onto the stack:

1. Push the value of the `C` register zero-extended to 16 bits.
2. `PUSH AB`
3. `PUSH HL`
4. `PUSH IX`
5. `PUSH DS`

`PUSH ALL` performs multiple push operations in one instruction:

1. Push `(F' << 8) | F`.
2. Push `(C' << 8) | C`.
3. Push `AB'`.
4. Push `HL'`.
5. Push `IX'`.
6. Push `DS'`.
7. `PUSH AB`
8. `PUSH HL`
9. `PUSH IX`
10. `PUSH DS`

NOTE: If the RM operand is `[SP+imm]`, the `SP` register's old value will be used in the offset calculation, before it is decreased.


### `POP` - Pop Data from Stack

Operation: `dst = [SP]; SP += 2`

Removes data from the top of the stack and stores the data to the destination operand.

Flags (unless the destination is `F` or `ALL`):

- `SZIH-VDC` - Not modified.
- `----A---` - **Auto-Index Using Data Segment**

The following encodings are available for `POP`:

| Opcode Word | Operation Size | Destination |
| :-: | :-: | :- |
| `0100 0011 000d dddd` | 16-bit | RM.DST |
| `0100 0011 0001 1111` | 16-bit | `F` |
| `0100 0011 0011 1111` | 16-bit | `GPR` |
| `0100 0011 0111 1111` | 16-bit | `ALL` |

- It is not possible for the RM operand to specify `SP` in register direct addressing, as the resulting opcode word coincides with `POP F`.

Although `F` is an 8-bit register, `POP F` performs a 16-bit pop operation. `F` receives the lower 8 bits of the popped word value.

`POP GPR` performs multiple pop operations in one instruction to pop all general-purpose registers off of the stack:

1. `POP DS`
2. `POP IX`
3. `POP HL`
4. `POP AB`
5. Pop `C`. Although `C` is an 8-bit register, this is a 16-bit pop operation. `C` receives the lower 8 bits of the popped word value.

`POP ALL` performs multiple pop operations in one instruction:

1. `POP DS`
2. `POP IX`
3. `POP HL`
4. `POP AB`
5. Pop `DS'`.
6. Pop `IX'`.
7. Pop `HL'`.
8. Pop `AB'`.
9. Pop a 9th word. `C` receives the lower 8 bits, and `C'` receives the upper 8 bits, of this word.
10. Pop a 10th word. `F` receives the lower 8 bits, and `F'` receives the upper 8 bits, of this word.

NOTE: If the RM operand is `[SP+imm]`, the `SP` register's old value will be used in the offset calculation, before it is increased.


### `EXS` - Exchange Shadow Registers

Operation: `reglist <=> reglist'`

Exchanges the values of one or more main registers with the values of their corresponding shadow registers.

The opcode word for `EXS` has the binary format of `0000 0f0c dsix hlab`. Each bit labeled with a letter corresponds to an 8-bit register with the matching name. If a bit is set, the values of its corresponding main and shadow registers will be exchanged.

The entire list of registers to exchange is specified by one or more operands. Each operand is written like a Register Direct operand, using either a single 8-bit register's name or a concatenation of multiple 8-bit registers' names in any order.

Examples: `EXS A` `EXS DHL` `EXS AB, HL, IX, DS, C, F`

The special situation in which no registers are specified is canonized as the `NOP` instruction.

NOTE: If `F` is one of the specified registers, the `I` flag will not be changed as that is the only flag to not have a shadow.



## Instruction Set - Arithmetic

Instructions in this category perform math operations on data using the ALU.


### `ADD`, `ADC` - Add

Operations: `rmw += src`, `rmw += src + CF`

Adds the source operand to the read-modify-write operand.

`ADC` adds an extra 1 if the `C` flag is set, allowing the carry output of a previous addition or other operation to be carried into the addition. It merely performs a 3-addent addition.

Flags:

- `--I---D-` - Not modified.
- `S-------` - Set if the sum is negative, cleared if not. This directly copies its highest bit.
- `-Z------` - Set if the sum is zero, cleared if not.
- `---H----` - Set if the addition carried from bit 3 to bit 4 (8-bit) or from bit 11 to bit 12 (16-bit), cleared otherwise.
- `----A--C` - Both set if the addition produced a carry, both cleared otherwise.
- `-----V--` - Set if the sum of the signed operands is outside the signed range of the operation size, cleared otherwise. This is when the addents have the same sign but the sum ends up having the opposite sign.

The following encodings are available for these instructions:

| | Opcode Word | Operation Size | Read-Modify-Write | Source |
| :-: | :-: | :-: | :- | :- |
| `ADD` | `1000 0a0a iiii iiii` | 8-bit, 16-bit | Accumulator | Immediate |
| `ADD` | `1000 0a1a 000s ssss` | 8-bit, 16-bit | Accumulator | RM.SRC |
| `ADD` | `1000 0a1a 001m mmmm` | 8-bit, 16-bit | RM.RMW | Accumulator |
| `ADD` | `0100 0001 iiii iiii` | 8-bit | `C` | Immediate |
| `ADC` | `1001 0a0a iiii iiii` | 8-bit, 16-bit | Accumulator | Immediate |
| `ADC` | `1001 0a1a 000s ssss` | 8-bit, 16-bit | Accumulator | RM.SRC |
| `ADC` | `1001 0a1a 001m mmmm` | 8-bit, 16-bit | RM.RMW | Accumulator |

- The `i` bits represent the 8-bit immediate value. If the accumulator is 16-bit, the immediate value is zero-extended to 16 bits. If an immediate value above `$00FF` is desired, the RM operand should be used.

NOTE: `ADD C, imm` does not change any of the flags. The purpose of this variant of `ADD` is to apply an immediate offset relative to a base I/O port number.


### `SUB`, `SBC` - Subtract

Operations: `rmw -= src`, `rmw -= src + CF`

Subtracts the source operand from the read-modify-write operand.

`SBC` subtracts an extra 1 if the `C` flag is set, allowing the carry output of a previous subtraction or other operation to be carried into the subtraction. It merely performs a subtraction using 2 subtrahends.

Flags:

- `--I---D-` - Not modified.
- `S-------` - Set if the difference is negative, cleared if not. This directly copies its highest bit.
- `-Z------` - Set if the difference is zero, cleared if not.
- `---H----` - Set if the subtraction carried from bit 3 to bit 4 (8-bit) or from bit 11 to bit 12 (16-bit), cleared otherwise.
- `----A--C` - Both set if the subtraction produced a carry, both cleared otherwise.
- `-----V--` - Set if the difference of the signed operands is outside the signed range of the operation size, cleared otherwise. This is when the difference ends up having the opposite sign of the minuend and the same sign as the subtrahend.

The following encodings are available for these instructions:

| | Opcode Word | Operation Size | Read-Modify-Write | Source |
| :-: | :-: | :-: | :- | :- |
| `SUB` | `1010 0a0a iiii iiii` | 8-bit, 16-bit | Accumulator | Immediate |
| `SUB` | `1010 0a1a 000s ssss` | 8-bit, 16-bit | Accumulator | RM.SRC |
| `SUB` | `1010 0a1a 001m mmmm` | 8-bit, 16-bit | RM.RMW | Accumulator |
| `SBC` | `1011 0a0a iiii iiii` | 8-bit, 16-bit | Accumulator | Immediate |
| `SBC` | `1011 0a1a 000s ssss` | 8-bit, 16-bit | Accumulator | RM.SRC |
| `SBC` | `1011 0a1a 001m mmmm` | 8-bit, 16-bit | RM.RMW | Accumulator |

- The `i` bits represent the 8-bit immediate value. If the accumulator is 16-bit, the immediate value is zero-extended to 16 bits. If an immediate value above `$00FF` is desired, the RM operand should be used.


### `INC`, `DEC` - Increase, Decrease

Operations: `rmw += imm`, `rmw -= imm`

Adds/Subtracts an immediate value in the range of 1 to 8 to/from the read-modify-write operand.

If the operation size is 16-bit and the RM operand is register direct, flags are affected as follows:

- `SZIH-VDC` - Not modified.
- `----A---` - Set if the addition/subtraction produced a carry, cleared otherwise.

Otherwise, flags are affected as follows:

- `--I---DC` - Not modified.
- `S-------` - Set if the result is negative, cleared if not. This directly copies its highest bit.
- `-Z------` - Set if the result is zero, cleared if not.
- `---H----` - Set if the addition/subtraction carried from bit 3 to bit 4 (8-bit) or from bit 11 to bit 12 (16-bit), cleared otherwise.
- `----A---` - **Auto-Index Using Data Segment**
- `-----V--` - Set when the highest bit of the RM operand transitions from clear to set (`INC`) or from set to clear (`DEC`), cleared otherwise.

The following encodings are available for these instructions:

| | Opcode Word | Operation Size | Read-Modify-Write | Source |
| :-: | :-: | :-: | :- | :- |
| `INC` | `0010 0000 iiim mmmm` | 8-bit | RM.RMW | Immediate |
| `INC` | `0010 0100 iiim mmmm` | 16-bit | RM.RMW | Immediate |
| `DEC` | `0010 0001 iiim mmmm` | 8-bit | RM.RMW | Immediate |
| `DEC` | `0010 0101 iiim mmmm` | 16-bit | RM.RMW | Immediate |

- The `i` bits represent the immediate value minus 1.


### `MULU`, `MULSW` - Multiply

Operations: `AB = A * src`, `ABHL = AB * src`

Multiplies the source operand by an implied accumulator register, storing the product to an implied destination register. `MULU` treats the factors as unsigned, and `MULSW` treats the factors as signed.

Flags:

- `--IH--DC` - Not modified.
- `S-------` - Set if the product is negative, cleared if not. This directly copies bit 7 of the new value of `A`.
- `-Z------` - Set if the product is zero, cleared if not.
- `----A---` - **Auto-Index Using Data Segment**
- `-----V--` - This flag is cleared.

These instructions have the following encodings:

| | Opcode Word | Operation Size | Source |
| :-: | :-: | :-: | :- |
| `MULU` | `0001 0010 000s ssss` | 8-bit | RM.SRC |
| `MULSW` | `0001 0010 001s ssss` | 16-bit | RM.SRC |


### `DIVU`, `DIVSW` - Divide

Operations: `(AB / src); B = quotient; A = remainder`, `(ABHL / src); HL = quotient; AB = remainder`

Divides an implied register by the source operand, storing the quotient to its low half and the remainder to its high half.
- The remainder is the result of a modulo operation.

`DIVU` takes in a 16-bit unsigned dividend and 8-bit unsigned divisor, and produces 8-bit unsigned quotient and remainder results.

`DIVSW` takes in a 32-bit signed dividend and 16-bit signed divisor, and produces 16-bit signed quotient and remainder results. The magnitude (positive or negative) of the remainder is the same as the magnitude of the dividend.

Dividing by zero is invalid. When the source operand is 0, the quotient and remainder will not be written into the accumulators. The state of the `Z` flag can be tested to determine whether the division failed.

Flags:

- `--IH--DC` - Not modified.
- `S-------` - Set if the truncated quotient is negative, cleared if not. This directly copies its highest bit. Unpredictable if a division by zero attempted to be performed.
- `-Z------` - Set if a division by zero attempted to be performed, cleared otherwise.
- `----A---` - **Auto-Index Using Data Segment**
- `-----V--` - Set if the quotient is outside the 8-bit unsigned (`DIVU`) or 16-bit signed (`DIVSW`) range, cleared otherwise. Unpredictable if a division by zero attempted to be performed.

These instructions have the following encodings:

| | Opcode Word | Operation Size | Source |
| :-: | :-: | :-: | :- |
| `DIVU` | `0001 0010 010s ssss` | 8-bit | RM.SRC |
| `DIVSW` | `0001 0010 011s ssss` | 16-bit | RM.SRC |

NOTE: If `DIVSW` is used with an auto-indexed memory operand using `HL` as the offset register, `HL` will be modified first before being used as the lower 16 bits of the dividend.


### `NEG`, `NGC` - Negate

Operations: `rmw = 0 - rmw`, `rmw = 0 - (rmw + CF)`

Subtracts the read-modify-write operand from zero, storing the difference to the read-modify-write operand. This performs a **2's complement** negation.

`NGC` subtracts an extra 1 if the `C` flag is set, allowing the carry output of a previous negation or other operation to be carried into the subtraction. It merely performs a subtraction using 2 subtrahends.

Flags:

- `--I---D-` - Not modified.
- `S-------` - Set if the result is negative, cleared if not. This directly copies its highest bit.
- `-Z------` - Set if the result is zero, cleared if not.
- `---H----` - Set if the subtraction carried from bit 3 to bit 4 (8-bit) or from bit 11 to bit 12 (16-bit), cleared otherwise.
- `----A--C` - Both set if the subtraction produced a carry, both cleared otherwise.
- `-----V--` - Set if the result is `-128` (8-bit) or `-32768` (16-bit), cleared otherwise.

These instructions have the following encodings:

| | Opcode Word | Operation Size | Read-Modify-Write |
| :-: | :-: | :-: | :- |
| `NEG` | `0010 0010 000m mmmm` | 8-bit | RM.RMW |
| `NEG` | `0010 0110 000m mmmm` | 16-bit | RM.RMW |
| `NGC` | `0010 0011 000m mmmm` | 8-bit | RM.RMW |
| `NGC` | `0010 0111 000m mmmm` | 16-bit | RM.RMW |
