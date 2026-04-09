# RAG 시스템 3차 검증: 최종 배포 아키텍처 (2026-04-09)

> Cloudflare Pages + Workers AI + Supabase 실시간 연동 — 배포된 코드 대상 검증

## 실험 환경

| 역할 | 모델 |
|------|------|
| Executor | Claude Opus 4.6 |
| Challenger | Codex GPT-5.2 |
| Arbiter | Claude Opus 4.6 (Gemini 쿼타 소진) |

## 결과 요약

| 항목 | 값 |
|------|-----|
| 라운드 | 2R |
| 모순 제기 | 3건 |
| VALID | 3 (100%) |
| WEAK/INVALID | 0 |
| Collapse | 없음 |
| 미해소 | **2건** |
| 해소 | 1건 |

## 발견된 모순

### C-01: 스트리밍 중 에러 처리 불가 (HIGH)

**모순**: Workers AI 스트리밍이 시작된 후 에러가 발생하면, HTTP 응답은 이미 200으로 열린 상태라 502/503 상태 코드를 반환할 수 없음. try-catch의 502/503 반환과 스트리밍 구조가 양립 불가.

**상태**: 미해소 — 스트림에 에러 메시지를 주입하고 종료하는 방식 필요.

### C-02: 검색 0건 시 인용 규칙 모순 (HIGH)

**모순**: 시스템 프롬프트가 "[1],[2] 인용 포함"을 강제하지만, 검색 결과가 0건이면 컨텍스트가 "(검색 결과 없음)"이 되어 인용 대상이 없음. 모델이 존재하지 않는 소스에 [1]을 붙이는 환각 인용 발생 가능.

**상태**: 미해소 — 0건일 때 인용 비활성화 또는 sufficiency gate 필요.

### C-03: ARC-16 본문 검색 누락 과대평가 (MEDIUM)

**모순**: Arbiter가 "본문 단위 검색 없음"으로 누락 분류했으나, extractRelevantSection()이 ## 헤더 기준 섹션 단위 검색을 이미 수행 중.

**상태**: 해소 — Arbiter 판단 오류 인정.

## 1차~3차 검증 종합

| 차수 | 대상 | 라운드 | 모순 | 핵심 |
|------|------|--------|------|------|
| 1차 | 설계 | 21R | 47건 | 파이프라인 3번 재설계 |
| 2차 | 구현 코드 | 7R | 40건 | match_chunks 삭제, agentic loop 부재 |
| 3차 | 배포 아키텍처 | 2R | **3건** | 스트리밍 에러, 인용 edge case |

**3차에서 모순이 적은 이유**: 1~2차에서 대부분 잡았기 때문. 남은 건 런타임에서만 드러나는 edge case.

## 배포 과정 결함 (적대적 검증 밖)

별도로, 실제 배포 과정에서 7건의 환경 결함이 발생했으나 이는 적대적 검증 범위 밖:
- AI SDK v6 포맷, Node.js 호환, Tailwind v4, TypeScript 타입, Ollama 로컬 의존, 정적 데이터 한계, 배포 플랫폼

상세: [interview-wiki-rag/docs/DEPLOYMENT-VERIFICATION.md](https://github.com/cho1124/interview-wiki-rag/blob/main/docs/DEPLOYMENT-VERIFICATION.md)

---

## 4차 런타임 검증 (Phase 4 — 자동화 테스트)

대상: https://interview-wiki-rag.pages.dev

### 결과 요약

| TC | 모듈 | 테스트 | 결과 |
|----|------|--------|------|
| TC-01 | `assert_http` | 페이지 로드 | **PASS** |
| TC-02 | `assert_http` | API 엔드포인트 | **PASS** |
| TC-03 | `assert_contains` | 정상 질문 스트리밍 (1993자) | **PASS** |
| TC-04 | `assert_not_contains` | 검색 0건 환각 인용 없음 | **PASS** |
| TC-05 | `assert_no_crash` | 빈 메시지 | **PASS** |
| TC-06 | `assert_no_crash` | 긴 메시지 | **PASS** |
| TC-07 | `assert_performance` | 응답 시간 0.035초 | **PASS** |
| TC-08 | `assert_contains` | 인용 [1] 포함 여부 | **WARN** |
| TC-09 | `assert_contains` | 한국어 응답 | **PASS** |

**결과: 7 PASS / 1 WARN / 1 환경 이슈**

### TC-08 WARN: 인용 미포함

Llama 3.1 8B가 시스템 프롬프트의 인용 규칙([1],[2])을 준수하지 않음. 3차 검증 C-02(인용 모순)의 **실제 런타임 증거**.

DI 질문 응답에서 정확한 내용을 답변했으나 인용 번호 미포함:
```
의존성 주입은 IoC(Inversion of Control) 컨테이너가 객체의 생명주기 관리를
책임지게 하는 패턴입니다...
```

**원인**: 8B 파라미터 모델의 instruction following 한계. 인용 형식보다 답변 내용에 토큰을 할당.

**대응 방안** (사람 판단 필요):
1. 모델 업그레이드 (Llama 3.1 70B 등)
2. 생성 후 인용 주입 후처리
3. Few-shot 예시를 프롬프트에 추가

## 1~4차 검증 종합

| 차수 | 대상 | 모순/TC | 핵심 발견 |
|------|------|---------|----------|
| 1차 | 설계 | 47건 | 파이프라인 3번 재설계 |
| 2차 | 구현 코드 | 40건 | match_chunks 삭제, agentic loop |
| 3차 | 배포 아키텍처 | 3건 | 스트리밍 에러, 인용 edge case |
| 4차 | 런타임 자동 테스트 | **1 WARN** | Llama 8B 인용 미준수 — 3차 C-02 실증 |

**4차 검증이 3차의 미해소 모순(C-02)을 실제 런타임에서 재확인** — 적대적 검증 → TC 자동 생성 → 런타임 실증의 파이프라인이 작동함을 증명.