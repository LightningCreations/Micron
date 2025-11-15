# Micron Instruction Set

## Gen Design

Word Size: 32 bit

Instruction Size/Alignment: 4 bytes

Variable Sized Instructions: No

Common Flags/Condition Code Driven

## Instruction format

8-bit (LSB) opcode followed 24-bit payload.

LSB First Order

## Registers

### Register Maps

There are 8 maps of registers:
* Map 0: General Purpose
* Map 1: System Configuration
* Map 2: I/O Transfer Registers
* Map 3: Information
* Map 4: Coprocessor Control
* Maps 8-15: Co-processor Registers.

There are 32 registers of each type. Except for Map 0, not all registers may be defined.

Co-processor Registers are available only if the applicable co-processors are connected.

### Map 0: General Purpose

Assembly syntax: `r`*`n`*.

All Registers of the Map are defined. Certain Registers have special meaning:
* `r0` is the zero-register. It reads as zero, and writes are ignored.

### Map 1: System Configuration

Assembly syntax: `sys`*`n`* or alias.

Refer to the following table of defined registers. System Configuration Registers may be written only with values that meet register-specific constraints.



| Regno. | Aliases  | Description    |
| ------ | -------- | -------------- |
| 0      | `sysctl` | System Control |
| 2      | `inttab` | Interrupt Table Pointer |
| 31     | `intret` | Interrupt Procedure return register |

Reading or Writing an undefined register causes `EX[2]`. Writing an invalid value to a defined register causes `EX[5]`

#### System Control (Map 1, Register 0)

Format:
```
+0-----------------------------31+
|i000000000000000000000000000000t|
+--------------------------------+
```

(All bits indicated as 0 must be written with 0)


| Bits | Name     | Description |
| ---- | -------- | --- |
| `i`  | Interrupt Enable (IE)     | Only process IRQs when set to 1. |
| `t`  | Trap     | Set to 1 by the processor when an Exception occurs. IRQs and Unit Error interrupts are not processed while this bit is set. Only cleared manually. |


#### Interrupt/Exception Table (Map 1, Register 2)

Format:
```
+0-----------------------------32+
|000aaaaaaaaaaaaaaaaaaaaaaaaaaaaa|
+--------------------------------+
```

(All bits indicated as 0 must be written with 0)

Bits `a` contain the 29 most significant bits of an 8-byte aligned address which points to the interrupt table. 512 bytes starting from this address refer to 64 8-byte entries of the interrupt table, which use the following format, in LSB-first order using little-endian byte encoding:
```
+0-----------------------------31+
|p0tttttttttttttttttttttttttttttt|
+32----------------------------63+
|00000000000000000000000000000000|
+--------------------------------+
```

The `t` bits are the 30 most significant bits of the address to transfer control to when the specified interrupt occurs.

The `p` bit must be set for all interrupt vectors that are present and valid to execute.

##### Interrupts

The first 16 interrupt entries are reserved for hardware exceptions, these interrupts are allocated as follows (and the `n`th entry in this list is designated elsewise as `EX[n]`):
* Entry `0`: Exception Handling Fault - an exception is raised when the `t` flag is set. 
* Entry `1`: Bus Fault - accessing memory in a particular manner causes an error, or attempts to access memory that doesn't exist.
* Entry `2`: Invalid Instruction - An instruction that is executed is an unknown opcode, reserved, malformed, or invalid
* Entry `3`: Unaligned Branch Target - an indirect branch is unaligned.
* Entry `4`: Consistency - An invalid system control register was loaded from memory, or an invalid value was written to a system register.
* Entry `7`: Non-maskable Interrupt - May be raised in response to a priority signal external to the processor that requires immediate resolution. This is handled like an IRQ, but does not obey the `i` flag. 
* Entries `8`-`15`: Co-processor Unit `n` Error - The corresponding Coprocessor unit `n` signals an error after a `CPIn` instruction (`n` is Exception number - 4).
* Entries `5`, `6`, and `16`-`31` are reserved.

