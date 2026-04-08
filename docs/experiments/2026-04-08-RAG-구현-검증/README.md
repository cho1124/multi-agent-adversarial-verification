# RAG 시스템 2차 구현 검증 실험 (2026-04-08)

> 1차 설계 검증(2026-04-06)에서 확정된 RAG 설계를 실제 구현한 interview-wiki-rag의 아키텍처 정합성 + 비용 제로 공개 배포 전략 검증. **3모델 적대적 검증 (Claude/Codex/Gemini).**

---

## 결론

### 실무 결론: "설계-구현 괴리는 3모델 검증에서 2.5배 더 많이 드러난다"

| 지표 | 단일모델 (Claude 3역할) | 3모델 (Claude/Codex/Gemini) |
|------|----------------------|---------------------------|
| 라운드 | 7R | **7R** (동일) |
| 모순 제기 | 11건 | **40건** (+263%) |
| VALID | 10 (91%) | **39 (97.5%)** |
| WEAK | 1 (9%) | **1 (2.5%)** |
| CRITICAL 발견 | 차원 불일치 1건 | **차원 불일치 + match_chunks 삭제 + citation 비강제 + 캐시 미연결 + 라우터 SPOF** 5건 |
| 단일모델 미발견 | — | **30건** (C2,C3,C5,C6,C8,C11~C25 + S2 C7,C10~C15) |

**핵심:** 단일모델은 "명백한 수치 불일치"(차원 384 vs 1536)는 찾았지만, "코드 경로 분석"(match_chunks 삭제, citation 비강제, 캐시 무효화 미연결)은 놓쳤다. Codex(GPT-5.4)가 코드를 직접 읽고 행 번호까지 인용하며 반박한 것이 결정적 차이.

### 단일모델 대비 3모델이 추가로 발견한 핵심 결함

| # | 결함 | 단일모델 | 3모델 | 차이 |
|---|------|---------|------|------|
| 1 | v3 SQL에서 match_chunks 삭제 → quiz/explain/compare 깨짐 | 미발견 | **C2 VALID** | Codex가 SQL 파일 직접 읽어서 발견 |
| 2 | search 노드가 agentic loop이 아닌 단순 생성기 | 미발견 | **C3 VALID** | 도구 바인딩 vs 실제 tool_calls 처리 구분 |
| 3 | 캐시 무효화가 인덱싱 파이프라인에 미연결 | "전역 flush"만 지적 | **C5 VALID** | store_chunks()에서 호출 자체가 없음 |
| 4 | citation validation이 비강제 (로그 수준) | 미발견 | **C6 VALID** | validate→재생성/거절 로직 부재 |
| 5 | 보호 장치가 search에만 적용 | 미발견 | **C8 VALID** | quiz/explain/compare에 gate/citation 없음 |
| 6 | 라우터 JSON 파싱 fallback 없음 | 미발견 | **C11 VALID** | SPOF — 전체 그래프 중단 |
| 7 | 라우터 semantic validation 없음 | 미발견 | **C16 VALID** | agent_type="foo" → KeyError |
| 8 | P1 항목 병렬 수정 불가 (강결합) | 미발견 | **C17 VALID** | workflow.py 196~223 핫스팟 공유 |
| 9 | L1/L2 캐시 키에 설정 미반영 | 미발견 | **C18,C19 VALID** | threshold 변경해도 30분/1시간 지연 |
| 10 | Supabase Edge Functions 이중 규칙 | "이중 구현" 정도 | **C20 VALID** | gate 기준 0.3/0.5 vs 0.5/0.7 정량 비교 |
| 11 | _run_agent() 1회 왕복 제한 | 미발견 | **C21 VALID** | while loop 아닌 공통 런타임 결함 |
| 12 | L3 캐시 키 sorted vs 인용번호 점수순 | 미발견 | **C14 VALID** | correctness bug, P2→P1 상향 |
| 13 | topic_lookup/relation_lookup Supabase 직접 조회 | 미발견 | **S2-C7 VALID** | 벡터만 로컬화해도 데이터 원본은 유료 |
| 14 | auto-detect 규칙 모호 (키 조합별 깨짐) | 미발견 | **S2-C13 VALID** | Anthropic만 있으면 임베딩 깨짐 |

### 검증된 것

