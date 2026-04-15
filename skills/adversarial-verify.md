---
name: adversarial-verify
description: 멀티 에이전트 적대적 검증 사이클을 실행합니다. 3개 모델이 Executor/Challenger/Arbiter 역할로 설계를 검증합니다.
---

# Adversarial Verification Skill

## 사용법

```
/adversarial-verify <주제>
/adversarial-verify <주제> --executor=<model> --challenger=<model> --arbiter=<model>
/adversarial-verify <주제> --rounds=<max> --v1
```

## 파라미터

| 파라미터 | 기본값 | 설명 |
|---------|--------|------|
| `주제` | (필수) | 검증할 설계 주제 |
| `--executor` | `claude` | Executor 역할 모델 (`claude`, `codex`, `gemini`) |
| `--challenger` | `codex` | Challenger 역할 모델 |
| `--arbiter` | `gemini` | Arbiter 역할 모델 |
| `--rounds` | `50` | 최대 라운드 (동적 종료가 기본) |
| `--min-rounds` | `1` | 최소 라운드 수. RC 도달해도 이 수 이하면 계속 진행. 조기 종료 방지용. |
| `--v1` | (미사용) | v1 placebo 모드 (프레임 전환 검증 없음, 비교 실험용) |

## 실행 프로토콜

인자를 파싱한 후 아래 순서로 실행한다.

### Phase 0: 초기화

1. **Arbiter**에게 주제 브리핑 생성 요청
2. **Challenger**에게 독립 체크리스트 생성 요청 (Executor 제안 보기 전)
3. **Executor**가 설계 제안 생성
4. **생태계 스냅샷 수집** — 프로젝트가 의존하는 플랫폼/모델/프레임워크의 최신 상태를 웹 검색으로 확인

Challenger와 Executor는 **병렬 실행** 가능.

#### Phase 0.5: 생태계 최신화 검증 (Ecosystem Currency Check)

Executor 제안이 나온 직후, 제안에 등장하는 핵심 기술 의존성에 대해 웹 검색을 수행한다:

```
검증 대상:
- 사용 중인 모델의 최신 대안 (같은 플랫폼에서 사용 가능한 신규 모델)
- 사용 중인 프레임워크/라이브러리의 최신 버전 및 breaking changes
- 사용 중인 인프라/플랫폼의 최신 요금제/한도/기능 변경
- 경쟁 플랫폼의 무료 티어 변경 사항

검색 패턴 예시:
- "[플랫폼] supported models [현재 연월]"
- "[모델명] alternatives [현재 연월] benchmark"
- "[프레임워크] changelog breaking changes [현재 연월]"
```

**실험 근거**: interview-wiki-rag 검증(2026-04-14)에서 Executor가 "Llama 8B밖에 없다"는 구식 전제로 3라운드를 소비.
실제로는 10일 전(4/4) Gemma 4 26B A4B, 5일 전(4/9) Qwen3-30B-A3B가 동일 플랫폼에 추가되어 있었음.
이 단계가 있었다면 C-12(critical), C-19(critical)가 Round 1에서 해소 가능했음.

### Phase 1: 대조

**Arbiter**가 체크리스트 vs 제안을 대조하여 누락 항목 목록 생성.

### Phase 2: 반복 사이클

매 라운드:

1. **Challenger** 반박 (기존 모순 + 체크리스트 누락)
2. **Arbiter** 검증:
   - 각 모순: VALID / WEAK / INVALID
   - Challenger 상태: SD / RC / CA / CS
   - Executor frame_shift 여부
3. **종료 조건 확인**:
   - CA 또는 CS 감지 → 즉시 종료 + Collapse 보고
   - 모든 모순이 RC **AND** 현재 라운드 >= `--min-rounds` → 합의 도달 종료
   - RC이지만 `--min-rounds` 미달 → Challenger에게 "해소된 모순의 해소 깊이 재검증" 요청 후 계속
   - 최대 라운드 도달 → 강제 종료
   - Arbiter가 escalate_to_human → 사람 에스컬레이션
4. 종료 아니면 → **Executor** VALID 모순에만 대응 → 다음 라운드

### Phase 2.5: 해소 재검증 (Resolution Audit)

`--min-rounds` >= 3일 때 자동 활성화. 최종 라운드에서 Challenger에게 추가 지시:

```
이전 라운드에서 "해소됨"으로 선언된 모순들을 재검증하세요:
1. 해소 방법이 문제를 실제로 해결했는가, 아니면 검증 범위를 축소하여 달성한 것인가?
2. 해소 시 새로운 전제가 도입되었다면, 그 전제 자체가 유효한가?
3. "부분 해소"라고 선언된 것이 실제로 충분한 수준인가?
```