The remaining entries (32-63), may be allocated as IRQ vectors.

Exceptions are raised regardless of the `i` bit. The `t` bit is set to `1` when an exception is raised. It is not modified by any other interrupt (including an NMI) being raised.

If an exception occurs raising `EX[0]`, the processor RESETs. 

#### Interrupt Return Pointer (Map 1, Register 31)

When an interrupt occurs, this register is set to the value of the instruction pointer of the next instruction to execute - in the case of an IRQ or NMI, this is immediately after the last instruction to execute (or the jump target, if that instruction was a taken branch). In the case of an exception, this is the instruction that raised the exception.

In the case of `EX[0]`, the value of this register is undefined. 

### Map 2: I/O Transfer Registers

Map 2 defines a sequence of input and output shift registers for transfering data to external peripherals. 

### Map 3: Information Registers

The Information Registers Map is a Read Only Map that contains information about the CPU. All Registers Presently Read 0. Writes are illegal and raise `EX[2]`

### Map 4: Coprocessor Control

Each Co-processor has a 32-bit control word, which is defined by the Coprocessor.

Register N in Map 4 is defined if Co-processor N is present and enabled. 

Reads and writes to an undefined register or a register corresponding to a not-present or disabled coprocessor results in `EX[2]`.

#### Map 4, Register 30: Coprocessor Enable

The Coprocessor Enable register allows the system software to control what coprocessors are operating and usable from the CPU.

Format:
```
+0-----------------------------32+
|EEEEEEEE000000000000000000000000|
+--------------------------------+
```

The bits marked `E` may be set by the program when the corresponding bit of Register 31 is set. Setting the nth bit to 1 enables the coprocessor and setting it to 0 disables it.

Bits marked as 0 must not be written with 1.

#### Map 4, Register 31: Coprocessor Present

The Coprocessor Enable register allows the system software to determine what coprocessors are connected to the CPU. This register is read-only and cannot be written from the CPU. Attempting such a write with a MOV instruction raises `EX[2]`.

Format:
```
+0-----------------------------32+
|PPPPPPPP000000000000000000000000|
+--------------------------------+
```

The nth bit is set to 1 if the nth coprocessor is present. Note that it is not guaranteed that the set of enabled coprocessors is contiguous or that the set of enabled coprocessors begins at 0.


### Map 8-15: Co-processor Maps

Co-processors connected to the system may expose up to 32 registers each. Registers in map `N` are only defined if the coprocessor co-processor (Co-processor N-8) is enabled.

## Instructions

### Undefined Instructions

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| UND      | `0x00` | -                          |
| UND      | `0xFF` | -                          |

(The Payload bits are ignored by both instructions)

Timing (Execute Latency): 0 cycles

Exception Order:
* `EX[2]` (decode): Unconditionally

Behaviour: Unconditionally raises Invalid Instruction errors

```
instruction UND():
    Raise(EX[2])
```

### Pause

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| PAUSE    | `0x01` | `kkkkkk000000000000000000` |


Timing (Execute Latency): 0 cycles + k

Behaviour: Delays execution for `k` cycles, 0-63


```
instruction PAUSE(k: u6):
    SuspendForClockTicks(k)
```

