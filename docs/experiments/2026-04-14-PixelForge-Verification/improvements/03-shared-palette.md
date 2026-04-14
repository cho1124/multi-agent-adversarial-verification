# 03. Shared Palette — Inter-frame Color Consistency

## Problem (Initial H-01)
Independent per-frame quantization caused color flickering during animation playback. Fatal for pixel art.

## Resolution
- `build_shared_palette()`: Collect opaque pixels from all frames -> single MEDIANCUT palette
- `apply_shared_palette()`: Apply shared palette preserving alpha channel
- R1 fix: num_colors clamping + putdata tuple conversion (caught by test)