1. **3모델 검증이 단일모델 대비 2.5배 더 많은 결함을 발견한다** — 동일 라운드(7R)에서 11건→40건
2. **Codex의 코드 직접 읽기가 결정적** — 행 번호 인용, SQL 파일 교차 대조, 호출 경로 추적
3. **Gemini의 대칭적 판정이 품질을 보장** — Challenger 과장 1건(C25)을 WEAK로 걸러냄
4. **의존성 그래프 기반 수정 순서가 합의됨** — P0→P1-A(캐시)→P1-B(state)→P1-C(로직)→P1-D(횡단)→P2

### 검증되지 않은 것 (한계)

1. **구현 검증 미수행** — 결함 식별만 완료, 실제 수정 코드는 아직 작성하지 않음
2. **retrieval 품질 비교 미수행** — 로컬 스택(ChromaDB+RRF) vs Supabase(pgvector+BM25) 성능 비교 실험 부재
3. **로컬 LLM citation 준수율 미검증** — gemma2:9b가 `[1]` 형식을 안정적으로 생성하는지 실험 필요

---

## 요구사항 점검

### 1. 무료인가?

**현재: 아니오. 계획: 가능하지만 구현 필요.**

| 구성요소 | 현재 | 비용 제로 계획 | 상태 |
|----------|------|-------------|------|
| Embedding | OpenAI text-embedding-3-small (~$18/월) | gte-multilingual-base (로컬) | 미구현 |
| LLM | gpt-4o-mini + claude-sonnet (~$1.4/월) | Ollama gemma2:9b (로컬) | 미구현 |
| Vector DB | Supabase pgvector ($0~25/월) | ChromaDB (로컬) | 미구현 |
| BM25 | PostgreSQL tsvector | rank_bm25 또는 SQLite FTS5 | 미구현 |
| **합계** | **$20~45/월** | **$0** | **미구현** |

**검증에서 확인된 추가 작업:**
- config.py에 `mode=cloud|local` 명시적 설정 추가 (auto-detect는 보조)
- 6개 어댑터(source/vector/embedding/LLM/BM25/metadata) 구현 필요
- migration/reindex 스크립트 필요
- 차원 3중 충돌(1536/384/768) 통일

### 2. 다른 사용자들도 쉽게 접근할 수 있는가?

**현재: 아니오.**

| 접근 방법 | 상태 | 문제 |
|----------|------|------|
| CLI (`scripts/chat.py`) | 존재 | 개발자만 가능, API 키 필수 |
| Supabase Edge Functions | 존재 | 별도 Supabase 프로젝트 필요 |
| 웹 UI (Gradio/Streamlit) | **미구현** | Dockerfile도 없음 |
| Docker Compose | **미구현** | — |
| HuggingFace Spaces | **미구현** | Ollama 서빙 재설계 필요 |

**검증에서 확정된 배포 로드맵:**
- Phase 1 (개발자): pip install + CLI + single-folder local-first
- Phase 2 (준개발자): Docker Compose + Gradio 웹 UI
- Phase 3 (비개발자): HF Spaces / 호스팅

### 3. 프론트엔드 파트 개발 했는가?

**아니오. 전혀 없음.**

현재 진입점:
- `scripts/chat.py` — Python CLI (input() 루프)
- `supabase/functions/chat/index.ts` — Edge Function (API endpoint)

웹 UI, 모바일, 데스크톱 앱 모두 미구현. Phase 2에서 Gradio/Streamlit 계획.

### 4. 추가적인 보안 이슈 혹은 논리의 허점?

**있음. 검증에서 발견된 항목:**

| # | 이슈 | 심각도 | 상태 |
|---|------|--------|------|
| 1 | **Prompt Injection** — 사용자 쿼리가 필터링 없이 LLM에 직접 전달 | HIGH | 미수정 |
| 2 | **라우터 SPOF** — JsonOutputParser 예외처리 없음 (C11) + semantic validation 없음 (C16) | CRITICAL | 미수정 |
| 3 | **Citation 비강제** — 검증 결과를 계산만 하고 재생성/거절 안 함 (C6) | HIGH | 미수정 |
| 4 | **캐시 정합성** — L1 키에 model/complexity 미반영 → 잘못된 응답 반환 (C23) | HIGH | 미수정 |
| 5 | **Edge Functions 이중 규칙** — Python과 다른 gate 기준(0.3/0.5 vs 0.5/0.7) (C20) | MEDIUM | 폐기 예정 |
| 6 | **L3 캐시 인용 오매핑** — sorted(chunk_ids) vs 점수순 인용번호 (C14) | HIGH | 미수정 |