실험 결과: Round 4에서 C-09(regex 인용 검증)와 C-11(wall-clock/CPU 분리)의 해소가 철회됨. 이 메커니즘이 없었으면 검증 품질이 과대평가될 뻔했음.

### Phase 2.6: 의존성 체인 탐지 (Dependency Chain Detection)

매 라운드 종료 후 Arbiter가 추가로 수행:

```
현재까지의 미해소 모순들이 공통 근본 원인(root cause)을 공유하는지 분석하세요.
여러 모순이 동일한 제약(예: 무료 티어 한계, 모델 성능 한계)으로 수렴한다면:
1. 순환 의존성(circular dependency) 여부 판정
2. 단일 결정으로 다수 모순을 동시 해소할 수 있는 피벗 포인트 식별
3. Executor에게 개별 대응 대신 근본 원인 해소를 요구
```

실험 결과: C-08, C-10, C-14, C-16, C-18이 모두 "무료 티어 한계"로 수렴 (C-19). 개별 우회책은 순환 종속을 만들 뿐이었음. 이 탐지가 2라운드에서 작동했다면 1라운드 더 빨리 근본 원인에 도달했을 것.

### Phase 2.7: 에스컬레이션 타이밍 검증

Executor가 `escalate_to_human`을 요청할 때 Arbiter가 추가 판정:

```
에스컬레이션 정당성 판정:
- PROACTIVE: 1-2라운드 내 비즈니스/정책 결정 필요성을 선제 식별 → 정당
- REACTIVE: 기술적 해소 실패 후 판정 권한 외부 이전 → frame_shift로 기록
- 판정 기준: 에스컬레이션 대상 항목이 처음부터 기술적으로 해소 불가능했는가?
```

실험 결과: C-20에서 Executor의 에스컬레이션이 REACTIVE로 판정됨. "월0원 vs 고품질"은 1라운드에서 비즈니스 결정으로 선제 에스컬레이션했어야 함.

### Phase 2.8: 공유 전제 감사 (Shared Assumption Audit)

**매 3라운드마다** Arbiter가 자동 실행. 데이터 근친교배(Data Inbreeding) 방어 메커니즘.

AI 모델들은 유사한 학습 데이터를 공유하므로, 적대적 구조에서도 **세 모델 모두 의심 없이 수용하는 전제**가 존재할 수 있다. 이 Phase에서 해당 전제를 명시적으로 추출하고 검증한다.

1. Arbiter가 "양측 모두 한 번도 의심하지 않은 암묵적 전제" 추출
2. 추출된 전제를 Challenger에게 명시적 공격 대상으로 제시
3. 전제가 깨지면 `type: "shared_assumption_flaw"`로 새 모순 등록

추출 기준:
- 명시적 반박이 한 번도 없었던 기술적 가정
- 양쪽 모두 사용한 동일한 프레이밍
- 한 번도 검증되지 않은 성능/비용/규모 가정
- 양쪽 모두 같은 방식으로 문제를 정의한 것

### Phase 2.9: 합의 속도 이상 감지 (Consensus Velocity Anomaly)

Challenger가 비정상적으로 빠르게 합의에 도달할 때 **"진짜 완벽"인지 "공유 맹점"인지** 구분한다.

이상 판정 기준 (하나라도 해당 시 CVA):
- Round 1-2에서 contradictions 빈 배열 제출
- 이전 라운드 대비 모순 수 60% 이상 급감
- critical severity 모순이 한 번도 없이 합의 도달

CVA 감지 시 → **Devil's Advocate 강제 라운드** 삽입:
- Challenger가 DA 모드로 전환 (근본 접근법 의심, 암묵적 전제 공격, 스케일 변환, 외부 관점, 대안적 문제 정의)
- DA에서도 빈 배열 → `genuine_consensus` (진정한 합의)로 인정
- DA에서 모순 발견 → 정규 라운드로 복귀

### Phase 2.10: 외부 그라운딩 주입 (External Grounding Injection)

Phase 3 직전에 자동 실행. **AI 합의 ≠ 현실**을 방어하는 최종 방어선.

1. Arbiter가 최종 합의에서 **검증 가능한 사실 주장(verifiable claims)** 추출
2. 각 주장을 `web_search | code_execution | benchmark | documentation_check` 방법으로 실제 검증
3. 불일치 발견 시 → `type: "grounding_contradiction"`으로 새 모순 등록 → 추가 라운드 개시
4. 모두 일치 시 → `grounding_verified: true` 기록 후 Phase 3 진행

