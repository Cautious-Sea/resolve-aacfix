# quicktime-aac-dispatch Specification

## Purpose
Enables reliable location and trampoline hooking of DaVinci Resolve's QuickTime audio format dispatch gate across compiler register allocation variations in Resolve 21 point releases.

## Requirements

### Requirement: Dynamic Gate 1 Fourcc Register Detection
The signature engine SHALL locate the gate1 dispatch `.mp3` compare instruction across variable register encodings (`r8d` through `r15d`) and determine the matching 64-bit register name.

#### Scenario: Resolve 21.0 to 21.1 binary using r15d
- **WHEN** locating gate1 in a binary where the dispatch instruction disassembles as `cmp r15d, 0x2e6d7033`
- **THEN** the site is successfully located and the fourcc register is identified as `r15`

#### Scenario: Resolve 21.1.1 binary using r12d
- **WHEN** locating gate1 in a binary where the dispatch instruction disassembles as `cmp r12d, 0x2e6d7033`
- **THEN** the site is successfully located and the fourcc register is identified as `r12`

#### Scenario: Rejection of non-register compare instructions
- **WHEN** candidate bytes match the opcode prefix but decode to a memory-based compare or an invalid register
- **THEN** signature verification fails and the location is rejected

### Requirement: Trampoline Dispatch with Dynamic Register
The additive patch applier SHALL supply the dynamically resolved 64-bit fourcc register to the e9tool trampoline argument generator for gate1 dispatch.

#### Scenario: Gate 1 e9tool argument generation
- **WHEN** building e9tool arguments for a binary with resolved gate1 register `REG` and target address `TGT`
- **THEN** the gate1 patch trampoline is configured to evaluate register `REG` against fourcc `0x61616320` and redirect to `TGT`
