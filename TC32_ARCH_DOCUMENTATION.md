# OpenTC32: TC32 Architecture Reference

**Architecture reference for the Telink TC32 CPU core.**

TC32 is the 32-bit, 16-bit-instruction-word CPU used in Telink's TLSR82xx family of Bluetooth LE SoCs (TLSR8232, TLSR825x, ...). Telink has never published an architecture manual for it and only ships a proprietary, GCC 4.x-based toolchain. This document is the first public description of the core, recovered by black-box analysis of the vendor assembler/disassembler and cross-checked against real firmware images.

> **Status:** This reference records the instruction encodings and core behaviour reconstructed from the vendor toolchain and firmware analysis. Behaviour that remains to be confirmed on physical TC32 silicon is listed in [Open questions](#13-open-questions). Similarity to ARM Thumb is used as a cross-check, not as proof of identical silicon behaviour.

> **Disclaimer:** OpenTC32 is an independent project. It is not affiliated with, endorsed by, or sponsored by Telink Semiconductor. "Telink" and "TC32" are used purely to identify the hardware being described.

---

## Table of contents

1. [Overview](#1-overview)
2. [Programmer's model](#2-programmers-model)
3. [Processor modes and register banking](#3-processor-modes-and-register-banking)
4. [Exceptions and interrupts](#4-exceptions-and-interrupts)
5. [Instruction set overview](#5-instruction-set-overview)
6. [Opcode map](#6-opcode-map)
7. [Instruction reference](#7-instruction-reference)
8. [Condition codes](#8-condition-codes)
9. [Flag behaviour summary](#9-flag-behaviour-summary)
10. [Undefined encodings](#10-undefined-encodings)
11. [Relationship to ARM Thumb](#11-relationship-to-arm-thumb)
12. [Notes for toolchain implementers](#12-notes-for-toolchain-implementers)
13. [Open questions](#13-open-questions)
14. [SoC-level features (out of scope)](#14-soc-level-features-out-of-scope)
15. [Contributing](#15-contributing)
16. [License](#16-license)

---

## 1. Overview

| Property | Value |
|---|---|
| Word size | 32-bit |
| Endianness | Little endian |
| Instruction width | 16 bits, plus one 32-bit pair (`tjl`) |
| Instruction alignment | 2 bytes |
| General registers | 16 (`r0`-`r15`) |
| Privileged modes used by firmware | SVC, IRQ |
| Instruction sets | One (no ARM/Thumb switching) |
| Multiply | `tmuls`, 32x32 → low 32 bits in-core |
| Division | None |
| Floating point / MMU / MPU | None |
| Reset vector | `0x00000000` |

TC32 is, in practice, a **re-encoded ARM7TDMI Thumb-1 instruction set** with Telink-specific mnemonics (everything is prefixed with `t`), a handful of privileged instructions added for exception handling, and no ARM state. Section [11](#11-relationship-to-arm-thumb) documents the exact correspondence, which is useful if you already know Thumb.

### Notation

- `w` is a 16-bit instruction halfword; bit ranges are written `[hi:lo]`.
- `Rd`, `Rs`, `Rn`, `Rm` are register fields. Unless stated otherwise a 3-bit field selects `r0`-`r7`.
- `imm` fields are zero-extended unless marked `sext`.
- `addr` is the address of the instruction being executed.
- `PC+4` in pc-relative contexts means `addr + 4`.
- Mnemonics are written the way the vendor `tc32-elf-objdump` prints them.

---

## 2. Programmer's model

### 2.1 General registers

| Register | Alias | Notes |
|---|---|---|
| `r0`-`r7` | | "Low" registers. Accessible by every instruction. Not banked. |
| `r8` | | "High" register. Only reachable via `tadd`/`tcmp`/`tmov`/`tjex` high-register forms. Not banked. |
| `r9` | | High register. Not banked. |
| `r10` | `sl` | High register. Not banked. |
| `r11` | `fp` | High register. Not banked. |
| `r12` | `ip` | High register. Not banked. |
| `r13` | `sp` | Stack pointer. **Banked per mode.** |
| `r14` | `lr` | Link register. **Banked per mode.** |
| `r15` | `pc` | Program counter. |

Only `r0`-`r7` can be used as operands in most instructions. `r8`-`r15` are available through the high-register group ([7.5](#75-high-register-operations)), the stack-pointer instructions, and the pc-relative address forms.

### 2.2 Program status register (CPSR)

```
 31  30  29  28  27                    8   7   6   5   4       0
+---+---+---+---+------------------------+---+---+---+-----------+
| N | Z | C | V |        reserved        | I | F | T |   mode    |
+---+---+---+---+------------------------+---+---+---+-----------+
```

| Bit(s) | Name | Meaning |
|---|---|---|
| 31 | N | Negative |
| 30 | Z | Zero |
| 29 | C | Carry (for subtraction: set when **no** borrow occurred) |
| 28 | V | Signed overflow |
| 7 | I | IRQ mask. `1` = IRQ disabled |
| 6 | F | FIQ mask. `1` = FIQ disabled |
| 5 | T | Present for ARM compatibility. Has no effect on decoding; there is only one instruction set |
| 4:0 | mode | Processor mode, see [section 3](#3-processor-modes-and-register-banking) |

Each exception mode has an SPSR that receives a copy of the CPSR on exception entry ([section 4](#4-exceptions-and-interrupts)).

### 2.3 Reset state

- Mode: **SVC** (`0x13`)
- `pc = 0x00000000`
- Banked `sp`/`lr` values are not defined by the architecture; firmware must initialise the stack pointer of every mode it intends to use (see [3.2](#32-initialising-banked-stack-pointers)).

The reset values of CPSR.I and CPSR.F have not been established from the available evidence.

### 2.4 Reading `pc`

- `tloadr Rd, [pc, #imm]` and `tadd Rd, pc, #imm` use `Align(addr + 4, 4)` as their base.
- Direct branches use `addr + 4` as their base ([7.6](#76-branches)).
- The value read when `r15` is used as an operand of a high-register operation (`tmov`, `tadd`, or `tcmp`) has not been established on hardware. Software should not rely on a particular value.

---

## 3. Processor modes and register banking

| Mode | CPSR[4:0] | Used by firmware |
|---|---|---|
| User | `0x10` | No |
| FIQ | `0x11` | No |
| IRQ | `0x12` | **Yes**, hardware interrupts |
| Supervisor | `0x13` | **Yes**, reset / main-loop context |
| Abort | `0x17` | No |
| Undefined | `0x1B` | No |
| System | `0x1F` | No |

Only `r13` (`sp`) and `r14` (`lr`) are banked, one copy per mode. `r0`-`r12` are shared by all modes. On every mode change the outgoing `sp`/`lr` are stored in the outgoing mode's bank and the incoming mode's `sp`/`lr` are loaded.

User, FIQ, Abort, Undefined, and System mode behaviour has not been exercised by the firmware examined so far. ARM7TDMI-like behaviour is a working hypothesis, not a confirmed TC32 property; in particular, FIQ banking of `r8`-`r12` is unresolved.

### 3.1 Changing mode

Writing CPSR with [`tmcsr`](#78-system--privileged-instructions) switches mode and swaps the banked `sp`/`lr` accordingly. The full 32-bit register value is applied: bits `[4:0]` select the mode and every other bit (flags, I, F, T) is taken from the source register.

### 3.2 Initialising banked stack pointers

Because every mode owns its own `sp`, startup code typically:

1. Starts in SVC.
2. Writes CPSR with the IRQ mode value (and the I bit set), loads `sp`.
3. Writes CPSR back with the SVC mode value, loads `sp`.

---

## 4. Exceptions and interrupts

### 4.1 Vector table

TC32 has a **sparse** vector table. Telink images keep image metadata in the words between vectors, so only some slots hold code.

| Address | Vector | Notes |
|---|---|---|
| `0x00` | Reset | Entry point |
| `0x04` | Undefined instruction | ⚠️ not observed in use |
| `0x08` | SWI | **Shares its slot with the image magic tag.** Firmware does not use it as a code vector |
| `0x0C` | Prefetch abort | ⚠️ not observed in use |
| `0x10` | **IRQ** | The single vector firmware actually uses |
| `0x18` | FIQ | ⚠️ not observed in use |

All other words in the first bytes of an image are metadata and must not be assumed to be code.

### 4.2 Exception entry (IRQ)

When an interrupt is taken, between instructions:

1. `SPSR_irq ← CPSR`
2. The current mode's `sp`/`lr` are saved to its bank.
3. Mode ← IRQ, **CPSR.I ← 1** (further IRQs masked). N/Z/C/V and the other control bits are left unchanged.
4. The IRQ bank's `sp`/`lr` are loaded.
5. `lr ← address of the next instruction that would have executed` (no `-4` fix-up is applied in hardware; software returns with the unadjusted value, see [4.4](#44-returning-with-treti)).
6. `pc ← 0x10`

An IRQ is accepted only when **CPSR.I = 0** and the core is **not already in IRQ mode**.

### 4.3 Software interrupt (`tserv`)

`tserv #imm8` enters SVC mode through the SWI vector with `lr = addr + 2` and `SPSR_svc = CPSR`, I set. Because the SWI slot (`0x08`) is occupied by the image tag on standard Telink images, `tserv` is not useful as a vectored call on stock firmware layouts.

### 4.4 Returning with `treti`

`treti {reglist}` is the exception-return instruction. In one operation it:

1. Pops the listed low registers from the current stack (ascending register order).
2. If bit 8 is set, pops one more word as the return address (the "`pc`" slot).
3. Restores `CPSR ← SPSR` of the current mode and switches register banks accordingly. The exception-mode `sp` ends up pointing just past the popped data.
4. If bit 8 was set, branches to the popped address (bit 0 cleared).

If no SPSR has been established for the current mode, the CPSR is not restored and only the stack/branch behaviour applies.

A typical IRQ handler therefore pushes `lr` and the registers it clobbers on entry, and ends with a single `treti` that unwinds them and returns to the interrupted context.

### 4.5 Undefined instructions

⚠️ The behaviour on silicon for an undefined encoding is **unknown**. It should not be assumed to trap through `0x04`.

---

## 5. Instruction set overview

### 5.1 Instruction formats

All instructions are 16 bits, except `tjl` which is a pair of 16-bit halfwords.

### 5.2 Mnemonic conventions

- All mnemonics start with `t`.
- A trailing `s` means the instruction updates the condition flags (`tadds`, `tsubs`, `tmovs`, ...).
- `tload*` / `tstore*` are loads / stores; `r` stands for "register".
- `tj*` are jumps (`tj`, `tjeq`, `tjl`, `tjex`).

### 5.3 Instruction groups

| Group | Instructions |
|---|---|
| Data processing, register | `tands` `txors` `tshftls` `tshftrs` `tasrs` `taddcs` `tsubcs` `trotrs` `tnand` `tnegs` `tcmp` `tcmpn` `tors` `tmuls` `tbclrs` `tmovns` |
| Data processing, immediate | `tmovs` `tcmp` `tadds` `tsubs` (imm8 and imm3 forms) |
| Shifts, immediate | `tshftls` `tshftrs` `tasrs` |
| High-register | `tadd` `tcmp` `tmov` `tnop` |
| Branch | `tj` `tj<cc>` `tjl` `tjex` |
| Load / store | `tloadr` `tloadrb` `tloadrh` `tloadrsb` `tloadrsh` `tstorer` `tstorerb` `tstorerh` |
| Block / stack | `tpush` `tpop` `tstorem` `tloadm` `tadd sp` `tsub sp` |
| System | `tserv` `treti` `tmcsr` `tmrcs` `tmssr` `tmrss` |
| Address generation | `tadd Rd, pc, #imm` `tadd Rd, sp, #imm` |

---

## 6. Opcode map

A complete, non-overlapping map of the 16-bit encoding space. Ranges are inclusive-exclusive on the upper end (`0x0000-0x0400` covers `0x0000`..`0x03FF`).

| Range | Instruction | Section |
|---|---|---|
| `0x0000-0x0400` | ALU register operations | [7.1](#71-alu-register-operations) |
| `0x0400-0x0500` | `tadd Rd, Rs` (high registers) | [7.5](#75-high-register-operations) |
| `0x0500-0x0600` | `tcmp Rd, Rs` (high registers) | [7.5](#75-high-register-operations) |
| `0x0600-0x0700` | `tmov Rd, Rs` (high registers), `tnop` | [7.5](#75-high-register-operations) |
| `0x0700-0x0800` | `tjex Rs` | [7.6](#76-branches) |
| `0x0800-0x1000` | `tloadr Rd, [pc, #imm]` | [7.7](#77-load-and-store) |
| `0x1000-0x2000` | Load/store, register offset | [7.7](#77-load-and-store) |
| `0x2000-0x2800` | `tstorerh Rd, [Rn, #imm]` | [7.7](#77-load-and-store) |
| `0x2800-0x3000` | `tloadrh Rd, [Rn, #imm]` | [7.7](#77-load-and-store) |
| `0x3000-0x3800` | `tstorer Rd, [sp, #imm]` | [7.7](#77-load-and-store) |
| `0x3800-0x4000` | `tloadr Rd, [sp, #imm]` | [7.7](#77-load-and-store) |
| `0x4000-0x4800` | `tstorerb Rd, [Rn, #imm]` | [7.7](#77-load-and-store) |
| `0x4800-0x5000` | `tloadrb Rd, [Rn, #imm]` | [7.7](#77-load-and-store) |
| `0x5000-0x5800` | `tstorer Rd, [Rn, #imm]` | [7.7](#77-load-and-store) |
| `0x5800-0x6000` | `tloadr Rd, [Rn, #imm]` | [7.7](#77-load-and-store) |
| `0x6000-0x6100` | `tadd sp, #imm` / `tsub sp, #imm` | [7.9](#79-stack-instructions) |
| `0x6100-0x6400` | *undefined* | [10](#10-undefined-encodings) |
| `0x6400-0x6600` | `tpush` | [7.9](#79-stack-instructions) |
| `0x6600-0x6800` | *undefined* | |
| `0x6800-0x6A00` | `treti` | [7.8](#78-system--privileged-instructions) |
| `0x6A00-0x6B00` | *undefined* | |
| `0x6B00-0x6C00` | `tmcsr` / `tmrcs` / `tmssr` / `tmrss` at `0x6BC0-0x6BE0`, rest *undefined* | [7.8](#78-system--privileged-instructions) |
| `0x6C00-0x6E00` | `tpop` | [7.9](#79-stack-instructions) |
| `0x6E00-0x7000` | *undefined* | |
| `0x7000-0x7800` | `tadd Rd, pc, #imm` | [7.7.4](#774-address-generation) |
| `0x7800-0x8000` | `tadd Rd, sp, #imm` | [7.7.4](#774-address-generation) |
| `0x8000-0x8800` | `tj` (unconditional) | [7.6](#76-branches) |
| `0x8800-0x9000` | *undefined* | |
| `0x9000-0x9800` | `tjl`, first halfword | [7.6](#76-branches) |
| `0x9800-0xA000` | `tjl`, second halfword | [7.6](#76-branches) |
| `0xA000-0xA800` | `tmovs Rd, #imm8` | [7.3](#73-immediate-data-processing) |
| `0xA800-0xB000` | `tcmp Rd, #imm8` | [7.3](#73-immediate-data-processing) |
| `0xB000-0xB800` | `tadds Rd, #imm8` | [7.3](#73-immediate-data-processing) |
| `0xB800-0xC000` | `tsubs Rd, #imm8` | [7.3](#73-immediate-data-processing) |
| `0xC000-0xCE00` | `tj<cc>` (conditional) | [7.6](#76-branches) |
| `0xCE00-0xCF00` | *undefined* | |
| `0xCF00-0xD000` | `tserv #imm8` | [7.8](#78-system--privileged-instructions) |
| `0xD000-0xD800` | `tstorem Rn!, {list}` | [7.9](#79-stack-instructions) |
| `0xD800-0xE000` | `tloadm Rn!, {list}` | [7.9](#79-stack-instructions) |
| `0xE000-0xE800` | `tasrs Rd, Rs, #imm` | [7.2](#72-immediate-shifts) |
| `0xE800-0xEA00` | `tadds Rd, Rn, Rm` | [7.4](#74-three-operand-add--subtract) |
| `0xEA00-0xEC00` | `tsubs Rd, Rn, Rm` | [7.4](#74-three-operand-add--subtract) |
| `0xEC00-0xEE00` | `tadds Rd, Rn, #imm3` | [7.4](#74-three-operand-add--subtract) |
| `0xEE00-0xF000` | `tsubs Rd, Rn, #imm3` | [7.4](#74-three-operand-add--subtract) |
| `0xF000-0xF040` | `tmovs Rd, Rs` (low registers) | [7.4.1](#741-register-move-low-registers) |
| `0xF040-0xF800` | `tshftls Rd, Rs, #imm` | [7.2](#72-immediate-shifts) |
| `0xF800-0x10000` | `tshftrs Rd, Rs, #imm` | [7.2](#72-immediate-shifts) |

### Quick view by top nibble

| `w[15:12]` | Contents |
|---|---|
| `0x0` | ALU, high-register ops, `tjex`, pc-relative load |
| `0x1` | Load/store, register offset |
| `0x2` | Halfword load/store, immediate offset |
| `0x3` | SP-relative word load/store |
| `0x4` | Byte load/store, immediate offset |
| `0x5` | Word load/store, immediate offset |
| `0x6` | SP adjust, `tpush`, `treti`, PSR moves, `tpop` |
| `0x7` | `tadd Rd, pc/sp, #imm` |
| `0x8` | `tj` |
| `0x9` | `tjl` |
| `0xA` | `tmovs`/`tcmp` immediate |
| `0xB` | `tadds`/`tsubs` immediate |
| `0xC` | `tj<cc>`, `tserv` |
| `0xD` | `tstorem`, `tloadm` |
| `0xE` | `tasrs` immediate, three-operand add/sub |
| `0xF` | `tshftls` / `tshftrs` immediate, low-register `tmovs` |

---

## 7. Instruction reference

### 7.1 ALU register operations

```
 15      10  9     6  5     3  2     0
+----------+--------+--------+--------+
| 0 0 0 0 0 0|  op   |   Rs   |   Rd   |
+----------+--------+--------+--------+
```

Range `0x0000`-`0x03FF`. `Rd` is both a source and the destination unless noted.

| op | Mnemonic | Operation | Flags |
|---|---|---|---|
| 0 | `tands Rd, Rs` | `Rd ← Rd & Rs` | N Z |
| 1 | `txors Rd, Rs` | `Rd ← Rd ^ Rs` | N Z |
| 2 | `tshftls Rd, Rs` | `Rd ← Rd << (Rs & 0xFF)` | N Z C |
| 3 | `tshftrs Rd, Rs` | `Rd ← Rd >> (Rs & 0xFF)` (logical) | N Z C |
| 4 | `tasrs Rd, Rs` | `Rd ← Rd >> (Rs & 0xFF)` (arithmetic) | N Z C |
| 5 | `taddcs Rd, Rs` | `Rd ← Rd + Rs + C` | N Z C V |
| 6 | `tsubcs Rd, Rs` | `Rd ← Rd - Rs - (1 - C)` | N Z C V |
| 7 | `trotrs Rd, Rs` | `Rd ← Rd ROR (Rs & 0xFF)` | N Z C |
| 8 | `tnand Rd, Rs` | flags ← `Rd & Rs`; **no register write** | N Z |
| 9 | `tnegs Rd, Rs` | `Rd ← 0 - Rs` | N Z C V |
| 10 | `tcmp Rd, Rs` | flags ← `Rd - Rs`; no register write | N Z C V |
| 11 | `tcmpn Rd, Rs` | flags ← `Rd + Rs`; no register write | N Z C V |
| 12 | `tors Rd, Rs` | `Rd ← Rd \| Rs` | N Z |
| 13 | `tmuls Rd, Rs` | `Rd ← (Rd * Rs)[31:0]` | N Z |
| 14 | `tbclrs Rd, Rs` | `Rd ← Rd & ~Rs` | N Z |
| 15 | `tmovns Rd, Rs` | `Rd ← ~Rs` | N Z |

Notes:

- **`tnand`** is the vendor's name for the *test* operation: it computes `Rd & Rs`, sets N and Z, and discards the result (the behaviour of ARM `TST`). Despite the name it is **not** a NAND.
- **Register-specified shifts and rotates** use only the low 8 bits of `Rs`:
  - amount `0`: the value is unchanged, N/Z are set from it, and C/V are preserved.
  - amount `1`-`31`: the standard shift/rotate result is written; C receives the last bit shifted out.
  - For `trotrs`, a non-zero amount divisible by 32 leaves the value unchanged and sets C to the original bit 31. Other amounts greater than 32 are equivalent to the corresponding rotation by `amount mod 32`; V is preserved.
  - For `tshftls`, `tshftrs`, and `tasrs`, carry behaviour for amounts `≥ 32` remains an open hardware question; see [13](#13-open-questions).
- **`tmuls`**: the in-core result is the low 32 bits only. The high 32 bits are not in the register file; see [14](#14-soc-level-features-out-of-scope).
- `tsubcs`, `tcmp`, `tnegs`: **C = 1 means no borrow** (ARM convention).
- Logical operations and `tmuls` leave C and V unchanged.

### 7.2 Immediate shifts

```
 15      11 10     6  5     3  2     0
+---------+---------+--------+--------+
| opcode  |  imm5   |   Rs   |   Rd   |
+---------+---------+--------+--------+
```

| Range | Mnemonic | Operation |
|---|---|---|
| `0xE000-0xE7FF` | `tasrs Rd, Rs, #imm` | arithmetic shift right |
| `0xF040-0xF7FF` | `tshftls Rd, Rs, #imm` | logical shift left, `imm = 1..31` |
| `0xF800-0xFFFF` | `tshftrs Rd, Rs, #imm` | logical shift right |

- For `tasrs` and `tshftrs`, an encoded `imm5` of `0` means a shift by **32**.
- For `tshftls`, `imm5 = 0` is not a shift: that encoding space (`0xF000-0xF03F`) is the low-register move, see [7.4.1](#741-register-move-low-registers).
- Flags: N Z C. V is preserved.

### 7.3 Immediate data processing

```
 15      11 10     8  7             0
+---------+---------+---------------+
| opcode  |   Rd    |     imm8      |
+---------+---------+---------------+
```

| Range | Mnemonic | Operation | Flags |
|---|---|---|---|
| `0xA000-0xA7FF` | `tmovs Rd, #imm8` | `Rd ← imm8` | N Z |
| `0xA800-0xAFFF` | `tcmp Rd, #imm8` | flags ← `Rd - imm8` | N Z C V |
| `0xB000-0xB7FF` | `tadds Rd, #imm8` | `Rd ← Rd + imm8` | N Z C V |
| `0xB800-0xBFFF` | `tsubs Rd, #imm8` | `Rd ← Rd - imm8` | N Z C V |

### 7.4 Three-operand add / subtract

```
 15         9  8     6  5     3  2     0
+------------+--------+--------+--------+
|   opcode   | Rm/imm3|   Rn   |   Rd   |
+------------+--------+--------+--------+
```

| Range | Mnemonic | Operation | Flags |
|---|---|---|---|
| `0xE800-0xE9FF` | `tadds Rd, Rn, Rm` | `Rd ← Rn + Rm` | N Z C V |
| `0xEA00-0xEBFF` | `tsubs Rd, Rn, Rm` | `Rd ← Rn - Rm` | N Z C V |
| `0xEC00-0xEDFF` | `tadds Rd, Rn, #imm3` | `Rd ← Rn + imm3` | N Z C V |
| `0xEE00-0xEFFF` | `tsubs Rd, Rn, #imm3` | `Rd ← Rn - imm3` | N Z C V |

#### 7.4.1 Register move, low registers

```
 15              6  5     3  2     0
+------------------+--------+--------+
| 1 1 1 1 0 0 0 0 0 0 |  Rs  |   Rd   |   (0xF000-0xF03F)
+------------------+--------+--------+
```

`tmovs Rd, Rs`: `Rd ← Rs`.

Flag behaviour of this instruction remains unresolved; see [13](#13-open-questions).

### 7.5 High-register operations

```
 15      10  9   8   7   6  5     3  2     0
+----------+---+---+---+---+--------+--------+
|  opcode  | 0 | 0 |H1 |H2 |  Rs    |  Rd    |
+----------+---+---+---+---+--------+--------+
```

- `Rd = H1 * 8 + w[2:0]` (bit 7 selects `r8`-`r15`)
- `Rs = H2 * 8 + w[5:3]` (bit 6 selects `r8`-`r15`)

| Range | Mnemonic | Operation | Flags |
|---|---|---|---|
| `0x0400-0x04FF` | `tadd Rd, Rs` | `Rd ← Rd + Rs` | **none** |
| `0x0500-0x05FF` | `tcmp Rd, Rs` | flags ← `Rd - Rs` | N Z C V |
| `0x0600-0x06FF` | `tmov Rd, Rs` | `Rd ← Rs` | **none** |

- Any of these may target `r15`, which performs a branch (`tadd pc, Rs`, `tmov pc, Rs`). The target is used as given.
- **`tnop`** is `tmov r8, r8`, encoding **`0x06C0`**.
- Mixed encodings with both H bits clear (`0x0400`-`0x043F` etc.) decode as low-register forms of the same operation.

### 7.6 Branches

#### `tj`: unconditional branch

```
 15      11 10                 0
+---------+---------------------+
| 1 0 0 0 0|       imm11        |   (0x8000-0x87FF)
+---------+---------------------+
```

`target = addr + 4 + (sext(imm11) << 1)`. Range: −2048 … +2046 bytes from `addr + 4`.

#### `tj<cc>`: conditional branch

```
 15     12 11     8  7             0
+---------+---------+---------------+
| 1 1 0 0 |  cond   |     imm8      |   (0xC000-0xCDFF)
+---------+---------+---------------+
```

`target = addr + 4 + (sext(imm8) << 1)` if the condition holds. Range: −256 … +254 bytes from `addr + 4`. `cond` values `14` and `15` are not branches (`0xCE00-0xCEFF` is undefined). See [section 8](#8-condition-codes).

#### `tjl`: branch with link (32-bit)

```
 first halfword  (0x9000-0x97FF):   1 0 0 1 0 | imm11[21:11]
 second halfword (0x9800-0x9FFF):   1 0 0 1 1 | imm11[10:0]
```

```
offset = sext22((hi11 << 11) | lo11) << 1
lr     ← addr + 4        (address of the instruction after the pair)
target = addr + 4 + offset
```

Range: ±4 MiB. The two halfwords must be adjacent; a first half not followed by a valid second half, or a lone second half, is undefined. `lr` receives the exact return address with **no** Thumb-style bit-0 tag.

#### `tjex`: branch to register

```
 15              7  6     3  2     0
+------------------+--------+--------+
| 0 0 0 0 0 1 1 1 x|   Rs   | 0 0 0  |   (0x0700-0x07FF)
+------------------+--------+--------+
```

`pc ← Rs & ~1`. `Rs` is `r0`-`r15` (4-bit field `w[6:3]`). There is **no link variant** and no instruction-set switch; bit 0 is simply cleared. In `0x0700-0x077F` the low three bits are ignored by the vendor decoder; in `0x0780-0x07FF` they must be zero (otherwise the encoding is undefined). Returns are normally `tjex lr` or `tpop {..., pc}`.

### 7.7 Load and store

All accesses are little endian.

#### 7.7.1 Register offset

```
 15      9  8     6  5     3  2     0
+---------+--------+--------+--------+
|  kind   |   Rm   |   Rn   |   Rd   |   (0x1000-0x1FFF)
+---------+--------+--------+--------+
```

`address = Rn + Rm`. `kind = w[15:9]` (values `0x08`..`0x0F`):

| kind | Range | Mnemonic | Width | Direction |
|---|---|---|---|---|
| `0x08` | `0x1000-0x11FF` | `tstorer Rd, [Rn, Rm]` | word | store |
| `0x09` | `0x1200-0x13FF` | `tstorerh Rd, [Rn, Rm]` | half | store |
| `0x0A` | `0x1400-0x15FF` | `tstorerb Rd, [Rn, Rm]` | byte | store |
| `0x0B` | `0x1600-0x17FF` | `tloadrsb Rd, [Rn, Rm]` | byte, sign-extended | load |
| `0x0C` | `0x1800-0x19FF` | `tloadr Rd, [Rn, Rm]` | word | load |
| `0x0D` | `0x1A00-0x1BFF` | `tloadrh Rd, [Rn, Rm]` | half, zero-extended | load |
| `0x0E` | `0x1C00-0x1DFF` | `tloadrb Rd, [Rn, Rm]` | byte, zero-extended | load |
| `0x0F` | `0x1E00-0x1FFF` | `tloadrsh Rd, [Rn, Rm]` | half, sign-extended | load |

#### 7.7.2 Immediate offset

```
 15      11 10     6  5     3  2     0
+---------+---------+--------+--------+
| opcode  |  imm5   |   Rn   |   Rd   |
+---------+---------+--------+--------+
```

| Range | Mnemonic | Offset |
|---|---|---|
| `0x2000-0x27FF` | `tstorerh Rd, [Rn, #imm]` | `imm5 * 2` |
| `0x2800-0x2FFF` | `tloadrh Rd, [Rn, #imm]` | `imm5 * 2` |
| `0x4000-0x47FF` | `tstorerb Rd, [Rn, #imm]` | `imm5` |
| `0x4800-0x4FFF` | `tloadrb Rd, [Rn, #imm]` | `imm5` |
| `0x5000-0x57FF` | `tstorer Rd, [Rn, #imm]` | `imm5 * 4` |
| `0x5800-0x5FFF` | `tloadr Rd, [Rn, #imm]` | `imm5 * 4` |

There are no signed-load forms with an immediate offset.

#### 7.7.3 SP-relative and PC-relative (word)

```
 15      11 10     8  7             0
+---------+---------+---------------+
| opcode  |   Rd    |     imm8      |
+---------+---------+---------------+
```

| Range | Mnemonic | Address |
|---|---|---|
| `0x0800-0x0FFF` | `tloadr Rd, [pc, #imm]` | `Align(addr + 4, 4) + imm8 * 4` |
| `0x3000-0x37FF` | `tstorer Rd, [sp, #imm]` | `sp + imm8 * 4` |
| `0x3800-0x3FFF` | `tloadr Rd, [sp, #imm]` | `sp + imm8 * 4` |

The pc-relative load is the standard literal-pool access; the pool must be within 1020 bytes **after** the instruction.

#### 7.7.4 Address generation

| Range | Mnemonic | Operation |
|---|---|---|
| `0x7000-0x77FF` | `tadd Rd, pc, #imm` | `Rd ← Align(addr + 4, 4) + imm8 * 4` |
| `0x7800-0x7FFF` | `tadd Rd, sp, #imm` | `Rd ← sp + imm8 * 4` |

Layout is as in 7.7.3 (`Rd` in `[10:8]`, `imm8` in `[7:0]`). No flags are changed.

### 7.8 System / privileged instructions

#### `tserv`: software interrupt

`0xCF00 | imm8`: see [4.3](#43-software-interrupt-tserv).

#### `treti`: return from exception

```
 15           9  8  7             0
+--------------+---+---------------+
| 0 1 1 0 1 0 0 | P |   reg list   |   (0x6800-0x69FF)
+--------------+---+---------------+
```

`P` (bit 8) = also pop the return address. `reg list` = `r0`-`r7`. Semantics in [4.4](#44-returning-with-treti).

#### PSR transfer

All four live in `0x6BC0-0x6BDF`:

```
 15                      5  4   3  2     0
+--------------------------+-------+--------+
| 0 1 1 0 1 0 1 1 1 1 0     |  sel  |   Rn   |
+--------------------------+-------+--------+
```

| Encoding | Mnemonic | Operation |
|---|---|---|
| `0x6BC0 \| Rn` | `tmcsr Rn` | `CPSR ← Rn` (mode switch, bank swap, flags and I/F/T replaced) |
| `0x6BC8 \| Rn` | `tmrcs Rn` | `Rn ← CPSR` |
| `0x6BD0 \| Rn` | `tmssr Rn` | `SPSR(current mode) ← Rn` |
| `0x6BD8 \| Rn` | `tmrss Rn` | `Rn ← SPSR(current mode)` |

`Rn` is `r0`-`r7`. Privilege checks (for example, executing `tmcsr` from User mode) have not been characterised.

### 7.9 Stack instructions

#### `tadd sp` / `tsub sp`

```
 15           8  7  6           0
+--------------+---+--------------+
| 0 1 1 0 0 0 0 0| S |    imm7     |   (0x6000-0x60FF)
+--------------+---+--------------+
```

`S = 0`: `sp ← sp + imm7 * 4` (`tadd sp, #imm`). `S = 1`: `sp ← sp - imm7 * 4` (`tsub sp, #imm`). No flags.

#### `tpush`

```
 15            9  8  7             0
+--------------+---+---------------+
| 0 1 1 0 0 1 0 | L |   reg list   |   (0x6400-0x65FF)
+--------------+---+---------------+
```

Pushes the listed low registers, plus `lr` if `L = 1`. `sp` is decremented by `4 * count` first; registers are stored in ascending order from the new `sp`, with `lr` at the highest address (full-descending stack).

#### `tpop`

```
 15            9  8  7             0
+--------------+---+---------------+
| 0 1 1 0 1 1 0 | P |   reg list   |   (0x6C00-0x6DFF)
+--------------+---+---------------+
```

Pops the listed low registers in ascending order; if `P = 1`, pops one more word into `pc` (bit 0 cleared, no mode change), i.e. a function return. `sp` is incremented past all popped words.

#### `tstorem` / `tloadm`

```
 15      11 10     8  7             0
+---------+---------+---------------+
| opcode  |   Rn    |   reg list    |
+---------+---------+---------------+
```

| Range | Mnemonic | Operation |
|---|---|---|
| `0xD000-0xD7FF` | `tstorem Rn!, {list}` | store ascending from `Rn`, then `Rn ← Rn + 4*count` |
| `0xD800-0xDFFF` | `tloadm Rn!, {list}` | load ascending from `Rn`, then `Rn ← Rn + 4*count` |

Write-back always applies when the list is non-empty. An empty list performs no access. Behaviour when `Rn` appears in a `tloadm` list remains unresolved.

### 7.10 Encoding examples

Derived from the rules above:

| Encoding | Assembly |
|---|---|
| `0x06C0` | `tnop` |
| `0xA001` | `tmovs r1, #1` |
| `0x0770` | `tjex lr` |
| `0x6510` | `tpush {r4, lr}` |
| `0x6D10` | `tpop {r4, pc}` |
| `0x6900` | `treti {pc}` |
| `0x6BD8` | `tmrss r0` |
| `0x6BC1` | `tmcsr r1` |
| `0x0804` | `tloadr r0, [pc, #4]` |
| `0xCF01` | `tserv #1` |
| `0x8000` | `tj .+4` |

---

## 8. Condition codes

Identical to ARM, except that only the 14 conditions below are valid in `tj<cc>`.

| cond | Suffix | Meaning | Flag test |
|---|---|---|---|
| 0 | `eq` | Equal | Z |
| 1 | `ne` | Not equal | !Z |
| 2 | `cs` | Carry set / unsigned ≥ | C |
| 3 | `cc` | Carry clear / unsigned < | !C |
| 4 | `mi` | Negative | N |
| 5 | `pl` | Positive or zero | !N |
| 6 | `vs` | Overflow | V |
| 7 | `vc` | No overflow | !V |
| 8 | `hi` | Unsigned > | C && !Z |
| 9 | `ls` | Unsigned ≤ | !C \|\| Z |
| 10 | `ge` | Signed ≥ | N == V |
| 11 | `lt` | Signed < | N != V |
| 12 | `gt` | Signed > | !Z && N == V |
| 13 | `le` | Signed ≤ | Z \|\| N != V |
| 14, 15 | n/a | Not encodable in `tj<cc>` | n/a |

Mnemonics: `tjeq tjne tjcs tjcc tjmi tjpl tjvs tjvc tjhi tjls tjge tjlt tjgt tjle`.

---

## 9. Flag behaviour summary

| Instruction class | N | Z | C | V |
|---|---|---|---|---|
| `tands` `txors` `tors` `tbclrs` `tmovns` `tnand` | ✔ | ✔ | – | – |
| `tmuls` | ✔ | ✔ | – | – |
| `tmovs Rd, #imm8` | ✔ | ✔ | – | – |
| Shifts / rotates (all forms) | ✔ | ✔ | ✔ | – |
| `taddcs` `tsubcs` `tnegs` `tcmp` `tcmpn` | ✔ | ✔ | ✔ | ✔ |
| `tadds` `tsubs` (all forms) | ✔ | ✔ | ✔ | ✔ |
| `tcmp` (imm8 and high-register) | ✔ | ✔ | ✔ | ✔ |
| `tadd` / `tmov` (high register) | – | – | – | – |
| `tadd`/`tsub sp`, `tadd Rd, pc/sp, #imm` | – | – | – | – |
| Loads, stores, push, pop, branches | – | – | – | – |

`–` means the flag is unchanged. Carry on addition is the unsigned carry-out; carry on subtraction is the inverted borrow.

---

## 10. Undefined encodings

The following halfword ranges are not produced by the vendor assembler and have no known defined behaviour:

| Range | Notes |
|---|---|
| `0x6100-0x63FF` | |
| `0x6600-0x67FF` | |
| `0x6A00-0x6BBF`, `0x6BE0-0x6BFF` | Gap around the PSR-transfer instructions |
| `0x6E00-0x6FFF` | |
| `0x07xx` with `w[2:0] ≠ 0` and `w ≥ 0x0780` | Invalid `tjex` |
| `0x8800-0x8FFF` | |
| `0x9000-0x97FF` not followed by `0x9800-0x9FFF` | Orphaned `tjl` prefix |
| `0x9800-0x9FFF` not preceded by `0x9000-0x97FF` | Orphaned `tjl` suffix |
| `0xCE00-0xCEFF` | Would be condition 14 |

The silicon response to these encodings (for example, exception, NOP, or another behaviour) is not documented. Software must not rely on any particular response.

---

## 11. Relationship to ARM Thumb

TC32 is a **bijective remap of Thumb-1** at the level of the top instruction nibble. Each Thumb instruction class has the same field layout as its TC32 counterpart; only `w[15:12]` differs. This is useful when porting Thumb tooling, and makes the encoding easy to remember:

| Thumb `w[15:12]` | TC32 `w[15:12]` | Instruction class |
|---|---|---|
| `0x0` | `0xF` | LSL / LSR immediate |
| `0x1` | `0xE` | ASR immediate, ADD/SUB register and imm3 |
| `0x2` | `0xA` | MOV / CMP imm8 |
| `0x3` | `0xB` | ADD / SUB imm8 |
| `0x4` | `0x0` | ALU, high-register ops, BX, LDR literal |
| `0x5` | `0x1` | Load/store register offset |
| `0x6` | `0x5` | Word load/store, immediate |
| `0x7` | `0x4` | Byte load/store, immediate |
| `0x8` | `0x2` | Halfword load/store, immediate |
| `0x9` | `0x3` | SP-relative word load/store |
| `0xA` | `0x7` | ADD Rd, PC/SP, #imm |
| `0xB` | `0x6` | SP adjust, PUSH, POP (+ new system instructions) |
| `0xC` | `0xD` | STMIA / LDMIA |
| `0xD` | `0xC` | Conditional branch / SWI |
| `0xE` | `0x8` | Unconditional branch |
| `0xF` | `0x9` | BL pair |

### Mnemonic correspondence

| TC32 | ARM Thumb |
|---|---|
| `tands` `txors` `tors` `tbclrs` `tmovns` `tmuls` | `ANDS` `EORS` `ORRS` `BICS` `MVNS` `MULS` |
| `tshftls` `tshftrs` `tasrs` `trotrs` | `LSLS` `LSRS` `ASRS` `RORS` |
| `taddcs` `tsubcs` | `ADCS` `SBCS` |
| `tnand` | `TST` |
| `tnegs` | `NEGS` / `RSBS #0` |
| `tcmp` `tcmpn` | `CMP` `CMN` |
| `tmovs` `tadds` `tsubs` | `MOVS` `ADDS` `SUBS` |
| `tmov` `tadd` (high) | `MOV` `ADD` (high registers) |
| `tj` `tj<cc>` `tjl` `tjex` | `B` `B<cc>` `BL` `BX` |
| `tloadr*` `tstorer*` | `LDR*` `STR*` |
| `tpush` `tpop` `tloadm` `tstorem` | `PUSH` `POP` `LDMIA` `STMIA` |
| `tserv` | `SVC` / `SWI` |

### Differences from Thumb-1

- **No ARM state, no interworking.** `tjex` clears bit 0 and nothing else; `tpop {pc}` likewise.
- **No `BLX`.** The `0x0780`-`0x07FF` range of `tjex` does not link.
- **`tjl` and `lr`:** `lr` receives the exact return address, **without** the Thumb bit-0 tag.
- **No `CPS`, `BKPT`, sign/zero-extend (`SXTB`, `UXTB`, ...), `REV*`** or other ARMv6-M additions.
- **New instructions:** `treti`, `tmcsr`, `tmrcs`, `tmssr`, `tmrss`, placed in unused Thumb space (`0xB8xx` and `0xBBxx` equivalents).
- **Immediate shift `#0`:** LSL `#0` is repurposed as the low-register `tmovs Rd, Rs`; LSR/ASR `#0` mean `#32`, as in ARM.
- **`tnop`** is `0x06C0` (Thumb's `0x46C0`).
- **Banking:** only `sp` and `lr` are banked.

---

## 12. Notes for toolchain implementers

Things worth knowing when writing an assembler, disassembler, or compiler backend:

- **Two register classes.** Almost every data-processing instruction only reaches `r0`-`r7`. `r8`-`r12` are reachable only through `tmov`/`tadd`/`tcmp` high forms (and `tjex`). Treat `r8`-`r12` as reload-expensive.
- **Flags are set almost everywhere on low-register ALU forms** (`tands`, `tadds`, `tsubs`, ...), while the high-register forms and SP/address forms do not touch them. There is no non-flag-setting low-register add/sub.
- **Branch ranges:** `tj<cc>` ±256 B, `tj` ±2 KiB, `tjl` ±4 MiB. Out-of-range conditional branches need an inverted `tj<cc>` over a `tj`/`tjl`.
- **Literal pools:** `tloadr Rd, [pc, #imm]` reaches only 0-1020 bytes forward, word aligned relative to `Align(addr + 4, 4)`.
- **No indirect call instruction.** Indirect calls must be synthesised (`tmov lr, ...` followed by `tjex`, taking care to set `lr` to the exact return address with **no** tag bit).
- **No hardware divide**; multiply is 32x32 → 32 in-core.
- **Instruction fetch is 16-bit.** `tjl` is the only 32-bit encoding.
- **Callee/caller register conventions** and the **object file ABI** are vendor-defined and are not covered by this document.

---

## 13. Open questions

The following behaviours remain unresolved from the available firmware and toolchain evidence. Hardware test results are welcome; please include the exact SoC, test sequence, and observed register/flag values.

| # | Topic | Current evidence / unresolved behaviour |
|---|---|---|
| 1 | **Carry-out for shifts of 32 bits or more** | The expected value results follow ARM-style shift semantics (LSL/LSR → 0, ASR → sign fill). TC32 carry behaviour for these cases has not been established. In particular, the carry for `LSR #32` (immediate or register) may be bit 31 of the source, as on ARM7TDMI, or may differ. This item does not apply to `trotrs`, whose multiple-of-32 behaviour is specified in §7.1. |
| 2 | **Flags on low-register `tmovs Rd, Rs`** (`0xF000-0xF03F`) | The mnemonic suggests N/Z are updated (as in Thumb); the model in use does not update flags. A hardware test is required to resolve this. |
| 3 | **Reading `r15` via high-register ops** | The value read (`addr + 2` vs `addr + 4`) is unconfirmed. |
| 4 | **Undefined encodings** | The response may be an exception, a NOP, or another implementation-defined behaviour. |
| 5 | **`tloadm` with `Rn` in the list** | The precedence between base-register write-back and a loaded value is unknown. |
| 6 | **FIQ, Abort, Undefined, User, System modes** | These modes have not been exercised; their register-banking behaviour remains unknown. |
| 7 | **Reset values of CPSR.I / CPSR.F** | Not established from the available evidence. |
| 8 | **IRQ `lr` convention** | Observed: handlers return by popping the unadjusted `lr` straight into `pc` via `treti`, consistent with `lr = address of next instruction`. Whether the hardware latches `+4` and the vendor startup code compensates has not been independently confirmed. |
| 9 | **Misaligned accesses** | Alignment faults / rotation / byte-lane behaviour unknown. |
| 10 | **Privilege checks** | `tmcsr` / `tmssr` from User mode untested. |
| 11 | **Cycle timing / pipeline** | Not characterised by the available tests. |
| 12 | **Write-back of `tstorem` with `Rn` in the list** | The stored value of `Rn` is currently assumed to be the original value; this has not been confirmed. |

---


## 14. SoC-level features (out of scope)

This document describes the CPU core only. Several behaviours that firmware sees as "part of the CPU" actually live in the SoC:

- **Memory map**, flash/XIP window, SRAM, retention RAM: SoC-specific.
- **Interrupt controller:** how interrupt sources are routed to the single IRQ vector, masked, and acknowledged.
- **Peripherals:** GPIO, timers, RF, UART/SPI/I²C, ADC, etc.
- **Hardware multiplier high word:** `tmuls` writes only the low 32 bits to `Rd`. On the **TLSR8232** the upper 32 bits are exposed through a memory-mapped register at **`0x6FC`**, which the vendor's 32x32→64 multiply helper reads after `tmuls`. Other TLSR82xx parts may differ.

Per-SoC documentation (memory maps, register tables) is planned as separate files alongside this reference.

---

## 15. Contributing

- **Hardware results are the most valuable contribution**, especially for the items in [section 13](#13-open-questions). Please include the chip part number, the test code (or its hexdump), and observed register/flag values.
- Corrections to encodings: please cite the vendor disassembly output that disagrees.
- New SoC data (memory maps, peripheral registers): open an issue first so the file layout stays consistent.

## 16. License

This document is dedicated to the public domain under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/). To the extent possible under law, the OpenTC32 contributors have waived all copyright and related or neighboring rights to this work. You may copy, modify, and redistribute it for any purpose, including commercial use, without asking permission.
