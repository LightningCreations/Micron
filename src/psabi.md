# psABI

## Types

### C Primitive Sizes

`CHAR_BIT` is 8.

| Type         | Size    |
| ------------ | ------- |
| `bool`[^1]   | 1       |
| `short`      | 2       |
| `int`        | 4       |
| `long`       | 4       |
| `long long`  | 8       |
| `float`      | 4       |
| `double`     | 8       |
| `long double`| 8       |
| `void*`      | 4       |
| `intptr_t`   | 4       |
| `size_t`     | 4       |
| `intmax_t`   | 8       |
| `wchar_t`    | 2       |

[^1]: Referred to as `_Bool` in C since C99 until C23.

### Char Types

`char` is unsigned by default.

### Primitive Alignment

The Size and alignment of `align_max_t` are both 4. Each primitive less than or equal to 4 bytes in size is aligned to its size, rounded up to the next power of two bytes. 
Each primitive that is greater than 4 bytes in size are aligned to 4 bytes. This includes `_BitInt(N)` types.

### Floating Point Formats

`float` matches the IEEE754 binary32 format.

`double` and `long double` both match the IEEE754 binary64 format.

## Registers

In Map 0, Registers r1-r15 are callee saved and are not preserved accross prodecure calls. Registers r16-r31 are caller saved and must be restored to their values at entry by the function. r0 is a constant 0 register and cannot be modified.

r15 is recommended for use by code patterns that use a register to compute a value for immediate use. The Assembler may make use of this register implicitly to assemble certain psuedo-instructions. 

Map 1 Registers should not be modified by toolchains, except through explicit arrangement with the program. The precise values of Map 1 Registers should not be relied upon.

Registers in Map 2 and Maps 4-7 are callee saved and are not preserved accross procedure calls.

r1 and r2 are used to return values up to 8 bytes in size. Registers r1 through r10 are used to pass up to 10 parameters. 

### Register Overview

| Register(s) | Purpose     | Callee/Caller Saved |
|-------------|-------------|---------------------|
| `r0`        | Constant 0  | Constant Register   |
| `r1`        | Param/Return Register | Caller Saved |
| `r2`        | Param/Return Register | Caller Saved |
| `r3`-`r10`  | Param Register | Caller Saved |
| `r11`-`r14` | Scratch Register | Caller Saved |
| `r15`       | Special-Purpose Scratch Register | Caller Saved |
| `r16`-`r27` | Callee Saved Register | Callee Saved |
| `r28`-`r29` | Reserved Register | Callee Saved/Reserved |
| `r30` | Stack Pointer | Callee Saved |
| `r31` | Return Pointer | Callee Saved |

### Stack, Link Register, Reserved Registers

Map 0 Registers r28, r29, r30, and r31 are reserved for special use, within the caller saved regions.

r28 and r29 are not used by this ABI, but may be used by future versions or by individual machines/systems/programs as a special registers. If modified by software complying with this ABI, it must be restored before returning from the current procedure or entering another procedure, unless it is modified in cooperation with the definition of the register. 
> [!NOTE]
> It is recommended for r28 to be used as a Thread Pointer on a multicore system.

r30 is reserved to be the stack pointer. Before entering a procedure, it must refer to a memory address which points to the end of a memory region that is available for the procedure to use to store its own variables and parameters. 
The Address must be aligned to 4 bytes, and must be mutable. 
Additionally, the memory region immediately following the pointers may be required to hold parameters passed on the stack. The stack grows downwards, away from the end of the region allocated for the stack. Any region of memory between the address in r30 up to the end of the stack shall be preserved by compliant software, unless mutated via a pointer. All memory below the stack pointer in the allocated memory region may be freely clobbered at any point (including by an interrupt handler) and must not be relied upon in any particular value.

r31 is reserved to be the standard link register. Upon entry to any procedure, `r31` shall contain the address to return to upon exit. 

> [!NOTE]
> While r31 remains caller saved, every function call that isn't a tailcall will necessarily modify this register and require the register to be spilled to memory.
> The exception is if the function does not expect to return.

### Parameter Passing/Return Convention

When passing or returning values, each value is classified as follows:
* Primitive Values,
* Non Trivial Aggregates

Non-Trivial Aggregates are types with an alignment greater than 4, or a class type C++ with one of the following special member functions being non-trivial, and which is not *trivially relocatable*:
* Copy or Move (Since C++11) Constructor,
* Destructor.

