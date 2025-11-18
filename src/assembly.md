# Assembly Syntax

## Operands

The following operand syntax types are used in instructions

| Short   | Name             | Description                              |
|---------|------------------|------------------------------------------|
| `GPR`   | GPR operand      | A General Purpose Register               |
| `SGPR`  | Shifted GPR operand | A General Purpose Register Left-shifted by a constant |
| `IOR`   | I/O Register operand | An I/O Transfer Register (Map 3)     |
| `ANYREG`| Any Register     | Any Register operand                     |
| `UIMM16`| Immediate (unsigned 16-bit) | 16-bit Immediate operand      |
| `SIMM16`| Immediate (signed 16-bit) | 16-bit Immediate operand        |
|`PCREL16`| PC Relative Address (16-bit) | 16-bit offset from IP in bytes|
| `OFF20` | Jump Offset (signed 20-bit) | 20-bit jump offset in words   |
| `UIMM32`| Immediate (unsigned 32-bit) | 32-bit immediate operand      |
| `SIMM32`| Immediate (signed 32-bit) | 32-bit immediate operand        |
| `BITW`  | Bit Width        | Width of a value in bits                 |
| `BYTESZ`| Byte Size        | Size of a value in bytes                 |
|`ABSIMM2`| Immediate (unsigned 2-bit) | 2-bit absolute (non-relocated) immediate |
|`ABSIMM5`| Immediate (unsigned 5-bit) | 5-bit absolute (non-relocated) immediate |
|`ABSIMM6`| Immediate (unsigned 6-bit) | 6-bit absolute (non-relocated) immediate |
|`ABSIMM8`| Immediate (unsigned 8-bit) | 8-bit absolute (non-relocated) immediate |
| `CC`    | Condition Code Name | A condition Code |


### GPR operands

A GPR operand is written as `r` followed by the index number of the register.

### Shifted GPR

A Shifted GPR operand is written as `GPR << ABSIMM5`

### I/O Register Operand

An I/O Register Operand is written as `io` followed by the index number of the register.

### Any Register

A Register in any map is either a GPR operand, and I/O Register operand, a system configuration register, a system information register, or a coprocessor register.

A system configuration register is written as either `sys` followed by the index number of the register or the alias name specified in the ISA (for example, the System Control Register may be written as `sys0` or `sysctl`).

A system information register is written as `info` followed by the index number of the register. 

A coprocessor register is written as `co` followed by the coprocessor number between 0 and 3, followed by `r`, followed by the register number.
If the Assembler is aware of the particular coprocessor expected in a given coprocessor slot, it may use the alias name provided by the Coprocessor's Assembly Supplement. 
For example, if the Assembler is aware of the presence of a FPU (according to <float-coproc.md>) in slot 0, it may alias `f0` as `co0r0`.

### Immediates

Immediate operands take an integer or a symbol. n-bit Unsigned Immediates require an unsigned quantity in `[0,2^n)`. Signed Immediates require a signed quantity in `[-2^(n-1), 2^(n-1))`. 

Other than Absolute Immediates (like `ABSIMM6`), immediate operands can have symbols. Unless specified with the `@pcrel` modifier, Immediate operands always treat symbols as absolute address in relocations. The relocation requires that the address fits in the specified size of immediate (a link error occurs if it does not). using the special modifiers `HI` and `LO` allows you to instead specify the lower or upper 16-bits of the symbol.

Unless modified, immediate relocations uses (given the appropriate n):
* `R_MICRON_<n>`
* `R_MICRON_LO16` (`LO <sym>`)
* `R_MICRON_HI16` (`HI <sym>`)

With the `@pcrel` modifier (or for `PCREL16`, see below), the relocation uses (given the appropriate n):
* `R_MICRON_PC16` 
* `R_MICRON_LOPC16` (`LO <sym>@pcrel`)
* `R_MICRON_HIPC16` (`HI <sym>@pcrel`)

### Offsets

The `PCREL16` and `OFF20` values are special cases of immediates. 
PCREL16 is identical to a `SIMM16` relocation, except that it defaults to resolving the specified symbol using a pc-relative relocation.

