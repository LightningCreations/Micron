# Floating Point Co-processor Unit

## Unit Configuration

The 6-bit field in the co-processor control register is set as follows. After writing to it on the CPU side, there is a 6 cycle execution delay before the unit becomes ready. If the unit is busy when the CPU writes to these bits, the Unit signals a Unit Error (Configuration Error) immediately. Pending Instructions are disposed of. 

Format:
```
+0----5+
|i00000|
+------+
```

When `i` is set, The Integer Op Extended Functions are enabled to operate on fpi registers. 

## Registers

Within the Co-processor Register Map, the following registers are defined:
* Registers 0-15 are floating-point registers, designated by name `fp`*`n`*
* Registers 16-23 are co-processor local integer registers, designated by `fpi`*`n`* where `n` is the register number - 16.
* Registers 24-31 are Unit Status Registers specified below, designated by `fpc`*`n`* where `n` is register number - 24, or an alias. 
    * Writes to these registers from the CPU incur a 2 cycle execution delay before the unit becomes ready. This delay is ordered properly with pending instructions, and a Unit Error is not signaled if the unit is busy. 
    * A subsequent write to any of these registers without an intervening instruction on the unit disposes of the previous write if it is still pending. 

### Unit Control/Status Registers

#### FP Unit Error/Operation Register (Register 24)

`fpc0` or `fpuop` (Register 24) is the FPU Unit Error/Operation Register.

This register can be overwrriten with 0. Overwriting with any other value raises a Unit Error (STATUS ERROR). 

The Register is `0` when the unit is first enabled, and becomes `0` after the configuration is changed or the unit is disabled and reenabled.

Format:
```
+0-----------------------------31+
|xxxzzzzzzzzzzzzzruiiiiiiiiiiiiii|
+--------------------------------+
```

`x` is set to the unit error number on an unit error, or `0` if there hasn't been a unit error since the last time the register was reset to `0`.

`i` contains the low 16-bits (with the 2 least significant bits discarded) of the most recently begun Unit Instruction.

`u` is set to 1 if any Unit Instruction has been issued since the register was last reset to `0`. This includes if the instruction is still pending or in-progress.

`r` is set to 1 if the unit is currently busy, and 0 otherwise. Note that an instruction like `FPSTREG` will always observe this bit being `0`, but a CPU issued `MOV` to map 0 can read the bit as 1.

Bits indicated by `z` are unused and must be ignored by programs.

##### Error Numbers

* Error 1 (FPE): A previous floating-point computation or status check signaled an unmasked floating-point exception
* Error 2 (INVALID INSTRUCTION): An undefined or invalid unit instruction was issued 
* Error 3 (STATUS ERROR): An invalid value was written to a status register

#### FP Control/Status Register (Register 25)

`fpc1` or `fpcsr` is the Floating-Point Control and Status Register. 

Format:
```
+0-----------------------------31+
|cccczouidvzzzzzz00000O̅U̅I̅D̅V̅0000rr|
+--------------------------------+
```

Bits indicated by `0` must be written with `0` or STATUS ERROR is thrown. Bits indicated by `z` must not be overwritten (may cause STATUS ERROR), and must be ignored by programs.

Bits `c` contain the status flags from the most recent comparison op. Integer comparisons and Bitwise TEST use the same format as the flags register for the primary ISA. 
Floating point partial-order comparison use the format `eltk`, which is set as follows:
* `e` is set to the partial-order equality
* `l` is set to the partial-order less-than
* `t` is set to the total-order less-than
* `k` is set to unordered.

Floating-point total-order comparisons use the format `EL0K`, which is set as follows:
* 

Bits `o`, `u`, `i`, `d`, `v` are set by floating-point operations that signal an FP Exception. 
Bits `O̅`, `U̅`, `I̅`, `D̅`, and `V̅` are control bits that determine exception masking. When set to `0`, the corresponding floating-point exception is masked and does not trigger Error 1. When set to `1`, the corresponding exception is raised. The exceptions are:
* Numeric Overflow (bits `o` (indication) and `O̅` (masking))
* Numeric Underflow (bits `u` (indication) and `U̅` (masking))
* Inexact Result (bits `i` (indication) and `I̅` (masking))
* Divide by Zero (bits `d` (indication) and `D̅` (masking))
* Invalid Operation (bits `v` (indication) and `V̅` (masking))

`r` is the dynamic rounding mode, set as follows:
* `0` is Round to Nearest, Ties Even
* `1` is Round to Zero
* `2` is Round To +∞
* `3` is Round to -∞

## Unit Functions

### Notes about `CPIxEF`

`CPIxEF` allows up to 64 functions (instead of 16-bits) to be uniquely invoked at the cost of the top 2 bits of the payload being forced to `0`.