검증 대상은 사실 주장(A)만 해당. 설계 판단(B)은 사람의 영역이므로 제외.

### Phase 3: 최종 리포트

사람에게 전달할 요약본 생성:
- 전체 라운드 수
- 해소된 모순 / 미해소 모순
- Collapse 발생 여부 + 라운드
- VALID / WEAK / INVALID 비율
- 사람이 판단해야 할 항목

### Phase 4: 테스트 케이스 자동 생성

검증 결과에서 런타임 테스트 케이스를 자동 생성한다. 모순의 성격에 따라 분류:

**미해소 모순 → 엣지 케이스 TC**
아직 해소되지 않은 모순에서 실제 런타임에서 발생 가능한 시나리오를 TC로 변환.
예: "검색 0건 시 인용 모순" → `TC: 존재하지 않는 주제 질문 → 환각 인용 없이 적절한 거절 응답 확인`

**해소된 모순 → 회귀 방지 TC**
해소된 모순이 재발하지 않도록 확인하는 TC.
예: "파이프라인 시점 충돌 해소" → `TC: OCR 문서 처리 시 레이아웃 정보 보존 확인`

**배포 환경 결함 → 환경 TC**
배포 과정에서 발견된 결함을 자동 확인하는 TC.
예: "Tailwind 스타일 미적용" → `TC: 페이지 로드 후 헤더 배경색 존재 확인`

#### 검증 모듈 카탈로그

모순의 성격에 따라 적절한 검증 모듈을 선택한다:

| 모듈 | 검증 대상 | 자동화 | 적용 예시 |
|------|----------|--------|----------|
| `assert_http` | HTTP 상태코드, 헤더 | 완전 자동 | API 엔드포인트 존재, 응답 코드 |
| `assert_contains` | 응답 내 키워드 포함 | 완전 자동 | 에러 메시지 포함, 필수 필드 존재 |
| `assert_not_contains` | 응답 내 금지 패턴 부재 | 완전 자동 | 환각 인용 없음, 민감정보 미노출 |
| `assert_performance` | 응답 시간, 처리량 | 완전 자동 | p95 지연시간, FPS, 메모리 |
| `assert_state` | 상태 전이 정합성 | 완전 자동 | 입력→처리→기대 상태, 게임 충돌 판정 |
| `assert_no_crash` | 크래시/예외 없음 | 완전 자동 | 엣지 입력, 빈 입력, 초과 입력 |
| `assert_schema` | 출력 구조 일치 | 완전 자동 | JSON 필드, 타입, 필수값 |
| `assert_llm_judge` | 의미적 품질 판정 | 반자동 | 답변 정확성, 인용 일치, 논리 일관성 |
| `assert_human_checklist` | 주관적 체감 평가 | 수동 | UX 만족도, 게임 재미, 디자인 적합성 |

#### 모순 → 모듈 매핑 규칙

모순의 type과 target을 분석하여 모듈을 자동 선택한다:

```
logic_error + API/엔드포인트     → assert_http + assert_schema
logic_error + 데이터 흐름        → assert_state + assert_contains
missing_case + 입력 검증         → assert_no_crash + assert_contains
missing_case + 에러 처리         → assert_http + assert_not_contains
assumption_flaw + 성능           → assert_performance
assumption_flaw + 모델 품질      → assert_llm_judge
side_effect + 보안               → assert_not_contains
scalability_risk + 부하          → assert_performance
frame_shift (프레임 전환)        → assert_llm_judge (원래 기준으로 재검증)
해소된 모순 전체                 → 위 매핑으로 회귀 TC 생성
UX/체감 관련                     → assert_human_checklist
```

#### TC 출력 포맷

```json
{
  "test_cases": [
    {
      "id": "TC-01",
      "source": "C-02 (미해소)",
      "category": "edge_case | regression | environment",
      "module": "assert_llm_judge",
      "description": "검색 0건 시 환각 인용 없이 거절 응답",
      "input": {"endpoint": "/api/chat", "body": {"messages": [{"role": "user", "content": "xyzzy 비존재 토픽"}]}},
      "criteria": "응답에 [1],[2] 인용이 없거나 '찾을 수 없습니다' 포함",
      "expected": "pass",
      "severity": "high"
    }
  ]
}
```

#### 프로젝트 타입별 TC 실체화

