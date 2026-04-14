# 02. ComfyUI 워크플로우 동적 바이패스

## 문제 (초기 검증 C-04)

Reference image 없이 워크플로우 실행 불가. IP-Adapter 노드(6,7,8,9)가 하드와이어링되어 `reference.png` 파일이 없으면 ComfyUI 에러. ControlNet도 동일.

## 논의 과정

| 라운드 | 제안/반박 | 결과 |
|--------|----------|------|
| 초기 | _bypass_ipadapter + _bypass_controlnet 구현 | 노드 제거 + 재배선 |
| R1 C-02 | ControlNetApply(단일 output) 호환성 | INVALID — 현재 워크플로우 미사용 |
| Meta M-01 | 미래 워크플로우에서 ControlNetApply 사용 시 버그 | WEAK — 인지 |

## 최종 해소

- `_bypass_ipadapter()`: IPAdapterAdvanced, IPAdapterModelLoader, CLIPVisionLoader, LoadImage[reference] 제거 → KSampler model을 LoRA 출력으로 재배선
- `_bypass_controlnet()`: ControlNetApplyAdvanced, ControlNetLoader, LoadImage[pose] 제거 → KSampler positive/negative를 CLIPTextEncode로 재배선
- **테스트**: 7개 바이패스 테스트가 그래프 무결성 + 재배선 정확성 자동 검증

## 개선된 점

- Reference image, pose image 모두 optional
- 바이패스 후에도 ComfyUI 워크플로우 그래프 무결성 유지 (테스트 보증)
