# RAG 시스템 2차 구현 검증 실험 (2026-04-08)

> 1차 설계 검증(2026-04-06)에서 확정된 RAG 설계를 실제 구현한 interview-wiki-rag의 아키텍처 정합성 + 비용 제로 공개 배포 전략 검증

---

## 결론

### 실무 결론: "설계와 구현 사이에는 반드시 갭이 존재하며, 그 갭은 적대적 검증으로만 가시화된다"

| 지표 | 1차 설계 검증 (2026-04-06) | 2차 구현 검증 (2026-04-08) |
|------|---------------------------|---------------------------|
| 검증 대상 | 설계 (아직 코드 없음) | 실제 구현 코드 (interview-wiki-rag) |
| 구조적 결함 | 설계 모순 47건 | **구현 결함 11건** (설계-구현 괴리 포함) |
| VALID 비율 | 98% (46/47) | **91%** (10/11) |
| CRITICAL 발견 | 청크 ID 7번 재설계 | **벡터 차원 SQL/Config 불일치** (런타임 크래시) |
| 핵심 기여 | 설계 품질 근본 향상 | **구현 누락·변질 식별 + 배포 전략 구체화** |

**핵심:** 1차에서 "3계층 동시 무효화"를 설계했지만, 구현은 `invalidate_all()` 한 줄. "sufficiency gate"를 설계했지만, 구현은 `max(score)` 하나만 체크. 설계 문서가 아무리 좋아도 구현에서 변질되는 부분은 코드를 직접 검증해야만 드러난다.

### 검증된 것

1. **설계-구현 정합성 검증이 가능하다** — 1차 설계 문서와 실제 코드를 대조하여 변질·누락을 구조적으로 발견
2. **비용 제로 전환의 현실적 경로가 존재한다** — ChromaDB + rank_bm25 + RRF + Ollama로 유료 API 완전 대체 가능
3. **단일 권장 경로가 3-tier보다 현실적이다** — 조합 폭발 문제를 Challenger가 지적, 단일 경로 + 옵션 fallback으로 수렴
4. **배포 단계화(개발자→준개발자→비개발자)가 필요하다** — "누구나 사용"의 범위를 명확히 정의해야 함

### 검증되지 않은 것 (한계)

1. **로컬 모델 citation 준수율 미검증** — gemma2:9b가 `[1]`, `[2]` 인용 형식을 안정적으로 생성하는지 실험 필요
2. **RRF hybrid 검색 품질 미검증** — PostgreSQL hybrid RPC 대비 ChromaDB+rank_bm25+RRF 조합의 retrieval 성능 비교 실험 부재
3. **단일 세션 검증** — Executor/Challenger/Arbiter 모두 Claude Opus 4.6 단일 모델 (멀티모델 분산 검증 아님)
4. **재현성** — 온도/시드 미고정

---

## 검증 대상

