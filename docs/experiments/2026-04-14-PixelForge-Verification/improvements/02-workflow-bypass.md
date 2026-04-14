# 02. ComfyUI Workflow Dynamic Bypass

## Problem (Initial C-04)
Reference image required for workflow execution. IP-Adapter nodes (6,7,8,9) hardwired to load `reference.png`.

## Resolution
- `_bypass_ipadapter()`: Remove IPAdapter chain, rewire KSampler model to LoRA output
- `_bypass_controlnet()`: Remove ControlNet chain, rewire KSampler conditioning to CLIPTextEncode
- **7 tests** verify graph integrity after bypass (single, dual, node removal, rewiring)
