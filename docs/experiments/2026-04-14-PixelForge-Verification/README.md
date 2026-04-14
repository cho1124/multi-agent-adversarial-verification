# PixelForge 코드베이스 적대적 검증 실험 (2026-04-14)

> 로컬 AI 픽셀 아트 생성기(ComfyUI + SDXL + Unity 파이프라인)의 전체 코드베이스를 **3모델 적대적 검증 (Claude/Codex/Gemini)** 으로 검증. 9건 결함 수정 + 29개 자동화 테스트 구축.

---

## 결론

### 실무 결론: "적대적 검증은 코드 리뷰가 놓치는 인터페이스 결함을 구조적으로 발견한다"

| 지표 | 본검증 (4R + Meta) | 테스트 인프라 검증 (1R) | 합계 |
|------|-------------------|----------------------|------|
| 라운드 | 4R + 1 Meta | 1R | **6R** |
| 모순 제기 | 20건 | 10건 | **30건** |
| VALID | 2 (10%) | 5 (50%) | **7** |
| WEAK | 8 (40%) | 4 (40%) | **12** |
| INVALID | 10 (50%) | 1 (10%) | **11** |
| Collapse | 미발생 | 미발생 | **0** |

**핵심:** 초기 코드 리뷰에서 직접 발견한 결함 9건(C-01~M-05)에 더해, 적대적 검증이 추가로 2건의 VALID 결함(fallback 팔레트 클램핑 누락, API 타입 계약 위반)을 발견. 테스트 인프라 검증에서는 E2E 커버리지 부재, 폴링 미테스트 등 5건의 실질적 공백을 발견.

### 검증 단계별 발견 결함

| 단계 | 발견 결함 | 수정 | 방법 |
|------|----------|------|------|
| 초기 리뷰 (단일모델) | 9건 (C-01~M-05) | 전체 수정 | Claude 직접 코드 분석 |
| 본검증 R1 (3모델) | +1 VALID (팔레트 클램핑) | 수정 | Codex가 fallback 코드 정밀 분석 |
| 메타 검증 (재반박) | +1 VALID (tuple 계약 위반) | 수정 | Codex가 API 시그니처 재검증 |
| 본검증 R3-R4 | 0 VALID | — | 공격력 소진 (합의 도달) |
| 테스트 인프라 R1 | +5 VALID | 전체 수정 | Codex가 커버리지 갭 분석 |

### 3모델 독립 평가 (최종)

| 평가 항목 | Claude | Codex | Gemini | 평균 |
|-----------|--------|-------|--------|------|
| 코드 정합성 | 9/10 | 9/10 | 9/10 | **9.0** |
| 실행 가능성 | 8/10 | 8/10 | 7/10 | **7.7** |
| Fallback 안정성 | 9/10 | 9/10 | 9/10 | **9.0** |
| Unity 연동 | 9/10 | 9/10 | 9/10 | **9.0** |
| 개발 착수 가능 | 8/10 | 8/10 | 8/10 | **8.0** |
| **종합** | **8.6** | **8.6** | **8.4** | **8.5** |

### 검증된 것

1. **모듈 간 인터페이스 정합성** — postprocess↔app, generator↔app 함수명/시그니처 완전 일치
2. **ComfyUI 워크플로우 동적 바이패스** — IP-Adapter/ControlNet 노드 제거 후 그래프 무결성 유지 (테스트 검증)
3. **공유 팔레트 품질** — 프레임 간 색상 일관성 + 알파 채널 보존 (테스트 검증)
4. **Python ↔ Unity 데이터 계약** — 메타데이터 JSON 구조 + padding 좌표 계산 완전 호환
5. **스프라이트시트 라운드트립** — assemble→split 픽셀 값 완전 일치 (패딩 유/무)
6. **에러 경로 안정성** — 서버 미도달, 워크플로우 누락, 비정사각형 프레임 정상 처리

### 검증되지 않은 것 (한계)

1. **실제 ComfyUI GPU 생성** — Mock 서버로만 검증, 실제 SDXL 출력 품질 미확인
2. **Unity 에디터 실행** — C# 컴파일/런타임 미검증 (Newtonsoft.Json 패키지 필요)
3. **LoRA 모델 다운로드** — HuggingFace 경로 실제 존재 여부 미확인

---

## 수정 내역 (12건)

### Critical (4건)