### Move 

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------32` |
| MOV      | `0x02` | `dddddsssss00lccccrmmmm00` |

Payload Bits Legend:
* `d`: Destination Register
* `s`: Source Register
* `m`: Map
* `r`: Direction
* `c`: Condition Code (See Jump)
* `l`: Latency Control

Timing (Execute Latency): `1+c+t`, where:
* `c` is 0 if Condition Code is 0, 1 if Condition Code is 15 or Latency Control is 0 and the Condition Check Fails, 2 if Latency Control is 1 or the Condition Code is not 15 and the Condition Check Succeeds
* `t` is 0 if Map is 0 or Latency Control is 0 and the Condition Check Fails, 2 if Map is not 0 when Latency Control is 1 or the Condition Check Succeeds.

Behaviour: Copies data between general purpose registers and to/from general purpose registers into other registers.

```
instruction MOV(d: u5, s: u5, m: u2, dir: u1, c: ConditionCode, l: bool):
    if m!=0:
        if dir==0:
            ValidateRegisterReadable(m,d);
        else:
            ValidateRegisterWritable(m,d);
    if CheckCondition(flags, c):
        let ms, md: u2;
        if dir==1:
            md = m;
            ms = 0;
        else:
            ms = m;
            md = 0;
        if md==3:
            Raise(EX[2]);
        let val: u32;
        val = ReadRegister(ms, s);
        if md == 2 or m > 3:
            ValidateConfigurationRegisterValue(d, val);
        WriteRegister(md, d, val);
    else:
        if l:
            if m!=0:
                SuspendForClockTicks(4);
            else:
                SuspendForClockTicks(2);
```

### LD/ST

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------32` |
| `ST`     | `0x03` | `dddddsssssww00000000000p` |
| `LD`     | `0x04` | `dddddsssssww00000000000p` |
| `LRA`    | `0x06` | `dddddx00oooooooooooooooo` |


Payload Bits Legend:
* `s`: Source Register
* `d`: Destination Register
* `w`: Width
* `i`: Immediate Value
* `o`: Offset
* `h`: Hi/lo bits
* `q`: Scale Quantity
* `p`: Push/Pop
* `x`: Sign/Zero Extend

Timing: 
* `ST`, `LD`: 4 Cycles, plus Memory Delay
* `LDI`, `LRA`: 1 Cycle
* `SLDI`: 2 cycles

Behaviour:
* `ST`: Stores `1 << w` bytes from `d` to `[s]`
* `LD`: Loads `1 << w` bytes from `[s]` into `d`
* `LDI`: Loads an immediate `i` into the first (h=0) or upper (h=1) 16 bits of `d`
* `LRA`: Loads the address `IP + o` (`o` is a signed immediate if `x` is true, and an unsigned immediate otherwise) into `d`. `IP` is taken from the beginning of the next instruction

```
instruction ST(s: u5, d: u5, w: u2, p: bool):
    let val = ReadRegister(0,s);
    let addr: u32;
    let width = 2 << w;
    if width == 8:
        Raise(EX[2]);
    if p:
        addr = ReadRegister(0,d) - width;
        WriteRegister(0,d, addr);

    WriteAlignedMemoryTruncate(addr, val, width);
    
instruction LD(s: u5, d: u5,w: u2, p: u2):
    let width = 2 << w;
    if width == 8:
            Raise(EX[2]);
    let addr: u32;
    addr = ReadRegister(0,s);
    if p:
        WriteRegister(0,s,addr + width);
    let val = ReadAlignedMemoryZeroExtend(addr, w+1);
    WriteRegister(0,s,val);
    
instruction LRA(d: u5, x: bool, i: u15):
    let val = SignExtendOrZeroExtend(i, x) + IP;
    WriteRegister(0,d,val);
```

### Immediate Arithmetic

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `ADDI`   | `0x08` | `dddddhcsiiiiiiiiiiiiiiii` |

Timing: 2
Payload Bits Legend:
* `d`: Destination Register
* `h`: High half
* `s`: Extend Sign
* `c`: Supress Flags Modification
* `i`: Immediate

Behaviour: Adds a 12-bit zero or sign-extended immediate to `d`.

### ALU Instructions

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `ADD`    | `0x09` | `dddddaaaaabbbbbcsssssp00` |
| `SUB`    | `0x0A` | `dddddaaaaabbbbbcsssssp00` |
| `AND`    | `0x0B` | `dddddaaaaabbbbbcsssssp00` |
| `OR`     | `0x0C` | `dddddaaaaabbbbbcsssssp00` |
| `XOR`    | `0x0D` | `dddddaaaaabbbbbcsssssp00` |

Timing: 2