### 5. 기존 면접위키 DB에서 정상적으로 성능 이슈를 개선했는가?

**부분적. 설계는 있지만 구현이 변질/누락.**

| 설계 (1차 검증) | 구현 상태 | 검증 결과 |
|----------------|----------|----------|
| 3계층 캐시로 비용 절감 | 구현됨 (in-memory dict) | **C4**: 재시작 시 소멸, 멀티프로세스 공유 없음 |
| 3계층 동시 무효화 | `invalidate_all()` 한 줄 | **C5**: 파이프라인에 연결 안 됨 |
| Sufficiency gate 2단계 | `max(score)` 하나만 | **C7,C13**: gate-검색 threshold 미분리 |
| Hybrid search (vec+BM25) | Supabase RPC 가중합 | **C9**: BM25 정규화 없이 실효 가중치 무의미 |
| 청크 ID 안정성 (content_hash) | 구현됨 | 정상 작동 |
| 멀티에이전트 라우팅 | 구현됨 (router→4 agent) | **C2,C3**: v3 RPC 깨짐 + search는 생성기 |
| 멀티턴 대화 | state에 선언만 | **C15,C24**: 실제 messages 무시, 횡단 변경 필요 |

---

## 실험 환경

| 역할 | 모델 | 호출 방식 | 버전 |
|------|------|----------|------|
| Executor | Claude Opus 4.6 | 메인 세션 | claude-opus-4-6 |
| Challenger | **Codex GPT-5.4** | codex exec --ephemeral | codex-cli 0.118.0 |
| Arbiter | **Gemini 3 Pro** | gemini CLI | gemini-cli 0.25.1 |

