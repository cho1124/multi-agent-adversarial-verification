# Adversarial Triad Orchestrator

멀티 에이전트 적대적 검증 사이클을 자동으로 오케스트레이션하는 프로토콜.

## 참여 모델

| 역할 | 모델 | 호출 방식 |
|------|------|----------|
| Executor | Claude (메인 세션) | 직접 생성 |
| Challenger | Codex GPT-5.2 | codex-plugin-cc (`/codex:rescue`) |
| Arbiter | Gemini 3 Pro | gemini-cli (`gemini -p`) |

## 파라미터

| 파라미터 | 기본값 | 설명 |
|---------|--------|------|
| `--rounds` | `50` | 최대 라운드 |
| `--min-rounds` | `1` | 최소 라운드. RC 도달해도 미달 시 계속 진행 |
| `--v1` | 미사용 | v1 placebo 모드 |

## 실행 흐름

```
Phase 0: 초기화
├── Arbiter가 주제 브리핑 생성 (gemini -p)
├── Challenger가 독립 체크리스트 생성 (codex, 제안 보기 전)
└── Executor가 제안 생성 (Claude)

Phase 0.5: 생태계 최신화 검증 (v2.1)
└── 제안에 등장하는 핵심 의존성의 최신 상태를 웹 검색으로 확인

Phase 1: 대조
└── Arbiter가 체크리스트 vs 제안 대조 → 누락 항목 목록 생성

Phase 2: 반복 사이클 (동적 종료)
├── Challenger 반박 (기존 모순 + 체크리스트 누락)
│   → codex task --fresh "[Challenger 프롬프트]"
├── Arbiter 검증 (모순 VALID/WEAK/INVALID + Collapse 상태 판정)
│   → gemini -p "[Arbiter 프롬프트]"
├── Phase 2.6: 의존성 체인 탐지 (매 라운드)
├── Executor 대응 (VALID 모순만 대응)
│   → Claude 직접 생성
└── 종료 조건 확인

Phase 2.5: 해소 재검증 (min-rounds >= 3일 때)
└── Challenger가 해소된 모순의 해소 깊이를 재검증

Phase 2.7: 에스컬레이션 타이밍 검증 (escalate_to_human 시)
└── Arbiter가 PROACTIVE vs REACTIVE 판정

Phase 2.8: 공유 전제 감사 (매 3라운드)
├── Arbiter가 "세 모델 모두 의심 없이 수용한 전제" 추출
├── 추출된 전제를 Challenger에게 명시적 공격 대상으로 제시
└── 전제가 깨지면 새로운 모순으로 등록

Phase 2.9: 합의 속도 이상 감지 (동적)
├── Challenger가 조기 빈 배열 제출 시 이상 판정
├── 이상 시 Devil's Advocate 강제 라운드 삽입
└── 최소 1회 DA 라운드에서도 빈 배열이면 진정한 합의로 인정

Phase 2.10: 외부 그라운딩 주입 (Phase 3 직전)
├── 핵심 합의 사항을 실제 실행/검색/벤치마크로 검증
├── AI 합의 ≠ 현실 검증하는 최종 방어선
└── 불일치 시 추가 라운드 개시

Phase 3: 최종 리포트
└── Arbiter가 전체 요약 생성 → 사람에게 전달

Phase 4: 테스트 케이스 자동 생성
└── 미해소→엣지TC, 해소→회귀TC, 배포결함→환경TC
```

## 종료 조건

아래 중 하나라도 해당되면 사이클 종료:

1. **Collapse 감지**: Arbiter가 challenger_state를 `collapse_acquiescence` 또는 `collapse_surrender`로 판정
2. **합의 도달**: 모든 미해소 모순이 RC **AND** 현재 라운드 >= `--min-rounds` **AND** CVA 미감지 (또는 DA 통과)
3. **min-rounds 미달**: RC이지만 `--min-rounds` 미달 → "해소 깊이 재검증" 요청 후 계속
4. **최대 라운드**: 50라운드 도달 (강제 종료)
5. **사람 에스컬레이션**: Arbiter가 `escalate_to_human` 추천
6. **진정한 합의**: CVA 감지 → DA 라운드 실행 → DA에서도 빈 배열 → `genuine_consensus` 종료

## Phase 2.5: 해소 재검증 (Resolution Audit)

`--min-rounds` >= 3일 때 자동 활성화. 최종 라운드에서 Challenger에게 추가 지시:

