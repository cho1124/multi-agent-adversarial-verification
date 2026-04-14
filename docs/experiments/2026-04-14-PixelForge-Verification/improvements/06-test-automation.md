# 06. 테스트 자동화 인프라

## 문제 (3모델 평가에서 "실행 가능성" 7~8/10)

ComfyUI 연동이 실제로 동작하는지는 런타임에서만 확인 가능. 워크플로우 노드 바이패스가 `_meta.title` 문자열에 의존. LoRA 파일 경로 불확실.

## 테스트 인프라 적대적 검증에서 발견된 공백 (5 VALID)

| ID | Challenger 지적 | Arbiter 판정 |
|----|----------------|-------------|
| C-02 | /history 즉시 success → 폴링 미테스트 | **VALID** |
| C-04 | generate_frames E2E 테스트 없음 | **VALID** |
| C-07 | 에러 경로 테스트 부재 | **VALID** |
| C-08 | REQUIRED_TITLES generator.py와 동기화 안됨 | **VALID** |
| C-10 | assemble→split 픽셀 값 미검증 | **VALID** |

## 최종 해소

### Mock ComfyUI 서버 (mock_comfyui.py)
- 14개 노드 타입 레지스트리
- 워크플로우 그래프 정적 검증 (노드 존재, 필수 입력, 링크 참조, 출력 슬롯)
- 폴링 시뮬레이션 (`polls_before_ready` — 첫 N회 빈 응답 후 success)
- 더미 이미지 반환

### Pre-flight 검증기 (preflight.py)
- 그래프 무결성 검증
- `_meta.title` 의존성 자동 동기화 (generator.py `BYPASS_TITLE_KEYWORDS` import)
- 모델 파일 존재 확인 (--comfyui-path 옵션)

### 29개 테스트 (3.07초)
- E2E: `generate_frames` 전체 경로 (ref 유/무)
- 에러: ServerUnreachableError, ValueError, WorkflowError
- 픽셀: assemble→split 라운드트립 완전 일치 (padding 유/무)

## 개선된 점

- `pytest tests/` 한 줄로 전체 파이프라인 3초 자동 검증
- 코드 수정 시 바이패스 그래프 파손 즉시 탐지
- preflight가 generator.py와 자동 동기화 → 이중 관리 제거
