# 05. Unity Editor Integration

## Problem (Initial H-03, CK-26)
1. Manual JSON parser (string indexOf) fragile — negative values, whitespace, nested keys
2. Slicing coordinates ignored padding

## Resolution
- JSON parser replaced with `JsonConvert.DeserializeObject<T>()` (Newtonsoft.Json)
- Padding-aware slicing: `cellW = FrameWidth + pad * 2`, `x = col * cellW + pad`
