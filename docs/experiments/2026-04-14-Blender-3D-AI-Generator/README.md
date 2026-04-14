# Blender 3D AI Generator 적대적 검증 실험 (2026-04-14)

> 텍스트/이미지 → 3D 모델 생성 AI 시스템(Blender 애드온, API 기반)의 전체 아키텍처 설계를 적대적 검증으로 검증.
> 14라운드, 58건 모순, 100% 해소, Rational Consensus 도달.

---

## 실험 파라미터

| 파라미터 | 값 |
|---------|-----|
| Executor 모델 | Claude Opus 4.6 (claude-opus-4-6) |
| Challenger 모델 | Codex GPT-5.2 (codex:codex-rescue 서브에이전트) |
| Arbiter 모델 | Gemini 3 Pro (gemini-cli 0.25.1) |
| 프롬프트 버전 | v2 (프레임 전환 검증 포함) |
| 종료 조건 | 동적 종료 + 인간 검증자 추가 요청 시 계속 |
| 총 라운드 | 14 |
| 합의 시도 | 4회 (R7, R9, R11, R14) |

---

## 결론

### 실무 결론: "빈 배열 합의를 3번 뚫고 들어가야 진짜 합의다"

| 지표 | 초기 설계 (R1) | 1차 합의 (R7) | 최종 확정 (R14) |
|------|---------------|--------------|----------------|
| 모순 발견 | 0건 | 38건 해소 | **58건 해소** |
| Timer 아키텍처 | on-demand Timer | 상시 500ms | 상시 500ms + **재진입 방지** |
| Queue 구조 | 단일 Queue | 2개 Queue | 2개 Queue + **배압/용량 제한** |
| 임포트 방식 | Timer에서 직접 | Timer + Undo 시도 | **Operator 전용** (Undo 보장) |
| 외부 의존성 | keyring, pydantic, rembg | keyring만 | **0개** (표준 라이브러리만) |
| 상태머신 | 3상태 | 5상태 | 6상태 + **failed→재시도** |
| 에러 계층 | 4단계 | 5단계 | **6단계** (+FileCorruption) |
| 파일 무결성 | 없음 | 없음 | **magic bytes + 크기 검증** |
| 프로필 구조 | 단일 | 단일 | **2계층** (portable/provider) |

**핵심 발견:**
1. **1차 합의(R7)는 허위 합의였다** — R8에서 Blender 특수성(Undo, 의존성) 5건 발견
2. **2차 합의(R9)도 불완전** — R10에서 상태머신, 파일 무결성 등 6건 발견
3. **3차 합의(R11)도 부족** — R12에서 테스팅, 워크플로우, 인코딩 등 6건 발견
4. **4차 합의(R14)에서 비로소 진정한 합의** — 총 20건의 "합의 후 추가 발견"이 설계를 근본적으로 강화

### 검증 수렴 패턴

```
R1:  10건 ████████████████████
R2:   8건 ████████████████
R3:   6건 ████████████
R4:   5건 ██████████    ← frame_shift (Timer 재설계)
R5:   5건 ██████████
R6:   4건 ████████
R7:   0건              ← 1차 합의 (허위)
R8:   5건 ██████████    ← 재검토 발견 (Blender 특수성)
R9:   0건              ← 2차 합의 (불완전)
R10:  6건 ████████████  ← 재검토 발견 (데이터 무결성)
R11:  0건              ← 3차 합의 (부족)
R12:  6건 ████████████  ← 재검토 발견 (운영/배포)
R13:  3건 ██████
R14:  0건              ← 최종 합의 (Rational Consensus)
```

**관찰:** 빈 배열 후 새로운 관점에서 재검토하면 평균 5.7건의 추가 모순이 발견됨. Challenger의 "합의 편향"을 시사하며, **인간 검증자의 "더 돌려라" 개입이 검증 품질에 결정적**.

### 검증된 것

1. **프레임 전환 검증 작동** — R4에서 Arbiter가 frame_shift 판정 → 아키텍처 재설계
2. **다각도 재검토의 가치** — 합의 후 3회 추가 검토에서 총 20건 추가 (전체의 34.5%)
3. **도메인 특수성 발견** — Blender API 스레드 안전성, Undo 시스템, 번들 Python 등
4. **VALID 비율 94.8%** — 58건 중 55 VALID, 3 WEAK, 0 INVALID
5. **Collapse 없음** — 14라운드 Challenger 논리적 일관성 유지

### 검증되지 않은 것 (한계)