| # | 파일 | 수정 | 라운드 |
|---|------|------|--------|
| 1 | app.py ↔ postprocess.py | import 함수명 통일 (3함수 → 5함수) | 초기 |
| 2 | app.py ↔ generator.py | API 시그니처 통일 (num_frames, server_url, PIL ref) | 초기 |
| 3 | setup.py | Pixel Art LoRA 모델 다운로드 추가 | 초기 |
| 4 | generator.py | IP-Adapter/ControlNet optional 바이패스 | 초기 |

### High (4건)

| # | 파일 | 수정 | 라운드 |
|---|------|------|--------|
| 5 | postprocess.py + app.py | 공유 팔레트 구현 (build + apply) | 초기 |
| 6 | spritesheet.py | split_spritesheet padding 지원 | 초기 |
| 7 | SpriteSheetPostprocessor.cs | Newtonsoft.Json 파서 + padding 좌표 | 초기 |
| 8 | postprocess.py | snap_to_pixel_grid 크기 불일치 수정 | 초기 |

### 적대적 검증에서 추가 발견 (2건)

| # | 파일 | 수정 | 라운드 |
|---|------|------|--------|
| 9 | postprocess.py + app.py | build_shared_palette num_colors 클램핑 + putdata tuple 변환 | R1 VALID |
| 10 | generator.py | non-square frame_size ValueError 방어 | Meta VALID |

### 테스트 인프라 (2건)

| # | 파일 | 수정 | 라운드 |
|---|------|------|--------|
| 11 | generator.py | BYPASS_TITLE_KEYWORDS 상수 추출 → preflight 자동 동기화 | 테스트R1 VALID |
| 12 | mock_comfyui.py | /history 폴링 시뮬레이션 (polls_before_ready) | 테스트R1 VALID |

---

## 테스트 자동화 인프라

### 구성

```
tests/
├── __init__.py
├── conftest.py          # session-scoped mock 서버 fixture
├── mock_comfyui.py      # Mock ComfyUI (노드 레지스트리 + 워크플로우 검증)
├── preflight.py         # Pre-flight CLI (그래프 + 타이틀 + 모델 파일)
└── test_pipeline.py     # 29개 테스트 (3.07초)
```

### 테스트 커버리지

| 카테고리 | 수 | 대상 |
|----------|---|------|
| Pre-flight | 2 | 워크플로우 그래프 + 타이틀 의존성 |
| 워크플로우 바이패스 | 7 | IP-Adapter/ControlNet 동적 제거 + 재배선 |
| 후처리 | 6 | 공유 팔레트, 그리드 스냅, 투명도 |
| 스프라이트시트 | 3 | 조립, 분할, 메타데이터 |
| Mock 서버 | 4 | 헬스, 큐잉, 검증, 폴링 시뮬레이션 |
| E2E 생성 | 2 | 전체 파이프라인 (ref 유/무) |
| 에러 경로 | 3 | 서버 미도달, 비정사각형, 워크플로우 누락 |
| 픽셀 라운드트립 | 2 | 패딩 유/무 픽셀 값 완전 일치 |
| **합계** | **29** | **3.07초** |

---

## WEAK 항목 (인지, 미수정)

| # | 내용 | 사유 |
|---|------|------|
| 1 | ControlNetApply(단일 output) 바이패스 미래 호환 | 현재 워크플로우 미사용 |
| 2 | build_shared_palette strip 대규모 입력 메모리 | 현실적 규모 문제 아님 |
| 3 | reference_image 이중 업로드 I/O | overwrite=true, idempotent |
| 4 | HuggingFace 모델 경로 하드코딩 | 기존 패턴 일관, 실패 시 예외 |
| 5 | ordered_dither num_colors vs levels^3 | 표준 기법, 파라미터명 혼동 가능 |
| 6 | 후처리 순서 불일치 (apply_pipeline vs app.py) | 독립 경로, 동시 미사용 |
| 7 | Mock 서버 클래스 변수 테스트 격리 | 현재 테스트 간 간섭 없음 |
| 8 | validate_workflow 타입 검증 부재 | 존재 여부 검증으로 주요 결함 커버 |

---

## 프로젝트 정보

- **레포지토리**: https://github.com/cho1124/PixelForge
- **커밋**: `e35453e` (feat: adversarial verification — 9건 결함 수정 + 테스트 자동화 인프라)
- **기술 스택**: Python (ComfyUI + Gradio + Pillow) + Unity C# (Editor Scripts)
- **모델**: Executor=Claude Opus 4.6, Challenger=Codex (GPT-5.2), Arbiter=Gemini 3 Pro
