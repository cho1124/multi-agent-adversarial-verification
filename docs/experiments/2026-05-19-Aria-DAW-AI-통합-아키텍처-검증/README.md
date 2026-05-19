# Aria DAW AI 통합 아키텍처 적대적 검증 (2026-05-19)

> Claude Opus 4.7의 추천이 통째로 폐기되고 Challenger의 대안 패러다임이 채택된 **최초 사례**.
> Aria — 학습/포트폴리오 목적의 C++/JUCE 8 기반 AI 보조 DAW.

---

## 결론

### 실무 결론: "Executor의 학습 가치 frame이 객관적 결함을 가렸고, Challenger가 frame 자체를 깨뜨림"

| 지표 | 검증 전 (Claude 추천) | 검증 후 (Arbiter 채택) |
|------|---------------------|----------------------|
| 아키텍처 | 옵션 A: ONNX Runtime + JUCE C++ 단일 프로세스 | **새 옵션**: LLM Copilot + Python IPC/API |
| AI 모델 | Magenta-계열 (구형 RNN/LSTM) | llama.cpp 로컬 또는 OpenAI/Anthropic API (M11에 최종 결정) |
| 패러다임 | MIDI 멜로디 생성 ("다음 마디 채워주세요") | 자연어 → DSL → MIDI 렌더링 ("LLM Copilot") |
| 타이밍 | M9~M10 (사인파 → 8세션 만에 완성) | **M12+** (DAW 코어 안정화 후) |
| 단일 프로세스 집착 | 강함 (학습 가치 명분) | 폐기 (실용성 우선) |

### 핵심 발견

1. **단일 자기부여 제약이 모든 결함의 root cause** — Executor가 'AI 추론의 C++ 단일 프로세스 내장'을 처음부터 가정한 것이 옵션 A 전체의 SPOF를 만들었음. 이 제약 포기 → C-01/C-02/C-03 동시 해소
2. **'학습 가치' frame이 의사결정 흐림** — Executor가 구형 기술 스택과 ONNX 변환 위험성을 'C++ inference 학습 가치'로 정당화 시도 (Arbiter: frame_shift=true)
3. **생태계 최신화 실패 (구식 전제)** — Executor의 Phase 1 web search가 'Magenta RealTime Atom active' 결론을 도출했으나, Challenger가 RNN/LSTM 모델의 ONNX 변환 현실성에 대해 더 정확한 판단 제시
4. **MAGDA의 존재가 차별성 frame을 무너뜨림** — 이미 JUCE + LLM 자연어 제어 DAW가 active 개발 중인 상황에서 단순 MIDI 생성은 차별성 ↓

---

## 실험 설계

| 항목 | 값 |
|------|-----|
| 검증 주제 | Aria DAW의 AI 통합 아키텍처 결정 (M9~M10 시점 가정) |
| Executor | Claude Opus 4.7 (claude-opus-4-7, 메인 세션) |
| Challenger | Gemini 3 Pro (gemini-cli 0.25.1, yolo mode, Bash 직접 호출) |
| Arbiter | Gemini 3 Pro (별도 호출, fresh context) |
| 라운드 수 | 1 (시간 제약 — 영준님 1시간 budget 내 압축 진행) |
| 총 모순 | 4건 |
| VALID | **4건 (전건 VALID)** |
| WEAK | 0건 |
| INVALID | 0건 |
| Collapse | 없음 (rational_consensus 달성) |
| Frame Shift | **true** (Executor의 '학습 가치 최대화' 프레임) |
| Executor 추천 채택 여부 | **❌ 폐기, 새 옵션 채택** |
| 사람 결정 | 영준님이 Arbiter 권고 전면 수용 |

### 모델 다양성 한계

Codex CLI 서브에이전트(`codex:codex-rescue`)가 외부 폴더 접근 불가 + 임베드 무시 환각 패턴(이전 Layer 2 시도에서 2회 실패)로 인해 사용 불가. Challenger와 Arbiter 모두 Gemini 3 Pro로 진행. 진정한 모델 독립성 약함 — 같은 모델 fresh context 호출로 우회 시도.

→ 별도 메모리 등록: `feedback_layer2_codex_vs_gemini.md`

---

## 라운드별 진행

### Round 1: Challenger 4건 제기

| ID | 유형 | 대상 | 심각도 | 판정 |
|----|------|------|:------:|:----:|
| C-01 | ecosystem_outdated + assumption_flaw | Magenta active 가정 + RNN/LSTM ONNX 변환 SPOF | high | **VALID** |
| C-02 | logic_error | 단일 프로세스 진입 단순성 평가 (오디오 스레드 lock-free vs ONNX 추론 alloc/blocking 모순) | high | **VALID** |
| C-03 | frame_shift | '학습 가치/포트폴리오 차별성' frame, MAGDA의 LLM 에이전트 대비 차별성 부재 | medium | **VALID** |
| C-04 | assumption_flaw | M9~M10 일정 물리적 비현실성 (사인파→DAW 코어+AI 통합 = 8세션) | high | **VALID** |

### Arbiter 판정 종합