1. **실제 구현 미검증** — 설계만 검증 완료
2. **API 실제 동작 미검증** — Meshy/Tripo3D 실제 응답 형식 미확인
3. **사용자 테스트 미실시** — UX 관련 설계는 실사용 시 검증 필요
4. **재현성** — 온도/시드 미고정
5. **Challenger 합의 편향** — 3회 허위 합의. 인간 개입 없이는 조기 종료

---

## 기술적 개선 사항

| # | 개선 사항 | 핵심 | 라운드 |
|---|----------|------|--------|
| 1 | [Timer 아키텍처 재설계](improvements/01-Timer-아키텍처.md) | on-demand→상시 500ms, 재진입 방지, 2개 Queue | R1-R5 |
| 2 | [외부 의존성 제거](improvements/02-외부-의존성-제거.md) | rembg/pydantic/keyring 제거, 표준 라이브러리만 | R1,R8,R9 |
| 3 | [상태머신 확장](improvements/03-상태머신-확장.md) | 3→6상태, failed→재시도, processing 고착 방지 | R10-R11 |
| 4 | [Undo 안전 임포트](improvements/04-Undo-안전-임포트.md) | Operator 전용, 임포터 레지스트리, 자동 오프셋 | R8,R10,R12 |
| 5 | [파일 무결성 검증](improvements/05-파일-무결성.md) | magic bytes + 크기 + Content-Type | R10-R11 |
| 6 | [비용 통제](improvements/06-비용-통제.md) | CompareRequest, 사전 추정, 예산 분리 | R5-R6,R10 |
| 7 | [프로필 이식성](improvements/07-프로필-이식성.md) | 2계층(portable/provider), JSON export/import | R12-R13 |
| 8 | [인코딩/국제화](improvements/08-인코딩.md) | UTF-8 통일, utf-8-sig BOM, 프롬프트 truncation | R10,R12-R13 |
| 9 | [테스트 전략](improvements/09-테스트-전략.md) | 3계층(단위/headless/수동), bpy 모킹 안함 | R12-R13 |

---

## 최종 확정 아키텍처

```
┌─────────────────────────── Blender 4.0+ ───────────────────────────┐
│                                                                      │
│  [User Input]──→ Operator.execute() ──→ request_queue (maxsize=5)   │
│       │              │ EXIF strip            │ put_nowait            │
│       │              │ 예산 체크              ↓                      │
│       │              │ 프롬프트 빌더    ┌─── Worker Thread ───┐     │
│       │              │ (길이 truncation) │  get() → API 호출   │     │
│       │              │                  │  (순수 네트워크 I/O) │     │
│       │              │                  │  파일 무결성 검증    │     │
│       │              │                  │  → result_queue.put()│     │
│       │              │                  └─────────────────────┘     │
│       │              │                        ↓                      │
│       │    ┌─── Timer 500ms (재진입 방지) ───────────────┐          │
│       │    │  1. result_queue.get() (최대 1건)            │          │
│       │    │     → DB write + UI "Import Ready" 표시     │          │
│       │    │  2. 30초 주기 유지보수 (TTL/아카이브)        │          │
│       │    │  3. ui_needs_refresh 체크 → UI 동기화       │          │
│       │    └─────────────────────────────────────────────┘          │
│       │                        ↓                                     │
│       └──→ "Import" 버튼 ──→ Operator.execute()                    │
│                                  │ 임포터 레지스트리 (런타임 감지)   │
│                                  │ 자동 오프셋 배치                  │
│                                  └→ Undo 스택 등록                  │
│                                                                      │
│  ┌─── SQLite (CONFIG, journal, 메인스레드 전용) ───┐               │
│  │  generations │ tasks │ usage │ settings          │               │
│  │  상태: requested→processing→completed|failed    │               │
│  │        failed→requested(재시도3회)              │               │
│  │        expired→(30일)→deleted                   │               │
│  └─────────────────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────────────┘
```

### 기술 스택 (서드파티 0개)

| 레이어 | 기술 |
|--------|------|
| 런타임 | Blender 4.0+ 내장 Python 3.11 |
| 비동기 | threading.Thread + queue.Queue + bpy.app.timers |
| 스토리지 | sqlite3 (표준 라이브러리) |
| API 통신 | urllib.request (표준) |
| API 키 | AddonPreferences (Blender 표준 관행) |
| 스키마 검증 | dict + isinstance |
| 파일 검증 | magic bytes + struct (표준) |
| 인코딩 | UTF-8 통일, utf-8-sig 읽기 |
| 테스트 | pytest (단위) + blender --background (headless) |