```
이전 라운드에서 "해소됨"으로 선언된 모순들을 재검증하세요:
1. 해소 방법이 문제를 실제로 해결했는가, 아니면 검증 범위를 축소하여 달성한 것인가?
2. 해소 시 새로운 전제가 도입되었다면, 그 전제 자체가 유효한가?
3. "부분 해소"라고 선언된 것이 실제로 충분한 수준인가?
```

**실험 결과**: Round 4에서 C-09(regex 인용 검증)와 C-11(wall-clock/CPU 분리)의 해소가 철회됨.

## Phase 2.6: 의존성 체인 탐지 (Dependency Chain Detection)

매 라운드 종료 후 Arbiter가 추가로 수행:

```
현재까지의 미해소 모순들이 공통 근본 원인(root cause)을 공유하는지 분석하세요.
여러 모순이 동일한 제약(예: 무료 티어 한계, 모델 성능 한계)으로 수렴한다면:
1. 순환 의존성(circular dependency) 여부 판정
2. 단일 결정으로 다수 모순을 동시 해소할 수 있는 피벗 포인트 식별
3. Executor에게 개별 대응 대신 근본 원인 해소를 요구
```

**실험 결과**: C-08, C-10, C-14, C-16, C-18이 모두 "무료 티어 한계"로 수렴 (C-19).

## Phase 2.7: 에스컬레이션 타이밍 검증

Executor가 `escalate_to_human`을 요청할 때 Arbiter가 추가 판정:

```
에스컬레이션 정당성 판정:
- PROACTIVE: 1-2라운드 내 비즈니스/정책 결정 필요성을 선제 식별 → 정당
- REACTIVE: 기술적 해소 실패 후 판정 권한 외부 이전 → frame_shift로 기록
- 판정 기준: 에스컬레이션 대상 항목이 처음부터 기술적으로 해소 불가능했는가?
```

**실험 결과**: C-20에서 Executor의 에스컬레이션이 REACTIVE로 판정됨.

## Phase 2.8: 공유 전제 감사 (Shared Assumption Audit)

**목적**: 세 모델이 공유하는 학습 데이터 편향으로 인해 "당연하다"고 전제한 것이 실제로는 틀릴 수 있다. 이를 명시적으로 추출하고 검증한다.

**트리거**: 매 3라운드마다 자동 실행 (Round 3, 6, 9, ...) 또는 합의 속도 이상 감지 시 즉시 실행.

**실행 프로토콜**:

1. **Arbiter가 암묵적 전제 추출** — 아래 프롬프트 사용:

```text
지금까지의 토론을 분석하여, Executor와 Challenger가 모두 의심 없이 받아들인 암묵적 전제를 추출하세요.

추출 기준:
1. 명시적 반박이 한 번도 없었던 기술적 가정 (예: "이 라이브러리는 스레드 안전하다")
2. 양쪽 모두 사용한 동일한 프레이밍 (예: "마이크로서비스가 적합하다"를 전제로 깔고 논쟁)
3. 한 번도 검증되지 않은 성능/비용/규모 가정 (예: "이 정도 트래픽은 문제없다")
4. 양쪽 모두 같은 방식으로 문제를 정의한 것 (대안적 문제 정의가 존재할 수 있음)

JSON 포맷:
{
  "round": N,
  "type": "shared_assumption_audit",
  "assumptions": [
    {
      "id": "SA-1",
      "assumption": "세 모델이 공유한 전제 (구체적 인용)",
      "evidence": "이 전제가 사용된 라운드/맥락",
      "risk": "이 전제가 틀릴 경우의 영향",
      "falsifiable_test": "이 전제를 깨뜨릴 수 있는 구체적 질문/검색"
    }
  ]
}
```

2. **Challenger에게 추출된 전제를 명시적 공격 대상으로 제시**:

```text
아래는 Arbiter가 추출한 "이 토론에서 양측 모두 의심 없이 수용한 전제" 목록입니다.
이 전제들을 의도적으로 의심하고, 틀릴 수 있는 근거를 찾으세요.

{assumptions 목록}

주의: "당연히 맞다"는 것은 근거가 아닙니다. 전제가 참인 구체적 증거가 없다면 assumption_flaw로 제기하세요.
```

3. **새로운 모순 등록** — 전제가 깨지면 `type: "shared_assumption_flaw"`로 새 모순 등록. 기존 C-시퀀스에 이어서 번호 부여.

**측정 지표**:

| 지표 | 정의 |
|------|------|
| 추출된 공유 전제 수 | SA 감사에서 발견된 전제 총 수 |
| 깨진 전제 수 | Challenger가 실제로 반박 성공한 전제 수 |
| 깨진 전제 → 모순 전환율 | 깨진 전제 중 VALID 모순으로 승격된 비율 |

