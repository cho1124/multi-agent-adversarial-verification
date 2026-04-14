# 01. Module Interface Unification

## Problem (Initial C-01, C-02)
app.py called postprocess.py and generator.py with completely wrong function names and parameter signatures. postprocess imports always failed -> only fallback used. generator.py would crash on ComfyUI connection success.

## Resolution
- **postprocess**: Unified to `quantize_palette`, `snap_to_pixel_grid`, `clean_transparency`, `build_shared_palette`, `apply_shared_palette`
- **generator**: Added `num_frames`, `server_url` alias, PIL Image `reference_image`, tuple `frame_size`
- **fallback**: All 5 functions implemented with identical signatures and matching behavior