`OFF20` is a 20-bit immediate that resolves a 22-bit pc-relative relocation or a 22-bit signed integer offset, and discards the lower two bits to encode the instruction. It has two constraints, in addition to the constraints that would apply to a 22-bit signed immediate:
* Absolute Expressions must be divisible by 4, and
* Relocation Expressions must produce a 4-byte aligned quantity. The Assembler may error if a misalignment is reliably detected (for example, a 3-byte offset from a symbol known to be 4-byte aligned).

`OFF20` uses `R_MICRON_JMPOFF` relocation.

### Sizes/Widths

A size or width expression is an absolute immediate that encodes a size (in bytes) or a width (in bits). 

A `BITW` operand is a 5-bit unsigned absolute immediate that can take on any value between `1` and `32`.  `32` is encoded as `0`. As a special case, the following synthetic constants are defined for `BITW` operands with the following values:
* `byte`: 8
* `half`: 16
* `word`: 32.

A `BYTESZ` operand is a 2-bit immediate with a special encoding of `lg(sz)` where `sz` is the real size in bytes. The value must be a power of 2 less than 8. The same synthetic constants are defined above to be `1`, `2`, and `4` respectively.

### Condition Code

Condition Codes appear only in mnemonics, and use the following 1 or 2 character short forms:

| Short Form | Condition Name | Number | Canonical |
|------------|----------------|--------|-----------|
| `NV`       | Never          | 0      | Yes       |
| `C`        | Carry          | 1      | Yes       |
| `B`        | Below          | 1      | No        |
| `Z`        | Zero           | 2      | Yes       |
| `EQ`       | Equal          | 2      | No        |
| `V`        | Overflow       | 3      | Yes       |
| `O`        | Overflow       | 3      | No        |
| `CE`       | Carry/Equal    | 4      | Yes       |
| `BE`       | Below or Equal | 4      | No        |
| `LT`       | Less           | 5      | Yes       |
| `LE`       | Less or Equal  | 6      | Yes       |
| `N`        | Negative       | 7      | Yes       |
| `S`        | Signed         | 7      | No        |
| `P`        | Positive       | 8      | Yes       |
| `NS`       | Not Signed     | 8      | No        |
| `GT`       | Greater Than   | 9      | Yes       |
| `NLE`      | Not Less or Equal| 9    | No        |
| `GE`       | Greater or Equal | 10   | Yes       |
| `NLT`      | Not Less       | 10     | No        |
| `A`        | Above          | 11     | Yes       |
| `NBE`      | Not Below or Equal | 11 | No        |
| `NCE`      | Not Carry or Equal | 11 | No        |
| `NV`       | Not Overflow   | 12     | Yes       |
| `NO`       | Not Overflow   | 12     | No        |
| `NZ`       | Not Zero       | 13     | Yes       |
| `NE`       | Not Equal      | 13     | No        |
| `NC`       | Not Carry      | 14     | Yes       |
| `NB`       | Not Below      | 14     | No        |
| `AE`       | Above or Equal | 14     | No        |
| `AL`       | Always         | 15     | Yes       |

## Instruction Syntax

The following charts describes how assemblers should interpret and assemble given instruction syntax forms and mnemonics.

Each Chart has the following information:
* Mnemonic: The name of the instruction, which should be interpreted case-insentively. The special variables `<c>` and `<x>` may be written here. `c` is as if defined `CC <c>` and `x` is as if defined `ABSIMM2 <x>`.
* Operands: A list of operands written as the Short ID followed by a variable in `<>` (e.g. `GPR <d>`) 
* Opcode: The Opcode of the instruction. This may reference the special variable `x` if defined in the Mnemonic.
* Special Payload Encoding: A list of variable assignments of the form `<var>=<val>` where `var` is an encoding variable defined for the opcode in the ISA Spec, and `val` is either an integer expression or a variable. Implicitly the name of each encoding variable that is defined as a syntax variable in the Mnemonic or the Operands list is assigned to the value of that syntax variable. The special encoding variable `P` refers to the entire 24-bit payload (where its contents are undefined/ignored)
* Canonical: Describes whether or not the instruction specification is canonical  Canonical Specifications are the primary (or only) way to describe a particular encoding (without a `.instr` directive). Non-canonical encodings may describe a more efficient way to write or read a particular encoding, or may be useful in niche circumstances. Disassemblers and machine-code generators (such as assembly printing from a compiler) should prefer the canonical specification if it has no context otherwise.