Payload Bits Legend:
* `a`: Source Register 1
* `b`: Source Register 2
* `d`: Destination Register
* `c`: Suppress Condition
* `p`: Shift Polarity
* `s`: Shift Quantity

Behaviour:
```
instruction {ADD, SUB, AND, OR, XOR}(a: u5, b: u5, d: u5, c: bool, s: u5, p: bool):
    let src1, src2: u32;
    if p:
        src1 = ReadRegister(0, a) << s;
        src2 = ReadRegister(0,b);
    else:
        src1 = ReadRegister(0, a);
        src2 = ReadRegister(0,b) << s;
    let dest: u32;
    let flags_val, flags_mask: u4;
    switch (instruction):
        case ADD:
            dest, flags_val = src1 + src2;
            flags_mask = 0xF;
        case SUB:
            dest, flags_val = src1 - src2;
            flags_mask = 0xF;
        case AND:
            dest = src1 & src2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
        case OR:
            dest = src1 | src2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
        case XOR:
            dest = src1 ^ src2;
            flags_val = LogicCondition(dest);
            flags_mask = 0x3;
    if not c:
        SetFlagsRegisterByMask(flags_mask, flags_val);

```

### Butterfly Shifts

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `BSL`    | `0x0E` | `dddddvvvvvqqqqqcx0wrrrrr` |
| `BSR`    | `0x0F` | `dddddvvvvvqqqqqcx0wrrrrr` |

Timing: 3

Payload Bits Legend:
* `d`: Destination Register
* `v`: Input Value
* `q`: Shift Quantity
* `c`: Suppress Condition
* `w`: Wrap Quantity
* `r`: Shift Remainder (Input value)
* `x`: Invert by Sign


Behaviour: Shifts `v` by `q` and places the value in 


### Branches

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `JMP`    | `0x10` | `cccclllllooooooooooooooo` |
| `JMPR`   | `0x11` | `cccclllllrrrrr0000000000` |

Payload Bits Legend:
* `c`: Condition Code
* `l`: Link Register
* `o`: Destination Offset (Bits 2..17)
* `r`: Destination Register

Timing: `2+t+l+r` where:
* `t` is `1` if the branch is taken and `0` if it is not taken
* `l` is `1` if Link Register is non-zero and the branch is taken, and `0` otherwise
* `r` is `1` for `JMPR` and `0` for `JMP`

Behaviour: Jumps to the destination, if the condition is satisfied, saving the return address in `l` if taken:

* `JMP`: The offset is `IP + o * 4` where `o` is a signed offset. `IP` is the same as the return address and points to the beginning of the next instruction
* `JMPR`: The offset is read from `r`


```
instruction JMP(c: ConditionCode, l: u5, o: u15):
    let disp = SignExtend(o) << 2;
    let curr_ip = IP;
    if CheckCondition(flags, c):
        if l != 0:
            WriteRegister(0,l, curr_ip);
        IP = curr_ip + disp;

instruction JMPR(c: ConditionCode, l: u5, r: u5):
    let addr = ReadRegister(0,r);
    if addr & 3 != 0:
        Raise(EX[3]);
    let curr_ip = IP;
    if CheckCondition(flags, c):
        if l != 0:
            WriteRegister(0,l, curr_ip);
        IP = addr;
```

#### Condition Code

`JMP`, `JMPR`, and `MOV` all use a 4-bit condition code to encode the branch condition. This includes conditions for "Always" and "Never". 

