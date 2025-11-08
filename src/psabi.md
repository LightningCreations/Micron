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

Map 1 Registers should not be modified by toolchains, except through explicit arrangement with the program. The precise values of Map 1 Registers should not be relied upon.

Registers in Map 2 and Maps 4-7 are callee saved and are not preserved accross procedure calls.

r1 and r2 are used to return values up to 8 bytes in size. Registers r1 through r10 are used to pass up to 10 parameters. 

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

!{#copyright}