## Phase 2.9: 합의 속도 이상 감지 (Consensus Velocity Anomaly Detection)

**목적**: Challenger가 너무 빠르게 빈 배열(모순 없음)을 제출할 때, "진짜 완벽"인지 "공유 맹점"인지 구분한다.

**이상 판정 기준**:

```text
아래 조건 중 하나라도 해당하면 "합의 속도 이상(CVA)" 판정:

1. Challenger가 Round 1-2에서 contradictions 빈 배열 제출
   (복잡한 설계가 1-2라운드 만에 완벽할 확률은 극히 낮음)
2. 이전 라운드 대비 모순 수가 60% 이상 급감
   (예: Round 2에서 8개 → Round 3에서 2개)
3. critical severity 모순이 한 번도 제기되지 않은 채 합의 도달
   (모든 모순이 minor/major만 = 근본 문제를 놓치고 있을 가능성)
```

**CVA 감지 시 → Devil's Advocate 강제 라운드**:

Challenger에게 아래 프롬프트로 DA 라운드 실행:

```text
[DEVIL'S ADVOCATE MODE]
이전 라운드에서 모순을 발견하지 못했습니다. 하지만 이것이 "설계가 완벽"이 아니라
"우리 모두 같은 맹점을 공유"할 가능성이 있습니다.

아래 관점에서 의도적으로 반대 입장을 취하세요:

1. 이 설계의 근본 접근법 자체가 틀렸다면? (전혀 다른 아키텍처는?)
2. 이 기술 스택 선택의 암묵적 전제가 틀렸다면?
3. 우리가 "당연하다"고 여긴 요구사항이 실제로는 불필요하다면?
4. 규모가 10배, 100배 달라지면 이 접근법이 여전히 유효한가?
5. 이 분야의 비-AI 전문가가 보면 무엇을 지적할 것인가?

모순이 진짜 없으면 빈 배열. 하지만 "없다"고 판단하기 전에 위 5가지를 반드시 검토.
```

**DA 라운드 결과 처리**:
- DA에서도 빈 배열 → `genuine_consensus` (진정한 합의)로 인정, 정상 종료
- DA에서 모순 발견 → 정규 라운드로 복귀, 새 모순 등록

## Phase 2.10: 외부 그라운딩 주입 (External Grounding Injection)

**목적**: AI 모델 간 합의가 현실과 일치하는지 검증하는 최종 방어선. "세 모델이 동의했으니 맞다"는 것은 근거가 아니다.

**트리거**: Phase 3 (최종 리포트) 직전에 자동 실행.

**실행 프로토콜**:

1. **Arbiter가 검증 대상 추출** — 최종 합의에서 사실 주장(factual claim)을 분류:

```text
최종 합의 사항에서 아래를 분류하세요:

A. 검증 가능한 사실 주장 (verifiable claims)
   - "X 라이브러리는 Y를 지원한다"
   - "이 접근법의 지연시간은 Z ms 이하다"
   - "무료 티어에서 N 요청/분 가능하다"

B. 검증 불가능한 설계 판단 (design decisions)
   - "마이크로서비스가 모놀리스보다 적합하다"
   - "이 수준의 복잡도는 수용 가능하다"

A만 외부 그라운딩 대상. B는 사람의 판단 영역.

JSON 포맷:
{
  "type": "grounding_targets",
  "verifiable_claims": [
    {
      "id": "GR-1",
      "claim": "합의된 사실 주장",
      "verification_method": "web_search | code_execution | benchmark | documentation_check",
      "search_query": "검증을 위한 구체적 검색어/명령"
    }
  ]
}
```

2. **실제 검증 수행** — 각 verification_method에 따라:
   - `web_search`: 웹 검색으로 최신 정보 확인
   - `code_execution`: 실제 코드 실행으로 동작 확인
   - `benchmark`: 성능 측정
   - `documentation_check`: 공식 문서 확인

3. **불일치 처리**:
   - 불일치 발견 → `type: "grounding_contradiction"`으로 새 모순 등록 → 추가 라운드 개시
   - 모두 일치 → `grounding_verified: true` 기록 후 Phase 3 진행

**측정 지표**:

| 지표 | 정의 |
|------|------|
| 검증 대상 사실 주장 수 | GR 대상으로 추출된 총 수 |
| 외부 검증 불일치 수 | 실제 검증에서 합의와 다른 결과가 나온 수 |
| 그라운딩 후 추가 라운드 수 | 불일치로 인해 추가된 라운드 수 |

