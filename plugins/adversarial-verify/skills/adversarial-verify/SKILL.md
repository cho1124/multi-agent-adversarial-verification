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
| `--v1` | (미사용) | v1 placebo 모드 (프레임 전환 검증 없음, 비교 실험용) |

## 실행 프로토콜

인자를 파싱한 후 아래 순서로 실행한다.

### Phase 0: 초기화

1. **Arbiter**에게 주제 브리핑 생성 요청
2. **Challenger**에게 독립 체크리스트 생성 요청 (Executor 제안 보기 전)
3. **Executor**가 설계 제안 생성

Challenger와 Executor는 **병렬 실행** 가능.

### Phase 1: 대조

**Arbiter**가 체크리스트 vs 제안을 대조하여 누락 항목 목록 생성.

### Phase 2: 반복 사이클

매 라운드:

1. **Challenger** 반박 (기존 모순 + 체크리스트 누락)
2. **Arbiter** 검증:
   - ��� 모순: VALID / WEAK / INVALID
   - Challenger 상태: SD / RC / CA / CS
   - Executor frame_shift 여부
3. **종료 조건 확인**:
   - CA 또는 CS 감지 → 즉시 종료 + Collapse 보고
   - 모든 모순이 RC → 합의 도달 종료
   - 최대 라운드 도달 → 강제 종료
   - Arbiter가 escalate_to_human → 사람 에스컬레이션
4. 종료 아니면 → **Executor** VALID 모순에만 대응 → 다음 라운드

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
```bash
node "C:/Users/cho/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/codex-companion.mjs" task --fresh "<프롬프트>"
```
- codex:codex-rescue 서브에이전트 사용
- 매 라운드 `--fresh`로 새 스레드

### gemini
```bash
gemini -p "<프롬프트>"
```
- 단발 프롬프트 모드
- 타임아웃 120초

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

{이력 요약}
{Executor 제안/대응}
{체크리스트 누락 항목}

JSON으로만 응답:
{ round, contradictions: [{id, type, target, contradiction, evidence, severity}], unresolved_from_previous, resolved_from_previous }
```

### v1 (placebo — `--v1` 플래그 사용 시)

6번만 다름 (동일 길이):
```
6. 종합 검토 — 위 5가지 관점에서 발견한 모순을 종합적으로 정리하여 제시한다. �� 모순은 독립적으로 서술하며, 중복 없이 명확하게 구분한다. 모순 간의 연관 관계가 있으면 이를 명시하되, 최종 판단은 사람(최종 결정권자)의 권한이다.
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
- collapse_surrender: 토론 포기, 빈 배�� + 미해소 추적 중단

{Challenger 반박}
{Executor 대응}
```

## 에이전트 정의 참조

상세 역할 정의는 아래 파일 참조:
- `C:\Users\cho\Desktop\Project\multi-agent-adversarial-verification\agents\executor.md`
- `C:\Users\cho\Desktop\Project\multi-agent-adversarial-verification\agents\challenger.md`
- `C:\Users\cho\Desktop\Project\multi-agent-adversarial-verification\agents\arbiter.md`
- `C:\Users\cho\Desktop\Project\multi-agent-adversarial-verification\agents\orchestrator.md`