All types that are not Non-Trivial Aggregates, and only have fields or elements of Primitive types (recursively) are Primitive Values.


Each parameter/return value in order is assigned a passing mode:
    * If the paramater/return value is larger than 8 bytes in size, it is passed/returned in memory,
    * If the parameter/return value is a Non-Trivial Aggregate, it is passed/returned in memory,
    * Otherwise, it is passed directly/returned.

If the return value is returned in memory, an implicit first parameter is inserted, which is a pointer to storage suitable for placing the entire return value. This pointer is then returned in `r1`.

A parameter passed in memory is replaced with a single 4 byte value passed directly, which points to the memory used to pass the value. 

After replacement, values passed/returned directly are divided into up to 2 4-byte chunks. Any chunk which is not present or consists entirely of padding bytes is discarded when passing directly. Then, each chunk in parameter order (with least significant byte first) is passed by allocating the next available register in `r1-r10`. If any chunk of a parameter cannot be allocated a register, the entire value is pushed to the stack in Right to Left Order (with the Leftmost parameter occupying the least significant address). The most significant address of the parameter area is 4 byte aligned, and up to 3 bytes are inserted after the leftmost parameter to align the stack to 4 bytes. Padding is inserted between parameters to align each parameter to the smaller of their size rounded up to the next power of 2, and 4 bytes. Once the first parameter is passed on the stack, no further parameters are passed in registers.

Each returned chunk of the return value is returned in the first available register of `r1` and `r2`. 

## Floating point co-processor

The Use of a hardware floating-point coprocessor to compile floating-point operations is permitted. Due to variability in machine allocations, the co-processor used for floating-point operations is not specified herein, and must be specified by the appropriate machine supplement or toolchain configuration options. Use of a particular co-processor number with hardware floating-point operations is not compatible with a system that does not have a floating-point co-processor in the appropriate slot.

Regardless of the use of a floating-point co-processor, floating-point values are not passed using floating-point registers, and are still passed using general purpose parameter registers when <= 8 bytes in size.

## ELF Files

### OSABI

The following OSABI values are defined

| OSABI | Constant          | Description         |
|-------|-------------------|---------------------|
| `0-63`| Multiple          | See gABI OSABI list |
| `255` | `OSABISTANDALONE` | Standalone/Freestanding target |

### Relocations

| Relocation Name | Relocation Number | Size    | Value | Description           |
|-----------------|-------------------|---------|-------|-----------------------|
|R_MICRON_NONE    | 0                 | 0 bits  | `0`   |Performs no Operation |
|R_MICRON_32      | 1                 | 32 bits | `S`   | Relocates against an absolute 32-bit address |
|R_MICRON_PC32    | 2                 | 32 bits | `S-IP`| Relocates against the 32-bit offset from the current address |
|R_MICRON_LO16    | 3                 | 16 bits | `TRUNC(S)` | Relocates against the lower 16 bits of an absolute 32-bit address |
|R_MICRON_PC16    | 4                 | 16 bits | `S-IP` |Relocates against the 16-bit offset from the current address |
|R_MICRON_LOPC16  | 5                 | 16 bits | `TRUNC(S-IP)` | Relocates against the lower 16 bits of a 32-bit offset from the current address |
|R_MICRON_HI16    | 6                 | 16 bits | `S>>16`  | Relocates against the upper 16-bits of an absolute 32-bit address |
|R_MICRON_HIPC16  | 7                 | 16 bits | `(S-IP)>>16` | Relocates against the upper 16-bits of a 32-bit offset |
|R_MICRON_JMPO    | 8                 | 16 bits | `(S-IP)>>2`  | Relocates against the aligned 17 bit offset suitable for a jump instruction, writing the top bits to the upper 15 bits of the word |
|R_MICRON_RELAX16_PC32 | 32           | 64 bits  | N/A | Hints that a link editor or other tool may convert a pointed to code sequence that loads a 32-bit pc relative offset into one that loads a 16-bit pcrelative offset |
|R_MICRON_RELAX16_32 | 33             | 64 bits | N/A | Hints that a link editor or other tool may convert a pointed-to code sequence that loads a 32-bit absolute address into one that loads a 16-bit absolute address, or a 16-bit offset|
|R_MICRON_RELAXJMPOFF_PC32 | 34       | 96 bits | N/A | Hints that a link editor or other tool may convert a pointed-to code sequence that loads and jumps to a 32-bit pc relative address into one that performs a direct jump to a 17-bit aligned offset from the resulting instruction |
| N/A           | 35-63               | 0 bits | N/A  | Reserved for future relaxation hints and must be ignored by link editors. Must not be generated by toolchains |