All unit functions that encode a rounding mode store the static rounding control in the top 2 bits. Thus invoking these functions with `CPIxEF` cannot select a static rounding mode other than "Round to Nearest, Ties Even" (rounding mode 0). These instructions can still select a dynamic rounding control, and a rounding control other than mode `0` can be set in fpcsr. 
Other unit functions do not use the top two bits of the payload.

Functions that rely on the rounding mode and that are only accessible to `CPIxEF` (function numbers > `0x0F`) do not support static rounding control.

### Pause FPU 

| Mnemonic | Function | Payload                |
| -------- | -------- | ---------------------- |
|          | `0--6`   | `0-----------------19` |
| `FPAUSE` | `0x00`   | `kkkk0000000000000000` |

Timing (Execute Latency): 0 cycles + k

Behaviour: Delays execution for `k` cycles, 0-15

```
instruction PAUSE(k: u4):
    SuspendForClockTicks(k)
```

### Move FP Register

| Mnemonic | Function | Payload                |
| -------- | -------- | ---------------------- |
|          | `0--6`   | `0-----------------19` |
| `FPMOV`  | `0x01`   | `sssssdddddcccc000000` |

Moves between floating-point registers. The MOV is bitwise if it succeeds.

Payload bit legend:
* `s`: Source Register Number
* `d`: Destination Register Number
* `c`: Condition Code

Condition Code is interpreted the same way as for the `MOV` CPU Instruction, but it reads from `fpcsr` instead of `flags`

Timing: `1+c`, where:
* `c` is 0 if Condition Code is 0, 1 if Condition Code is 15 or the Condition Check Fails, 2 the Condition Code is not 15 and the Condition Check Succeeds

```
instruction FPMOV(s: u5, d: u5, c: ConditionCode):
    if CheckCondition(fpcsr & 0xF, c):
        let val: u32;
        val = ReadRegister(UNIT, s);
        WriteRegister(UNIT, d, val);
        CheckAndThrowPendingUnitErrors();
```

### Floating-point Conversions

| Mnemonic | Function | Payload                |
| -------- | -------- | ---------------------- |
|          | `0--6`   | `0-----------------19` |
| `CVTF2F` | `0x04`   | `ssssddddffkk00000yrr` |
| `CVTI2F` | `0x05`   | `sss0ddddi0kk00000yrr` |
| `CVTF2I` | `0x06`   | `ssssddd0ffi000000yrr` |

Payload bit legend:
* `s`: Source Register Number (FP Register)
* `d`: Destination Register Number (FP Register)
* `f`: Source Floating-point Format
* `k`: Destination Floating-point Format
* `y`: dynamic rounding control.
* `r`: Static rounding mode.
* `i`: Signed Integer

Timing: 
* For `CVTF2F`, If `f` and `k` are the same format: 1 Cycle,
* For `CVTF2F`, If `f>k`: 3 Cycles,
* Otherwise: 5 Cycles.

```

enum FloatFormat:
   FP16 = 0,
   FP32 = 1,

instruction CVTF2F(s: u4, d: u4, f: FloatFormat, k: FloatFormat, y: bool, r: RoundingMode):
    let rnd = y ? r : fpcsr >> 30;
    let val = ReadRegister(UNIT, s);
    let rounded = FpRoundToFormat(val, f, k, rnd);
    WriteRegister(UNIT, d, rounded);
    CheckAndThrowPendingUnitErrors();

instruction CVTI2F(s: u3, d: u4, i: bool, k: FloatFormat, y: bool, r: RoundingMode):
    let rnd = y ? r : fpcsr >> 30;
    let val = ReadRegister(UNIT, s);
    let rounded = FpRoundIntToFormat(val, i, k, rnd);
    WriteRegister(UNIT, d, rounded);
    CheckAndThrowPendingUnitErrors();

instruction CVTF2I(s: u4, d: u3, i: bool, f: FloatFormat, y: bool, RoundingMode):
    let rnd = y ? r : fpcsr >> 30;
    let val = ReadRegister(UNIT, s);
    let rounded = FpRoundToInt(val, i, k, rnd);
    WriteRegister(UNIT, d, rounded);
    CheckAndThrowPendingUnitErrors();
```

### Floating-point arithmetic

| Mnemonic | Function | Payload                |
| -------- | -------- | ---------------------- |
|          | `0--6`   | `0-----------------19` |
| `FADD`   | `0x08`   | `ddddaaaabbbb00000yrr` |
| `FSUB`   | `0x09`   | `ddddaaaabbbb00000yrr` |
| `FMUL`   | `0x0A`   | `ddddaaaabbbb00000yrr` |
| `FFMA`   | `0x0B`   | `ddddaaaabbbbcccc0yrr` |

```
instruction FADD()
```

!{#copyright}