**검증 대상**: [cho1124/interview-wiki-rag](https://github.com/cho1124/interview-wiki-rag) 커밋 `225cdb3`

---

## 전체 결과 요약

| 항목 | 소주제1: 아키텍처 | 소주제2: 비용 제로 배포 | 합계 |
|------|-----------------|---------------------|------|
| 라운드 | 5R | 2R | **7R** |
| 총 모순 제기 | 25건 | 15건 | **40건** |
| VALID | 24 (96%) | 15 (100%) | **39 (97.5%)** |
| WEAK | 1 (4%) | 0 | **1 (2.5%)** |
| INVALID | 0 | 0 | **0** |
| Collapse | 없음 | 없음 | **없음** |
| 프레임 전환 감지 | 0회 | 0회 | **0회** |
| 종료 방식 | conditional_consensus | conditional_consensus | |

---

## 소주제1: 현재 아키텍처 구조 검증

### 초기 제안 (Executor)

멀티에이전트(router→search/quiz/explain/compare), 3계층 캐시, Sufficiency Gate + Citation, Hybrid Search가 구현되어 있으며, RAG 시스템으로서 기본 완성도를 갖추었다.

### 라운드별 진행

| Round | 모순 | VALID | WEAK | 핵심 쟁점 |
|-------|------|-------|------|-----------|
| R1 | 10건 | 10 | 0 | 차원 불일치(C1), match_chunks 삭제(C2), search 비에이전트(C3), 캐시 dict(C4), 무효화 미연결(C5), citation 비강제(C6), gate 단일값(C7), search만 보호(C8), BM25 미정규화(C9), 프롬프트 미사용(C10) |
| R2 | 5건 | 5 | 0 | 라우터 SPOF(C11), citation 강제 주장 논리비약(C12), gate-검색 threshold 미분리(C13), L3 캐시 키 순서(C14), 멀티턴 미구현(C15) |
| R3 | 6건 | 6 | 0 | 라우터 semantic validation(C16), P1 병렬 수정 불가(C17), L2 캐시 키 설정 미반영(C18), L1 캐시 키 불충분(C19), Edge Functions 이중 규칙(C20), _run_agent 1회 왕복(C21) |
| R4 | 4건 | 3 | 1 | 의존성 순서 오류(C22), L1 키 complexity 누락(C23), 멀티턴 횡단 변경(C24), 스크립트 문법(C25-WEAK) |
| R5 | 0건 | — | — | **conditional_consensus** (조건: Phase C가 search+공통 경로 모두 포함) |

### 핵심 전환점

**R1 — Codex가 10건 한꺼번에 제기 (단일모델 대비 +5건)**

Codex(GPT-5.4)가 모든 Python/SQL 파일을 직접 읽고 행 번호까지 인용하며 반박. 특히:
- C2: `setup_v3.sql`이 `match_chunks`를 DROP만 하고 재생성하지 않는 것 발견
- C6: `citation.py`의 validate → workflow.py의 캐시 저장 경로를 추적하여 "비강제" 입증
- C10: prompts/ 파일 5개 중 실제 로딩은 `search.txt` 1개뿐임을 확인

**R3 — 캐시 계층 전체가 공격받음**

C18(L2 키에 retrieval 설정 미반영) + C19(L1 키에 complexity 미반영) + C22(의존성 순서 오류)가 연쇄적으로 제기되어, "P1 수정해도 캐시가 우회한다"는 구조적 문제 확인.

### 확정된 수정 의존성 그래프

```
P0: 차원 통일 + match_chunks + 라우터 validation + 캐시-파이프라인 연결
  ↓
P1-Phase A: L1/L2/L3 캐시 키 버전업 (complexity/model/설정 포함) ← 선행
  ↓
P1-Phase B: state.py 계약 변경 (messages 기반 횡단)
  ↓
P1-Phase C: agentic loop (search + _run_agent 공통) + citation 강제 + gate threshold 분리
  ↓
P1-Phase D: 비search 보호 + 멀티턴 (엔트리포인트 포함 횡단)
  ↓
P2: 캐시 영속화, BM25 정규화, 프롬프트 통일
```

Edge Functions 폐기 확정.

---

## 소주제2: 비용 제로 + 공개 배포 전략

### 초기 제안 (Executor)

모든 유료 API를 로컬 대안(gte-multilingual-base + Ollama + ChromaDB + rank_bm25 + RRF)으로 교체, pip install로 간편 배포.

### 라운드별 진행

| Round | 모순 | VALID | 핵심 쟁점 |
|-------|------|-------|-----------|
| R1 | 9건 | 9 | 로컬 분기 없음(C1), auto-detect 미구현(C2), ChromaDB+RRF 코드 없음(C3), 차원 3중 충돌(C4), 이중 경로(C5), requirements 누락(C6), Supabase 데이터 원본 잔존(C7), 배포 파일 없음(C8), HF Spaces 비현실(C9) |
| R2 | 6건 | 6 | 설정 계약 재정의 필요(C10), Supabase 6곳 결합(C11), single-file→single-folder(C12), auto-detect 모호(C13), BM25 계약 소멸(C14), migration 스크립트 누락(C15) |

### 확정된 비용 제로 구성

| 구성요소 | 현재 (유료) | 확정 대안 |
|----------|------------|----------|
| Embedding | OpenAI text-embedding-3-small | gte-multilingual-base (768d, 로컬) |
| LLM | gpt-4o-mini + claude-sonnet | Ollama gemma2:9b-instruct |
| Vector DB | Supabase pgvector | ChromaDB (single-folder) |
| BM25 | PostgreSQL tsvector | SQLite FTS5 또는 rank_bm25 |
| Hybrid | Supabase RPC 가중합 | RRF k=60 (순위 기반) |
| 데이터 원본 | Supabase tables | SQLite (로컬) |
| 설정 모드 | 유료 API 하드코딩 | `mode=cloud\|local` 명시적 설정 |

### Phase 1 최소 요구사항 (확정)

1. local source adapter (SQLite)
2. local vector adapter (ChromaDB)
3. local embedding adapter (HuggingFace)
4. local LLM adapter (Ollama)
5. local BM25 adapter (FTS5 또는 rank_bm25)
6. migration/export + reindex 스크립트
7. `mode=cloud|local` config 설정
8. requirements.txt 갱신

---

## 기술적 개선 사항 (상세)

| # | 개선 사항 | 핵심 | 라운드 | 단일모델 대비 |
|---|----------|------|--------|-------------|
| 1 | [벡터 차원 불일치](improvements/01-벡터차원-불일치.md) | SQL(384) vs Config(1536) CRITICAL | S1-R1 | 동일 발견 |
| 2 | [토큰 추정 결함](improvements/02-토큰추정-결함.md) | ÷3 → tiktoken | S1-R1 | 동일 발견 |
| 3 | [Sufficiency Gate](improvements/03-sufficiency-gate.md) | max → 복합 판정 + threshold 분리 | S1-R1,R2 | 3모델이 threshold 분리(C13)까지 추가 발견 |
| 4 | [캐시 전역 flush](improvements/04-캐시-전역flush.md) | 파이프라인 미연결 + 키 불충분 | S1-R1,R3,R4 | 3모델이 미연결(C5)+키 문제(C18,C19,C22,C23)까지 심화 |
| 5 | [Prompt Injection](improvements/05-prompt-injection.md) | 입력 sanitize | S1-R1 | 동일 발견 |
| 6 | [비용 제로 전환](improvements/06-비용제로-전환.md) | 6 어댑터 + FTS5 + migration | S2-R1,R2 | 3모델이 Supabase 6곳 결합(C11)+BM25 소멸(C14) 추가 |
| 7 | [배포 단계화](improvements/07-배포-단계화.md) | single-folder local-first | S2-R1,R2 | 3모델이 single-file 과장 교정(C12) |
| 8 | [match_chunks 삭제](improvements/08-match-chunks.md) | v3 SQL DROP 미복원 → quiz/explain/compare 깨짐 | S1-R1 | **3모델만 발견** |
| 9 | [citation 비강제](improvements/09-citation-비강제.md) | 검증→재생성/거절 로직 부재 | S1-R1,R2 | **3모델만 발견** |
| 10 | [라우터 SPOF](improvements/10-라우터-SPOF.md) | JSON 파싱 + semantic validation 없음 | S1-R2,R3 | **3모델만 발견** |
| 11 | [캐시 키 정합성](improvements/11-캐시키-정합성.md) | L1/L2/L3 키에 설정/순서 미반영 | S1-R3,R4 | **3모델만 발견** |
| 12 | [agentic loop 제한](improvements/12-agentic-loop.md) | search+_run_agent 1회 왕복 | S1-R1,R3 | **3모델만 발견** |

---

## 단일모델 실험과의 비교

| 영역 | 단일모델 (Claude 3역할) | 3모델 (Claude/Codex/Gemini) |
|------|----------------------|---------------------------|
| Challenger 품질 | 구조적 모순 위주 | **코드 행 번호 인용 + SQL 교차 대조 + 호출 경로 추적** |
| Arbiter 품질 | 자기 판정 (편향 위험) | **Gemini 독립 판정, C25 WEAK 필터링** |
| 발견 깊이 | 표면적 (수치 불일치, 패턴 불균일) | **심층적 (캐시 우회, citation 비강제, 의존성 순서)** |
| 합의 품질 | 일반적 수정 방향 | **의존성 그래프 + Phase A~D + 조건부 합의** |
| VALID 비율 | 91% | **97.5%** (Challenger 반박 품질 향상) |
| 작업량 산정 | 9단계 리스트 | **P0 4건 + P1 4-Phase + P2 3건 + 6 어댑터** |

### 핵심 차이: Codex의 코드 직접 읽기

단일모델 실험에서 Claude가 3역할을 했을 때는 "코드를 한 번 읽고 기억에 의존"하는 구조. Codex는 **매 라운드마다 코드를 새로 읽고** 행 번호를 인용하며 반박하여:
- `setup_v3.sql`의 DROP 후 미복원 (C2)
- `workflow.py:196~223`의 핫스팟 공유 (C17)
- `cache/query_cache.py`의 키 생성 로직 (C19, C23)
- `supabase/functions/chat/index.ts`의 하드코딩 기준값 (C20)

등 단일모델이 놓친 14건의 결함을 추가 발견.

---

## 시스템 검증 결과

| 구조 | 작동 여부 | 근거 |
|------|----------|------|
| 3모델 분산 검증 | **작동** | Codex 코드 직접 읽기 + Gemini 독립 판정 = 단일모델 대비 +263% 발견 |
| Challenger 코드 분석 | **탁월** | 매 라운드 파일 재스캔, 행 번호 인용, SQL 교차 대조 |
| Arbiter 대칭성 | **작동** | C25 WEAK 판정 (Challenger 과장 필터링) |
| conditional_consensus | **작동** | 양 소주제 모두 조건부 합의로 건설적 종료 |
| Collapse 방지 | **해당 없음** | Executor가 매 라운드 모순 전부 인정 (방어 시도 없음) |
