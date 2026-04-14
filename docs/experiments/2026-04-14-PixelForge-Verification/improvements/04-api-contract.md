# 04. API Type Contract Honesty

## Problem (Meta M-03, originally R1 C-01)
`generate_frames(frame_size: int | tuple[int, int])` accepted tuples but silently discarded height. Non-square (64, 128) would produce 64x64.

## Discussion
- R1: Arbiter judged WEAK (UI only provides squares)
- Meta: Challenger re-argued as API contract violation -> Arbiter judged **VALID**

## Resolution
Non-square tuple raises explicit `ValueError("Non-square frame_size not supported")`.
Test: `test_non_square_frame_size_raises`
