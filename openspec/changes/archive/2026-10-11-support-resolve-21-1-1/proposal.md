# Proposal

## Why

In DaVinci Resolve Studio 21.1.1, the compiler changed the register allocated for holding the QuickTime audio fourcc from `r15d` to `r12d` during dispatch. As a result, the `gate1_dispatch` byte signature (which hardcoded the ModRM byte `0xff` for `cmp r15d, '.mp3'`) fails to match in Resolve 21.1.1 binaries, causing `aac-fix` and the patcher engine to refuse patching the 21.1.1 build. Pull Request #3 resolves this by wildcarding the ModRM byte, verifying with Capstone that the instruction is `cmp <r8d..r15d>, '.mp3'`, and dynamically passing the detected register to the e9tool trampoline.

## What Changes

- **Wildcard ModRM byte in `gate1_dispatch`**: Update the `gate1_dispatch` signature in `aacpatch/sites.py` from `41 81 ff ...` to `41 81 ?? ...` to locate the `.mp3` compare instruction regardless of whether the compiler used `r12d`, `r15d`, or another extended register (`r8d`..`r15d`).
- **Dynamic register detection and verification**: Introduce `cmp_imm_reg` and verification logic in `aacpatch/sites.py` to ensure matched instructions are strictly `cmp <r8d..r15d>, 0x2e6d7033` (rejecting memory operands or non-compare instructions), and retrieve the 64-bit register name.
- **Pass detected register to e9tool trampoline**: In `aacpatch/additive.py`, read `g1_reg` dynamically from the matched binary and pass it to e9tool's trampoline invocation instead of the hardcoded `"r15"`.
- **Formatting and test updates**: Format register outputs cleanly in `aacpatch/additive.py`, add unit tests in `tests/test_signature.py` covering both `r15` (21.0.x-21.1) and `r12` (21.1.1) as well as rejecting invalid ModRM encodings, and update documentation in `docs/reverse-engineering.md` and comments in `src/trampoline/aacadd.c`.

## Capabilities

### New Capabilities

- `quicktime-aac-dispatch`: Locates and dynamically routes the QuickTime audio fourcc dispatch gate (gate1) to the FFmpeg audio codec handler across variable register allocations in Resolve 21 versions (including 21.0.x, 21.1, and 21.1.1).

### Modified Capabilities

<!-- None: no pre-existing specs in openspec/specs/ -->

## Impact

- **Affected code**: `aacpatch/sites.py`, `aacpatch/additive.py`, `src/trampoline/aacadd.c`, `docs/reverse-engineering.md`, and `tests/test_signature.py`.
- **Dependencies**: Uses `capstone` (already a project dependency) for instruction disassembly and register inspection.
- **Compatibility**: Retains full backwards compatibility with Resolve 21.0.x and 21.1 while adding compatibility for 21.1.1.