- **Challenger 상태**: rational_consensus
- **Executor frame_shift**: **true** ('학습 가치 최대화' 프레임으로 구형 기술 스택 정당화 시도)
- **의존성 체인**: C-01/C-02/C-03 모두 단일 root cause로 수렴 — 'AI 추론의 C++ 단일 프로세스 내장' 제약. 이 제약 포기 = 단일 피벗으로 다수 해소
- **에스컬레이션 필요**: AI 패러다임 선택 (Copilot vs MIDI 생성), C++ 단일 프로세스 집착 포기 여부

### 최종 결정 (Arbiter)

- **옵션**: **새 옵션 — LLM Copilot + Python IPC/API**
- **일정**: M12+ 로 연기
- **이유**:
  1. 옵션 A는 레거시 모델 ONNX 변환 리스크 + C++ 비동기 통합 복잡성으로 좌초 위험
  2. LLM Copilot은 구현 난이도 ↓ + 2026 트렌드 부합 + 포트폴리오 가치 ↑
  3. DAW 코어 없는 상태에서 AI 얹는 건 모래 위에 집 짓는 격

---

## 사람 결정 (영준님)

| 에스컬레이션 항목 | 영준님 결정 |
|---|---|
| Arbiter 권고 수용 여부 | **전면 수용** |
| AI 패러다임 | **LLM Copilot (Challenger 권고)** |
| 단일 프로세스 집착 | 포기 (Python IPC/API 수용) |
| 일정 | M12+ 연기 수용 |

---

## 해소/미해소 분류

### 해소 (4건, 단일 피벗으로 동시 해소)
- C-01: 단일 프로세스 폐기 → ONNX 변환 우회 (LLM API 사용)
- C-02: 외부 IPC로 격리 → 오디오 스레드와 추론 완전 분리
- C-03: 패러다임 자체 전환 → MAGDA와 차별 (코드 생성 task로서의 DSL)
- C-04: M12+로 일정 연기 → 코어 안정화 후 AI 도입

### 미해소 (0건) — 모두 단일 피벗으로 해소 가능

---

## Validation Stack에서의 의미

Aria 프로젝트의 환각 방지 4계층 중 **Layer 3 (Adversarial Triad) 가동 첫 사례**.

| Layer | 결과 |
|---|---|
| L1 Web Verify | ✅ Phase 1 리서치 (불완전 — Challenger가 더 정확) |
| **L3 Adversarial Triad** | ✅ **Executor 추천 패배 + 더 나은 방향 도출** |
| L2 Codex Review | ✕ (codex 서브에이전트 sandbox 한계, [[feedback_layer2_codex_vs_gemini]] 참조) |
| L4 Build Gate | — (코드 변경 없음) |

이번 검증의 의미:
- Layer 3가 단순 sanity check를 넘어 **실질적인 방향 전환을 만들어냄**
- Executor(Claude)가 자기 frame에 갇혀 있던 걸 Challenger가 깨뜨림
- "환각 방지 인프라가 실제 본전 뽑은 두 번째 사례" (첫 사례는 Gemini Layer 2 코드 리뷰)

---

## 프로세스 개선 도출

이번 검증에서 도출된 패턴 / 향후 개선 후보:

| 개선 | 출처 | 효과 |
|------|------|------|
| Layer 2 모델 선택 가이드 | Codex 서브에이전트 sandbox 차단 | Gemini CLI를 외부 폴더 코드 리뷰 우선 채택 |
| Frame Shift 조기 탐지 | 'C++ 단일 프로세스 학습 가치' frame | Executor가 자기부여 제약을 정당화로 사용하는 패턴 |
| 의존성 체인 단일 피벗 식별 | C-01/C-02/C-03 root cause 공유 | 개별 대응 대신 root cause 해소 권고 |
| 시간 압축 1라운드 모드 | 1시간 budget 제약 | 풀 프로토콜 (50라운드) 대비 빠른 의사결정 가능, 단 모순 깊이 제한 |

---

## 한계

1. **Challenger와 Arbiter 동일 모델 (Gemini 3 Pro)** — 진정한 모델 독립성 약함. Codex 서브에이전트 차단으로 회피. 향후 Codex CLI 직접 호출 시도 가치 있음
2. **1라운드 압축** — 풀 프로토콜 (다수 라운드 + 해소 재검증) 미적용. C-04 (일정)는 더 깊은 분석 여지 있을 수 있음
3. **PoC 미실시** — Arbiter 권고대로 1주일 내 LLM Copilot PoC 진행해야 결정 신뢰도 추가 확보
4. **Phase 1 리서치 한계** — Executor의 web search가 Magenta active 여부를 부정확하게 결론. Challenger의 'PyTorch 중심 생태계' 주장도 추가 검증 필요

---

## 관련 자료

- 검증 대상 문서: `https://github.com/cho1124/Aria/blob/main/docs/AI_INTEGRATION_PROPOSAL_DRAFT.md` (Claude 초안, 폐기됨)
- 적대적 검증 전문: `https://github.com/cho1124/Aria/blob/main/docs/AI_INTEGRATION_VERIFICATION.md`
- 정식 플랜: `https://github.com/cho1124/Aria/blob/main/docs/AI_INTEGRATION_PLAN.md`
- Aria 프로젝트 리포: `https://github.com/cho1124/Aria` (private)