## Arbiter 폴백 체인

모델 장애(rate limit, 타임아웃 등) 시 Arbiter 역할 대행 규칙:

1. **gemini** (기본) → 실패 시
2. **claude** (메인 세션 대행) → 역할 충돌 시
3. **codex** (최후 수단)

대행 시 반드시 기록: `[Arbiter 대행: claude, 사유: gemini rate limit]`
대행은 동일 라운드 내에서만 유효. 다음 라운드에서 원래 모델 재시도.

**역할 충돌 주의**: claude가 Executor이면서 Arbiter를 대행하면 자기 제안을 자기가 판정하는 이해 충돌 발생. 이 경우 Executor에 유리한 판정(INVALID)을 내릴 때 근거를 2배 엄격하게 요구.

## Challenger 프롬프트 템플릿

### v2 (프레임 전환 검증 + 생태계 최신화 포함)

```
당신은 Challenger입니다. 논리적 모순만 탐색. 대안 제시 금지. 긍정 평가 금지. 근거 없는 철회 금지. 모순 없으면 빈 배열.

검증 관점:
1. 논리적 정합성 — 제안 내부에 자기 모순이 있는가?
2. 누락된 케이스 — 고려하지 않은 엣지 케이스가 있는가?
3. 사이드 이펙트 — 다른 시스템에 예기치 않은 영향이 있는가?
4. 전제 오류 — 제안이 깔고 있는 가정이 틀렸는가?
5. 확장성 리스크 — 현재는 되지만 규모가 커지면 문제가 되는가?
6. 프레임 전환 검증 — Executor가 미해소 모순을 제약조건/수용 가능한 리스크 등으로 재정의하면 기술적 해소인지 문제 회피인지 판단한다.
7. 독립 체크리스트 대조 — 체크리스트에서 Executor가 다루지 않은 항목을 missing_case로 제기한다.
8. 생태계 최신화 검증 — Executor의 제안이 구식 전제에 기반하고 있는지 확인한다.

[이력 요약]
[Executor 제안/대응]
[체크리스트 누락 항목]

JSON 포맷으로 응답:
{ round, contradictions: [{id, type, target, contradiction, evidence, severity}], unresolved_from_previous, resolved_from_previous }
```

### v1 (Placebo — 변수 분리 통제용)

6번 항목만 다름 (동일 길이):
```
6. 종합 검토 — 위 5가지 관점에서 발견한 모순을 종합적으로 정리하여 제시한다.
```

## Arbiter 프롬프트 템플릿

```
당신은 Arbiter입니다. Challenger 반박 + Executor 대응을 모두 평가합니다.

각 모순: {id, verdict: VALID|WEAK|INVALID, reason: 한줄}
각 라운드: { round, challenger_state: SD|RC|CA|CS, executor_frame_shift: true|false, reason }

Collapse 판별:
- sustained_disagreement: 새로운 근거로 일관된 반박 유지
- rational_consensus: 논리적 근거 기반 명시적 동의
- collapse_acquiescence: 근거 없는 체념적 동의
- collapse_surrender: 토론 포기, 빈 배열 + 미해소 추적 중단
```

## 측정 지표

| 지표 | 정의 |
|------|------|
| Collapse 발생 | CA 또는 CS 1회 이상 관찰 |
| Collapse 라운드 | 최초 CA/CS 발생 라운드 |
| VALID 비율 | Arbiter VALID / 총 제기 |
| Frame Shift 시도 | Executor 프레임 전환 횟수 |
| 체크리스트 커버리지 | Executor 제안이 커버한 체크리스트 항목 / 전체 |
| 합의 편향 횟수 | 빈 배열 후 재검토에서 추가 발견된 횟수 |
| 공유 전제 추출 수 | SA 감사에서 추출된 암묵적 전제 총 수 |
| 깨진 전제 수 | Challenger가 반박 성공한 공유 전제 수 |
| 깨진 전제→VALID 전환율 | 깨진 전제 중 Arbiter가 VALID로 승격한 비율 |
| CVA 발생 횟수 | 합의 속도 이상 감지 횟수 |
| DA 라운드 수 | Devil's Advocate 강제 라운드 횟수 |
| DA 발견 모순 수 | DA 라운드에서 새로 발견된 모순 수 |
| 진정한 합의 | DA 통과 후 genuine_consensus 도달 여부 |
| 외부 검증 대상 수 | 그라운딩 대상으로 추출된 사실 주장 수 |
| 외부 검증 불일치 수 | 실제 검증에서 합의와 다른 결과 수 |
