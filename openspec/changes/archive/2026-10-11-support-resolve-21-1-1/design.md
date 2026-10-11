# Design

## Context

See `proposal.md` for motivation.

The QuickTime AAC patch relies on `gate1_dispatch` to hook Resolve's audio format check. In Resolve 21.0.0 through 21.1.0, the check was compiled as `cmp r15d, 0x2e6d7033` (ModRM `0xff`), jumped to the FFmpeg handler via `je`, and e9tool hooked `r15`. In Resolve 21.1.1, the compiler allocated `r12d` (ModRM `0xfc`).

## Goals / Non-Goals

**Goals:**
- Make `gate1_dispatch` agnostic to the compiler's register allocation among extended registers (`r8d` through `r15d`).
- Statically verify candidate sites using Capstone disassembly to prevent false-positive matches (such as memory dereferences or differing opcodes).
- Dynamically provide the located register to e9tool's trampoline argument builder.
- Add test coverage for both legacy (`r15d`) and new (`r12d`) patterns and invalid ModRM encodings.

**Non-Goals:**
- Changes to other patch gates (Gate 2, Gates 3a-c, Gate 4, or MKV gates), which remain unchanged between 21.1 and 21.1.1.
- Changes to the C trampoline logic in `src/trampoline/aacadd.c` (the trampoline already accepts any register passed by e9tool).
- Supporting non-x86_64 architectures or non-Linux platforms.

## Decisions

### Decision 1: Wildcard the ModRM byte in `gate1_dispatch` signature
- **Choice**: Replace `ff` with `??` in `41 81 ff 33 70 6d 2e 0f 84 ?? ?? ?? ??` -> `41 81 ?? 33 70 6d 2e 0f 84 ?? ?? ?? ??`.
- **Rationale**: The REX prefix `0x41` combined with opcode `0x81` and immediate `0x2e6d7033` (`.mp3`) provides a strong anchor. Wildcarding byte offset 2 accommodates any extended register ModRM encoding.
- **Alternative considered**: Having two separate signatures for 21.1 and 21.1.1. Rejected because it fragments the signature set and does not generalize if subsequent point releases re-allocate registers again.

### Decision 2: Verify candidate instruction with Capstone (`_v_cmp_reg_imm`)
- **Choice**: Implement a verifier callback on `Signature` that disassembles the instruction and validates it is strictly `cmp <reg>, <imm>` where `<reg>` is one of `r8d`..`r15d`.
- **Rationale**: Wildcarding ModRM allows non-register encodings (such as memory dereferences `[r15]`). Capstone verification ensures only valid register compare instructions match.
- **Alternative considered**: Decoding ModRM bits manually via bitwise operations. Rejected because Capstone is already an active dependency and instruction disassembly is more readable and less error-prone.

### Decision 3: Dynamically derive the 64-bit register for e9tool (`cmp_imm_reg`)
- **Choice**: Parse the 32-bit register from Capstone (`r12d`, `r15d`) and strip the trailing `d` to yield the 64-bit register name (`r12`, `r15`) passed to e9tool trampoline `R(reg, AAC_FOURCC, target)`.
- **Rationale**: e9tool expects 64-bit register names in trampoline predicates (`-P`). Deriving this directly from the disassembled binary keeps `additive.py` decoupled from hardcoded assumptions.

## Risks / Trade-offs

- **[Risk]** The compiler could potentially switch from an extended register (`r8d..r15d`, requiring REX prefix `0x41`) to a legacy register (`eax`, `ebx`, `ecx`, `edx`, `esi`, `edi`).
  → **Mitigation**: The signature requires `0x41 0x81`. If a non-extended register is used, the patcher will fail safely with a clean `LookupError` rather than patching an invalid address.
- **[Risk]** Potential ambiguity if another instruction in `.text` has the same pattern.
  → **Mitigation**: `Signature.locate` strictly asserts exactly one hit across all executable ranges, throwing `LookupError` if more than one candidate matches.