```
enum ConditionCode is u4:
    Never = 0,
    Carry = 1,
    Zero = 2,
    Overflow = 3,
    CarryOrEqual = 4,
    SignedLess = 5,
    SignedLessOrEq = 6,
    Negative = 7,
    Positive = 8,
    SignedGreater = 9,
    SignedGreaterOrEq = 10,
    Above = 11,
    NotOverflow = 12,
    NotZero = 13,
    NotCarry = 14,
    Always = 15

function CheckCondition(flags: u32, c: ConditionCode) is bool:
    switch (c):
        case Never:
            return false;
        case Carry:
            return (flags & 0x80) != 0;
        case Zero:
            return (flags & 0x01) != 0;
        case Overflow:
            return (flags & 0x40) != 0;
        case CarryOrEqual:
            return (flags & 0x81) != 0;
        case SignedLess:
            return (((flags & 0x40) != 0) == ((flags & 0x02) != 0)) and (flags & 0x01) == 0;
        case SignedLessOrEq:
            return (((flags & 0x40) != 0) == ((flags & 0x02) != 0)) or (flags & 0x01) != 0;
        case Negative:
            return (flags & 0x02) != 0;
        case Positive:
            return (flags & 0x02) == 0;
        case SignedGreater:
            return not ((((flags & 0x40) != 0) == ((flags & 0x02) != 0)) or (flags & 0x01) != 0);
        case SignedGreaterOrEq:
            return not ((((flags & 0x40) != 0) == ((flags & 0x02) != 0)) and (flags & 0x01) == 0);
        case Above:
            return (flags & 0x81) == 0;
        case NotOverflow:
            return (flags & 0x40) == 0;
        case NotZero:
            return (flags & 0x01) == 0;
        case NotCarry:
            return (flags & 0x80) == 0;
        case Always:
            return true;
```

Behaviour:
* Multiplies 

### I/O Transfers

| Mnemonic | Opcode | Payload                    |
| -------- | ------ | -------------------------- |
|          | `0--7` | `8---------------------31` |
| `IN`     | `0x14` | `dddddpppppppp000000wwwww` |
| `OUT`    | `0x15` | `ssssspppppppp000000wwwww` |

Payload Bits Legend:
* s: Source Transfer Register
* d: Destination Transfer Register
* w: Transfer Bit Width
* p: Port Number

Timing: 7 + Port Delay

Behaviour: Shift `w` bits in an io transfer register in or out to an I/O Port
* `IN` : Shifts bits into the high bits of the transfer register
* `OUT`: Shifts bits out of the low bits of the transfer register

```
instruction IN(s: u5, p: u8, w: u5):
    let val = RotateRight(ReadBitsFromPort(p,w),w);
    let regval = ReadRegister(2,s) >> w;
    WriteRegister(2,s, val | regval);

instruction OUT(s: u5, p: u8, w: u5):
    let regval = ReadRegister(2,s);
    let val = regval & ((1 <<w)-1);
    WriteRegister(2, s, regval >> w);
    WriteBitsToPort(val, p, w);
```

### Flags Manipulation

| Mnemonic | Opcode   | Payload                    |
| -------- | -------- | -------------------------- |
|          | `0--7`   | `8---------------------31` |
| `LDFLAGS`| `0x18`   | `ddddd0000000000000000000` |
| `STFLAGS`| `0x19`   | `sssss0000000000000000000` |


Payload Bits Legend:
* s: Source Register
* d: Destination Register

Timing: 1

Behaviour:
* `LDFLAGS` loads the flags bits into the lower 4 bits of `d` (zero extended)
* `STFLAGS` stores the lower 4 bits of `s` into the flags bits

The Flags Bits are:

| `0--3` |
|--------|
| `cvnz` |

* `c`: Carry
* `v`: Signed Overflow
* `n`: Negative
* `z`: Zero

```
instruction LDFL(d: u5):
    let val = flags;
    WriteRegister(0,d, val);

instruction STFL(s: u5)
    let val = ReadRegister(0, s);
    flags = val & 0xF;
```

### Invoke Coprocessor Unit

| Mnemonic | Opcode   | Payload                    |
| -------- | -------- | -------------------------- |
|          | `0--7`   | `8---------------------31` |
| `CPIx`   | `0x20`+x | `ffffpppppppppppppppppppp` |
| `NCPIx`  | `0x28`+x | `ffffpppppppppppppppppppp` |
| `CPIxEF` | `0x30`+x | `ffffffpppppppppppppppppp` |
| `NCPIxEF`| `0x38`+x | `ffffffpppppppppppppppppp` |