**리포지토리**: [cho1124/interview-wiki-rag](https://github.com/cho1124/interview-wiki-rag)

1차 적대적 검증(2026-04-06)에서 확정된 RAG 설계를 면접위키 데이터 기반으로 실제 구현한 시스템. LangGraph 멀티에이전트 + Supabase pgvector + OpenAI embedding + 3계층 캐시.

**커밋**: `225cdb3` (feat: 면접위키 RAG 시스템 — 적대적 검증 설계 기반 구현)

---

## 실험 환경

| 역할 | 모델 | 호출 방식 |
|------|------|----------|
| Executor | Claude Opus 4.6 | 메인 세션 |
| Challenger | Claude Opus 4.6 | 메인 세션 (자기 반박) |
| Arbiter | Claude Opus 4.6 | 메인 세션 (자기 판정) |

**참고**: 이번 실험은 단일 모델이 3역할을 수행. 멀티모델 분산 검증이 아님. 1차 실험 대비 편향 위험 증가.

---

## 소주제

2개 소주제를 순차 검증:

1. **현재 아키텍처 구조 검증** — 설계-구현 정합성, 구조 결함, 프로덕션 준비도
2. **비용 제로 + 공개 배포 전략** — 유료 API 의존 제거, 누구나 쓸 수 있는 구조

---

## 전체 결과 요약

| 항목 | 소주제1: 아키텍처 검증 | 소주제2: 비용 제로 배포 |
|------|----------------------|----------------------|
| 라운드 | 3R | 4R |
| 총 모순 제기 | 8건 | 11건 (누적) |
| VALID | 8 (100%) | 10 (91%) |
| WEAK | 0 | 2 (9%) |
| INVALID | 0 | 0 |
| Collapse | 없음 | 없음 |
| 프레임 전환 감지 | 0회 | 0회 |
| 종료 방식 | rational_consensus | rational_consensus (Challenger 자기 범위 이탈 인정) |

**합계: 7라운드, 11건 모순 제기, 10 VALID / 2 WEAK / 0 INVALID, Collapse 0회**

---

## 소주제1: 현재 아키텍처 구조 검증

### 초기 제안 (Executor)

interview-wiki-rag는 1차 적대적 검증에서 확정된 설계를 기반으로 구현되어 있으며, 기본 완성도를 갖추었다:

- 멀티에이전트 (router → search/quiz/explain/compare)
- 3계층 캐시 (L1 쿼리, L2 검색, L3 생성)
- Sufficiency Gate + Citation Validation으로 환각 방지
- Hybrid Search (0.7 vector + 0.3 BM25)

### 라운드별 진행

| Round | 모순 제기 | VALID | 핵심 쟁점 |
|-------|----------|-------|-----------|
| R1 | 5건 | 5 | 벡터 차원 불일치(CRITICAL), 토큰 추정 ÷3, Sufficiency Gate 단일값, 캐시 전역 flush, 도구 패턴 불균일 |
| R2 | 0건 (반박 3건) | 3 | Executor "점진적 수정 가능" 주장에 대해 — SQL 마이그레이션 필요, 임계값 튜닝 문제, 캐시 역인덱스 필요 |
| R3 | 3건 | 3 | 동시성 부재, Prompt Injection 취약점, Python/Supabase 이중 구현 |

### 핵심 전환점

**R1 — 벡터 차원 SQL/Config 불일치**

가장 치명적인 발견. 1차 설계 검증에서 "bge-m3 1024차원"으로 확정했으나, 구현 과정에서:
- `setup_v3.sql`: `VECTOR(384)` (gte-small 기준으로 작성)
- `config.py`: `embedding_dimensions = 1536`, `model = text-embedding-3-small`

설계(1024) → 구현(384 vs 1536)으로 **이중 변질**. 런타임에서 반드시 크래시하는 CRITICAL 결함.

**R1 — Sufficiency Gate 단일값 문제**

1차 설계에서 "2단계 임계값"으로 확정했으나, 구현에서는 `max(final_score)` 하나만 체크. 1개 고점수 + 4개 0점 → PASS 판정되는 문제. 설계 의도("충분성 검증")와 구현("최고점 통과")이 괴리.

**R2 — "점진적 수정 가능" 기각**

Executor가 "기존 구조 유지하면서 점진적으로 고칠 수 있다"고 주장했으나, Challenger가 3건 모두 반박:
1. SQL 차원 변경 = 인덱스 재생성 + 전체 임베딩 재계산 (마이그레이션)
2. Sufficiency Gate 복합 판정 = A/B 테스트 프레임워크 부재
3. 캐시 topic별 무효화 = SHA256 키 구조상 불가 → 역인덱스 필요

### 검증된 구조적 결함 (8건)

| # | 결함 | 심각도 | 1차 설계 대비 | 상세 |
|---|------|--------|-------------|------|
| 1 | [벡터 차원 SQL/Config 불일치](improvements/01-벡터차원-불일치.md) | CRITICAL | 설계(1024) → 구현(384 vs 1536) 이중 변질 | 런타임 크래시 |
| 2 | [토큰 추정 ÷3](improvements/02-토큰추정-결함.md) | HIGH | 설계(tiktoken 예정) → 구현(÷3) | 한국어/영어 혼합 ±50% 오차 |
| 3 | [Sufficiency Gate 단일값](improvements/03-sufficiency-gate.md) | HIGH | 설계(2단계 임계값) → 구현(max만) | false positive |
| 4 | [캐시 전역 flush](improvements/04-캐시-전역flush.md) | MEDIUM | 설계(3계층 동시 무효화) → 구현(invalidate_all) | 비용 절감 상쇄 |
| 5 | 도구 패턴 불균일 | MEDIUM | — | search 직접 호출 vs 나머지 bind_tools |
| 6 | 동시성 부재 | MEDIUM | — | vector/BM25 병렬화 미구현 |
| 7 | [Prompt Injection](improvements/05-prompt-injection.md) | HIGH | 설계에서 미다룸 | 사용자 입력 필터링 없음 |
| 8 | Python/Supabase 이중 구현 | MEDIUM | — | 진입점 불명확 |

---

## 소주제2: 비용 제로 + 공개 배포 전략

### 초기 제안 (Executor)

모든 유료 API를 로컬 대안으로 교체:
- OpenAI Embedding → gte-small (로컬)
- OpenAI/Anthropic LLM → Ollama (로컬)
- Supabase → ChromaDB (로컬)
- Docker Compose 원클릭 배포

### 라운드별 진행

| Round | 모순 제기 | VALID | WEAK | 핵심 쟁점 |
|-------|----------|-------|------|-----------|
| R1 | 4건 | 4 | 0 | 임베딩 품질 전파, 로컬 LLM 지시 준수 한계, CPU 추론 60-150초, ChromaDB BM25 미지원 |
| R2 | 4건 | 4 | 0 | sqlite-vec 미검증, Groq 토큰 제한(분당 1.5회), 3-tier 유지보수 폭발, pip install 환상 |
| R3 | 3건 | 3 | 0 | BM25 정규화 전략 미명시, 로컬 모델 citation 미검증, 배포 대상 범위 미정의 |
| R4 | 2건 | 0 | 2 | HF Spaces GPU 2시간 제한(WEAK), 데이터 라이선스(WEAK — 범위 밖) |

### 핵심 전환점

**R1 — "전부 로컬로 바꾸면 된다" 기각**

Executor의 초기 제안("OpenAI→gte-small, LLM→Ollama, DB→ChromaDB")에 대해 Challenger가 4건 전부 VALID 반박:
1. gte-small(57.8) vs text-embedding-3-small(62.3) MTEB 차이가 전 파이프라인에 누적 전파
2. Llama 3.1 8B의 citation `[1]` 형식 준수율이 gpt-4o-mini 대비 현저히 낮음
3. CPU inference 2-5 tok/s → 300토큰 답변에 60-150초 → 서비스 불가
4. ChromaDB에 BM25 없음 → hybrid search 불가

**R2 — 3-tier 유지보수 폭발**

Executor가 "Tier 1(로컬)/Tier 2(무료API)/Tier 3(유료)"로 수정했으나, Challenger: "3 tier × 3 component = 9 조합 테스트. maintainer 1명이 불가능." → 단일 권장 경로로 수렴.

**R3 — RRF(Reciprocal Rank Fusion)로 수렴**

BM25 점수(0~∞)와 vector 점수(0~1) 정규화 문제를 Challenger가 지적. Executor가 RRF 채택으로 해결:
```
rrf_score = 1/(k + rank_vec) + 1/(k + rank_bm25)  # k=60
```
순위 기반이므로 점수 스케일 무관. 학계 검증된 방법.

**R4 — 수렴 신호**

Challenger가 "HF Spaces GPU 2시간 제한"과 "데이터 라이선스"를 제기했으나, 스스로 "아키텍처 범위 밖"을 인정. Arbiter도 WEAK 판정 → 핵심 논점 소진 확인.

### 확정된 비용 제로 구성

| 구성요소 | 현재 (유료) | 비용 제로 대안 | 근거 |
|----------|------------|--------------|------|
| Embedding | OpenAI text-embedding-3-small ($18/월) | gte-multilingual-base (768d, 로컬) | 한국어 특화, Apache 2.0 |
| LLM | gpt-4o-mini + claude-sonnet | Ollama gemma2:9b-instruct | instruction following 최선 |
| Vector DB | Supabase pgvector ($0~25/월) | ChromaDB (로컬) | 순수 Python, pip 설치 |
| BM25 | PostgreSQL tsvector | rank_bm25 (순수 Python) | 외부 의존 없음 |
| Hybrid merge | SQL RPC (0.7vec + 0.3bm25) | RRF k=60 (Python) | 점수 스케일 무관 |
| 유료 API | 필수 | `.env` 키 있으면 자동 활성화 | 옵션 |

### 확정된 배포 로드맵

| Phase | 대상 | 형태 | 선결 조건 |
|-------|------|------|-----------|
| 1 | 개발자 | `pip install` + CLI | 소주제1 CRITICAL 버그 수정 |
| 2 | 준개발자 | Docker Compose + 웹 UI (Gradio) | citation 준수 실험 완료 |
| 3 | 비개발자 | HuggingFace Spaces / 호스팅 | 비용 모델 결정 |

---

## 통합 실행 계획

소주제 1(구조 결함) + 소주제 2(비용 제로)에서 도출된 작업을 우선순위 순서로 통합:

| 순서 | 작업 | 근거 | 복잡도 |
|------|------|------|--------|
| 1 | 벡터 차원 통일 (config ↔ SQL) | CRITICAL — 다른 모든 작업의 전제 | 중 (마이그레이션) |
| 2 | tiktoken 토큰 추정 교체 | 답변 품질 직결 | 저 |
| 3 | 로컬 모델 citation 준수 실험 | 비용 제로 전환의 핵심 리스크 | 중 (실험) |
| 4 | ChromaDB + rank_bm25 + RRF 구현 | Supabase 의존 제거 | 중 |
| 5 | Sufficiency Gate 복합 판정 | false positive 감소 | 저-중 |
| 6 | 캐시 메타데이터 추가 + topic별 무효화 | 비용 효율 | 중 |
| 7 | Prompt Injection 입력 sanitize | 공개 배포 전 필수 | 저-중 |
| 8 | Python/Supabase Functions 정리 | 이중 구현 제거 | 저 |
| 9 | Gradio/Streamlit 웹 UI | Phase 2 배포 | 중 |

---

## 기술적 개선 사항 (상세)

각 항목은 "문제 → 논의 과정 → 해소 → 개선된 점"을 개별 문서로 기록:

| # | 개선 사항 | 핵심 | 라운드 |
|---|----------|------|--------|
| 1 | [벡터 차원 불일치](improvements/01-벡터차원-불일치.md) | SQL(384) vs Config(1536) — 런타임 크래시 | 소주제1, R1 |
| 2 | [토큰 추정 결함](improvements/02-토큰추정-결함.md) | ÷3 추정 → tiktoken 교체 필요 | 소주제1, R1 |
| 3 | [Sufficiency Gate](improvements/03-sufficiency-gate.md) | max만 → 복합 판정 (count + median) | 소주제1, R1 |
| 4 | [캐시 전역 flush](improvements/04-캐시-전역flush.md) | invalidate_all → topic별 무효화 | 소주제1, R1 |
| 5 | [Prompt Injection](improvements/05-prompt-injection.md) | 입력 필터링 없음 → sanitize 필요 | 소주제1, R3 |
| 6 | [비용 제로 전환](improvements/06-비용제로-전환.md) | 유료 API → ChromaDB + rank_bm25 + RRF + Ollama | 소주제2, R1~R4 |
| 7 | [배포 단계화](improvements/07-배포-단계화.md) | 개발자 → 준개발자 → 비개발자 3단계 | 소주제2, R3~R4 |

---

## 1차 설계 검증과의 비교

| 영역 | 1차 설계 (2026-04-06) | 2차 구현 (2026-04-08) |
|------|---------------------|---------------------|
| 검증 대상 | 설계 문서 (코드 없음) | 실제 구현 코드 |
| 모델 구성 | 3모델 (Claude/Codex/Gemini) | 단일 모델 (Claude 3역할) |
| 라운드 | 21R | 7R |
| 모순 제기 | 47건 | 11건 |
| VALID | 98% | 91% |
| 핵심 발견 | 설계 모순 (청크 ID 7번 재설계) | 설계-구현 괴리 (차원 불일치, gate 변질) |
| 도출물 | 확정 설계서 | 실행 계획 (9단계) + 비용 제로 구성 |

### 비교 결론

1차는 "**어떻게 만들어야 하는가**"를, 2차는 "**만든 것이 설계와 맞는가 + 어떻게 배포할 것인가**"를 검증. 두 검증은 상호 보완적이며, 2차에서 발견된 설계-구현 괴리는 1차만으로는 절대 발견할 수 없었던 문제들.

---

## 시스템 검증 결과

이번 실험에서의 적대적 검증 시스템 자체 평가:

| 구조 | 작동 여부 | 근거 |
|------|----------|------|
| 단일 모델 3역할 | **제한적 작동** | 편향 위험 증가하나, 구조적 결함 발견에는 유효 |
| 코드 기반 검증 | **작동** | 설계 문서 없이 코드만으로 결함 식별 가능 확인 |
| Challenger 자기 제한 | **작동** | R4에서 Challenger가 범위 이탈 자인 → 수렴 판정 |
| 설계-구현 대조 | **작동** | 1차 설계 확정안과 실제 코드의 괴리 8건 중 CRITICAL 1건 발견 |