### `UND`

| Mnemonic | Operands        | Opcode | Special Payload Encoding | Canonical |
|----------|-----------------|--------|--------------------------|-----------|
| `UND`    | -               | `0x00` | `P=0`                    | Yes       |

### `PAUSE`

| Mnemonic | Operands        | Opcode | Special Payload Encoding | Canonical |
|----------|-----------------|--------|--------------------------|-----------|
| `PAUSE`  | `ABSIMM6 <k>`   | `0x01` | N/A                      | Yes       |
| `NOP`    | -               | `0x01` | `k=0`                    | No        |

### `MOV`

| Mnemonic | Operands              | Opcode | Special Payload Encoding  | Canonical |
|----------|-----------------------|--------|---------------------------|-----------|
| `MOV<c>` | `GPR <d>, GPR <s>`    | `0x02` | `m=0, r=0, l=0`           | Yes       |
| `MOV<c>` | `GPR <d>, ANYREG <s>` | `0x02` | `m=MAP(s), r=0, l=0`      | Yes       |
| `MOV<c>` | `ANYREG <d>, GPR <s>` | `0x02` | `m=MAP(d), r=1, l=0`      | Yes       |
| `MOV`    | `GPR <d>, GPR <s>`    | `0x02` | `m=0, r=0, l=0, c=15`     | No        |
| `MOV`    | `GPR <d>, ANYREG <s>` | `0x02` | `m=MAP(s), r=0, l=0, c=15`| No        |
| `MOV`    | `ANYREG <d>, GPR <s>` | `0x02` | `m=MAP(d), r=1, l=0, c=15`| No        |
| `MOVL<c>`| `GPR <d>, GPR <s>`    | `0x02` | `m=0, r=0, l=1`           | Yes       |
| `MOVL<c>`| `GPR <d>, ANYREG <s>` | `0x02` | `m=MAP(s), r=0, l=1`      | Yes       |
| `MOVL<c>`| `ANYREG <d>, GPR <s>` | `0x02` | `m=MAP(d), r=1, l=1`      | Yes       |

### `LD`/`ST`

| Mnemonic | Operands                       | Opcode | Special Payload Encoding  | Canonical |
|----------|--------------------------------|--------|---------------------------|-----------|
| `LD`     | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x03` | `p=0`                     | Yes       |
| `ST`     | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x04` | `p=0`                     | Yes       |
| `LD`     | `GPR <d>, GPR <s>`             | `0x03` | `p=0,w=4`                 | No        |
| `ST`     | `GPR <d>, GPR <s>`             | `0x04` | `p=0,w=4`                 | No        |
| `PUSH`   | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x04` | `p=1`                     | Yes       |
| `POP`    | `GPR <d>, GPR <s>, BYTESZ <w>` | `0x03` | `p=1`                     | Yes       |
| `PUSH`   | `GPR <d>, GPR <s>`             | `0x04` | `p=1,w=4`                 | No        |
| `POP`    | `GPR <d>, GPR <s>`             | `0x03` | `p=1,w=4`                 | No        |
| `PUSH`   | `GPR <s>`                      | `0x04` | `p=1,w=4,d=30`            | No        |
| `POP`    | `GPR <d>`                      | `0x03` | `p=1,w=4,d=30`            | No        |

### `ADDI`

| Mnemonic | Operands                       | Opcode | Special Payload Encoding  | Canonical |
|----------|--------------------------------|--------|---------------------------|-----------|
| `ADDI`   | `GPR <d>, SIMM16 <s>`          | `0x08` |