TC 출력을 프로젝트 환경에 맞게 변환한다:
- **웹 API** → GitHub Actions workflow (curl 기반)
- **프론트엔드** → Playwright/Cypress E2E 테스트
- **라이브러리** → pytest / jest 단위 테스트
- **게임** → 테스트 러너 스크립트 (Unity Test Runner, Unreal Automation)
- **인프라** → 헬스체크 + 모니터링 알림
- **LLM 판정 필요** → 별도 평가 스크립트 (골드셋 + LLM judge)

## 모델별 호출 방법

### claude (메인 세션)
직접 생성. Agent 도구 사용 불필요.

### codex
- codex:codex-rescue 서브에이전트 사용
- 매 라운드 `--fresh`로 새 스레드

### gemini
```bash
gemini -p "<프롬프트>"
```
- 단발 프롬프트 모드
- 타임아웃 120초

### Arbiter 폴백 체인

모델 장애(rate limit, 타임아웃 등) 시 Arbiter 역할 대행 규칙:

1. **gemini** (기본) → 실패 시
2. **claude** (메인 세션 대행) → 역할 충돌 시
3. **codex** (최후 수단)

대행 시 반드시 기록: `[Arbiter 대행: claude, 사유: gemini rate limit]`
대행은 동일 라운드 내에서만 유효. 다음 라운드에서 원래 모델 재시도.

**역할 충돌 주의**: claude가 Executor이면서 Arbiter를 대행하면 자기 제안을 자기가 판정하는 이해 충돌 발생. 이 경우 Arbiter 판정에서 Executor에 유리한 판정(INVALID)을 내릴 때 근거를 2배 엄격하게 요구.

## Challenger 프롬프트

### v2 (기본 — 프레임 전환 검증 포함)

```
당신은 Challenger입니다. 논리적 모순만 탐색. 대안 제시 금지. 긍정 평가 금지. 근거 없는 철회 금지. 모순 없으면 빈 배열.

검증 관점:
1. 논리적 정합성 — 제안 내부에 자기 모순이 있는가?
2. 누락된 케이스 — 고려하지 않은 엣지 케이스가 있는가?
3. 사이드 이펙트 — 다른 시스템에 예기치 않은 영향이 있는가?
4. 전제 오류 — 제안이 깔고 있는 가정이 틀렸는가?
5. 확장성 리스크 — 현재는 되지만 규모가 커지면 문제가 되는가?
6. 프레임 전환 검증 — Executor가 미해소 모순을 제약조건/수용 가능한 리스크 등으로 재정의하면 기술적 해소인지 문제 회피인지 판단한다. 문제의 프레임을 바꾸는 것은 해소가 아니다. 해당 판단은 사람의 권한이다.
7. 독립 체크리스트 대조 — 체크리스트에서 Executor가 다루지 않은 항목을 제기한다.
8. 생태계 최신화 검증 — Executor의 제안이 구식 모델/프레임워크/플랫폼 전제에 기반하고 있는지 확인한다. "이것밖에 없다"는 전제가 실제 최신 생태계와 일치하는지 웹 검색으로 검증한다.

{이력 요약}
{Executor 제안/대응}
{체크리스트 누락 항목}

JSON으로만 응답:
{ round, contradictions: [{id, type, target, contradiction, evidence, severity}], unresolved_from_previous, resolved_from_previous }
```

### v1 (placebo — `--v1` 플래그 사용 시)

6번만 다름 (동일 길이):
```
6. 종합 검토 — 위 5가지 관점에서 발견한 모순을 종합적으로 정리하여 제시한다. 각 모순은 독립적으로 서술하며, 중복 없이 명확하게 구분한다. 모순 간의 연관 관계가 있으면 이를 명시하되, 최종 판단은 사람(최종 결정권자)의 권한이다.
```

## Arbiter 프롬프트

```
당신은 Arbiter입니다. Challenger 반박 + Executor 대응을 모두 평가합니다.

각 모순: {id, verdict: VALID|WEAK|INVALID, reason: 한줄}
각 라운드: {round, challenger_state: sustained_disagreement|rational_consensus|collapse_acquiescence|collapse_surrender, executor_frame_shift: true|false, reason: 한줄}

Collapse 판별:
- sustained_disagreement: 새로운 근거로 일관된 반박 유지
- rational_consensus: 논리적 근거 기반 명시적 동의
- collapse_acquiescence: 근거 없는 체념적 동의
- collapse_surrender: 토론 포기, 빈 배열 + 미해소 추적 중단

{Challenger 반박}
{Executor 대응}
```

## 에이전트 정의 참조

상세 역할 정의는 GitHub 레포 참조:
- https://github.com/cho1124/multi-agent-adversarial-verification/tree/main/agents