(`x` is a value from `0` to `7`, representing the co-processor number to invoke, for example, `CPI0` has opcode 0x20 and `NCPI7` has opcode 0x2F)

Timing: 2 + N where:
* For `CPIx` and `CPIxEF`, `N` is the delay in cycles before the co-processor becomes ready to execute again
* For `NCPIx` and `NCPIxEF`, `N` is 0. 

Payload Bits Legend:
* `f`: Co-processor function
* `p`: Co-processor instruction payload

Behaviour: Executes the specified Coprocessor function with the specified payload
* `CPIx`/`CPIxEF`: Waits for the Co-processor to finish all operations, and raises the appropriate unit error if the Coprocessor reports it,
* `NCPIx`/`NCPIxEF`: Finishes immediately.
* `CPIx`/`NCPIx`: Allows specifying up to 16 functions with a 20-bit payload
* `CPIxEF`/`NCPIxEF`: Allows specifying up to 64 functions with a 18-bit payload (bottom 18-bits of the 20-bit payload)

```
instruction {CPI0, CPI1, CPI2, CPI3}(f: u4, p: u20):
    let coproc: u4;
    switch (instruction):
        case CPI0:
            coproc = 0;
        case CPI1:
            coproc = 1;
        case CPI2:
            coproc = 2;
        case CPI3:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);
    WaitOnCoprocessor(coproc);
    if PullCoprocessorException(coproc):
        Raise(Ex[4+coproc]);

instruction {CPI0EF, CPI1EF, CPI2EF, CPI3EF}(f: u6, p: u18):
    let coproc: u4;
    switch (instruction):
        case CPI0EF:
            coproc = 0;
        case CPI1EF:
            coproc = 1;
        case CPI2EF:
            coproc = 2;
        case CPI3EF:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);
    WaitOnCoprocessor(coproc);
    if PullCoprocessorException(coproc):
        Raise(Ex[4+coproc]);

instruction {NCPI0, NCPI1, NCPI2, NCPI3}(f: u4, p: u20):
    let coproc: u4;
    switch (instruction):
        case NCPI0:
            coproc = 0;
        case NCPI1:
            coproc = 1;
        case NCPI2:
            coproc = 2;
        case NCPI3:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);

instruction {NCPI0EF, NCPI1EF, NCPI2EF, NCPI3EF}(f: u6, p: u18):
    let coproc: u4;
    switch (instruction):
        case NCPI0EF:
            coproc = 0;
        case NCPI1EF:
            coproc = 1;
        case NCPI2EF:
            coproc = 2;
        case NCPI3EF:
            coproc = 3;
    if not IsCoprocessorEnabled(coproc):
        Raise(EX[3]);
    
    ExecuteCoprocessorInstruction(coproc, f, p);
```

### Halt/Stop CPU

| Mnemonic | Opcode   | Payload                    |
| -------- | -------- | -------------------------- |
|          | `0--7`   | `8---------------------31` |
| `HALT`   | `0x30`   | `000000000000000000000000` |
| `STOP`   | `0x31`   | `000000000000000000000000` |

Timing: 
* `HALT`: 1
* `STOP`: N/A

Behaviour: Places the CPU in a low-power state and stops executing
* `HALT`: Execution resumes after an NMI, IRQ (if enabled), Unit Error (if `sysctl.t=0`), or RESET.
* `STOP`: Execution resumes after a RESET only. No other interrupts are serviced.

```
instruction HALT() {
    SetStatus(2);
    WaitForInterrupt();
}
instruction STOP() {
    SetStatus(3);
    ShutdownCpu();
}
```

## Initial State

### Execution Address

The CPU begins executing from address 0xFF00.

### Register Contents

* GPR, I/O Transfer, and Coprocessor Registers are undefined
* System Registers are set according to the following:
   * `sysctl`: All bits 0
   * `copctl`: P bits set according to which Coprocessors are connected. Other bits are 0
   * `inttab`: 0
   * `intret`: Undefined

!{#copyright}