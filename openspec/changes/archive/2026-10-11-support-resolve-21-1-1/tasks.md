# Tasks

## 1. Core Signature Engine Updates

- [x] 1.1 Implement `cmp_imm_reg` and `_v_cmp_reg_imm` in `aacpatch/sites.py` to disassemble candidate compare instructions, verify operand registers (`r8d`..`r15d`), and extract the 64-bit register name
- [x] 1.2 Update `gate1_dispatch` in `aacpatch/sites.py` to wildcard the ModRM byte (`41 81 ?? 33 70 6d 2e 0f 84 ?? ?? ?? ??`) and attach the Capstone verifier `_v_cmp_reg_imm(0x2e6d7033)`

## 2. Additive Applier Integration

- [x] 2.1 Update `resolve_addresses` in `aacpatch/additive.py` to resolve and return `g1_reg`, and update `build_e9tool_args` to pass `a["g1_reg"]` instead of `"r15"`
- [x] 2.2 Update address reporting in `aacpatch/additive.py` `main()` to format string register values cleanly alongside hexadecimal addresses

## 3. Documentation and Comments

- [x] 3.1 Update `docs/reverse-engineering.md` to explain compiler register allocation differences between 21.1 and 21.1.1 and the ModRM wildcarding approach
- [x] 3.2 Update `src/trampoline/aacadd.c` comment header to reflect that gate1 register is read off each build (`r15` vs `r12`)

## 4. Test Suite and Verification

- [x] 4.1 Add unit tests in `tests/test_signature.py` parameterizing `gate1_dispatch` across `0xff` (`r15`) and `0xfc` (`r12`), and asserting rejection of non-register ModRM forms
- [x] 4.2 Run pytest against `tests/test_signature.py` to verify that all existing and new signature tests pass

## Workflow follow-up

- Archive the change after implementation and test verification are complete.