S: The Address of the symbol being relocated

IP: The instruction pointer at the end of the relocation.

#### Link Relaxations

Toolchains (including assemblers and compilers) may emit certain code sequences to generate a load of a symbol address that may not be representable as a 16-bit address or offset, or a jump to an address that may not be representable as a 20-bit instruction offset. When doing so, the toolchain may emit relaxations into the object file's relocation table, pointing to the beginning of the relaxable code sequence, which are operative hints to link editors that the code sequences can be contracted to a smaller (usually single instruction) code sequence. 

Toolchains are not required to emit relaxations, and link editors are not required to make use of them. Any link relaxation is a hint and may be ignored and toolchains MUST NOT rely on them being processed for emitting correctly relocated code.

Certain specific code sequences are supported, and it is undefined behaviour to apply a relaxation to an ill-formed code sequence. Link Editors are not required to check for invalid code sequences, and are not required to preserve the behaviour of an invalid code sequence where a link relaxation is applied. It is further not required that the link editor check that relocations pointing into the relaxed code sequence refers to the same symbol as the

##### `R_MICRON_RELAX16_PC32` and `R_MICRON_RELAX16_32`

`R_MICRON_RELAX16_PC32` and `R_MICRON_RELAX16_32` hint that a 2 instruction long code sequence loads an address, either by loading an absolute relocated address, or by loading a relocated offset and adding it to the instruction pointer at the end of the code sequence, and may be relaxed to a single instruction that loads the same address. Note that the link editor is not required to preserve the kind of relocation indicated in the relaxation - the `PC32` vs. `32` refers to the manner of address loading (PC Relative vs. Absolute). The Link editor may emit either an absolute 16-bit address, or a 16-bit offset, where the resulting value loaded is the address of the appropriate symbol.

The below code sequences are written in assembly, with unrelocated machine code adjacent. `<val>+REG` refers to the value of `<val>` with the register number `REG` added. IE. if `REG` is `r11`, then `0x00+REG` is `0x0B`, and if `REG` is `r31`, then `0x40+REG` is `0x5F`.

REG is any GPR that is the same in both instructions

The following code sequence is supported for `R_MICRON_RELAX16_PC32(sym)`,
```
LRAU REG, R_MICRON_LOPC16(sym)-4  # 0x06 0x00+REG 0x00 0x00
ADDIH REG, R_MICRON_HIPC16(sym) # 0x08 0x40+REG 0x00 0x00
```

The following code sequence is supported for `R_MICRON_RELAX16_32(sym)`
```
LDI REG, R_MICRON_LO16(sym) # 0x05 0x00+REG 0x00 0x00
ADDIH REG, R_MICRON_HI16(sym) # 0x08 0x40+REG 0x00 0x00
```

The two resulting instructions that can be emitted by the link editor, if eligible, are below. If both are eligible, the link editor may choose which code sequence to emit.

```
# Code Sequence 1
LRA REG, R_MICRON_PC16(sym) # 0x06 0x00+REG 0x00 0x00
# Code Sequence 2
LDI REG, R_MICRON_LO16(sym) # 0x05 0x00+REG 0x00 0x00
```

##### `R_MICRON_RELAXJMPOFF_PC32`

`R_MICRON_RELAXJMPOFF_PC32` hints that the 3 instruction sequence it is applied to has the effective behaviour of loading the address of the specified symbol into an otherwise unused scratch register and jumping to that address.

The following code sequence is supported, where REG is any regster and CC is any Condition COde

```
LDI REG, R_MICRON_LO16(sym) # 0x05 0x00+REG 0x00 0x00
ADDIH REG, R_MICRON_HI16(sym) # 0x08 0x40+REG 0x00 0x00
JMPR{CC} REG
```

The following resulting code sequence may be emitted by the link editor, if eligible.

```
JMP{CC} R_MICRON_JUMPO(sym)
```

Note that is it is not guaranteed that the value of `REG` is preserved by the relaxation. Thus code following the relaxed code sequence must treat `REG` as having an undefined value.

!{#copyright}