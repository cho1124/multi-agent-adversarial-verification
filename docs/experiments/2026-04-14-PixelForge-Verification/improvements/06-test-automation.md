# 06. Test Automation Infrastructure

## Problem (3-model eval: "execution viability" 7-8/10)
ComfyUI integration only verifiable at runtime. Bypass logic depends on _meta.title strings. LoRA file paths uncertain.

## From Test Infra Adversarial Verification (5 VALID)
- C-02: /history polling not tested -> Added polling simulation
- C-04: No E2E generate_frames test -> Added 2 E2E tests
- C-07: No error path tests -> Added 3 error tests
- C-08: REQUIRED_TITLES not synced -> Auto-import from generator.py
- C-10: No pixel value verification -> Added roundtrip equality tests

## Result: 29 tests, 3.07s
`pytest tests/` validates entire pipeline without GPU.
