# 01. 모듈 간 인터페이스 통일

## 문제 (초기 검증 C-01, C-02)

app.py가 postprocess.py와 generator.py를 호출할 때 함수명/파라미터가 완전히 불일치. postprocess 모듈의 실제 함수는 항상 import 실패 → fallback만 사용. generator.py는 ComfyUI 연결 성공 시 즉시 크래시.

## 최종 해소

- **postprocess**: `apply_palette_quantization` → `quantize_palette`, `apply_pixel_grid_snap` → `snap_to_pixel_grid`, `cleanup_transparency` → `clean_transparency` + `build_shared_palette`, `apply_shared_palette` 추가
- **generator**: `num_frames`, `server_url`(alias), PIL Image `reference_image`, tuple `frame_size` 지원
- **fallback**: 5개 함수 전부 원본과 동일 시그니처/동작으로 구현

## 개선된 점

- ComfyUI 온라인 시 실제 postprocess 모듈 사용 (dead code 제거)
- generator.py ↔ app.py API 완전 호환
