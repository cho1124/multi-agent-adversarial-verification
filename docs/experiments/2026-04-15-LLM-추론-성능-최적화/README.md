# LLM 추론 성능 최적화 적대적 검증 (2026-04-15)

> interview-wiki-rag 프로젝트에 대한 LLM 추론 성능 최적화 방안 우선순위 검증. 2회 검증 수행.

---

## 결론

### 실무 결론: "Cloud API 프로젝트에서의 최적화는 전제부터 재검토해야 한다"

| 지표 | 대화 기반 논의 (검증 전) | 적대적 검증 (3모델) |
|------|----------------------|-------------------|
| 양자화/경량 모델 교체 | "1순위 최적화" | **적용 불가** (Cloud API) |
| 프로파일링 | 미언급 | **모든 최적화의 전제조건** |
| 평가 인프라 | 미언급 | **골드 데이터셋 필수** |
| 리랭커 도입 | "토큰 50-70% 감소" | **관련도 향상이 목적** (2800토큰 환경) |
| 시스템 프롬프트 축소 | "중간 효과" | **미미한 효과** (전체 토큰 대비 비중 작음) |

**핵심:** 대화에서 도출한 최적화 방안(양자화, 경량 모델, 입력 줄이기)은 **로컬 추론 환경** 기준이었고, 실제 프로젝트는 **클라우드 API 기반**이라 전제 자체가 틀렸다. 적대적 검증이 이 전제 오류를 Round 1에서 즉시 포착했다.

---

## 검증 1차: Cloud 모드 최적화 우선순위

### 메타데이터

| 항목 | 값 |
|------|---|
| 날짜 | 2026-04-15 |
| Executor | Claude (Opus 4.6) |
| Challenger | Codex (GPT-5.2) |
| Arbiter | Gemini (3 Pro) |
| 라운드 | 4 |
| 모순 | 28 |
| 해소 | 28 (100%) |
| VALID 비율 | 85% (24/28) |
| CRITICAL | 6 (모두 해소) |
| 종료 | Rational Consensus |
| Collapse | 없음 |
| Frame Shift | 없음 |

### CRITICAL 발견 사항

| ID | 내용 | 해소 방법 |
|----|------|----------|
| C-05 | 병목이 LLM인지 DB/네트워크인지 모르면서 LLM 최적화에만 집중 | Phase 0 프로파일링 추가 |
| C-20 | 평가 데이터셋(골드 Q&A) 없이 성공 판단 불가 | Phase 0에 골드 데이터셋 구축 포함 |
| C-02 | 2800토큰 컨텍스트에서 50-70% 감소 시 Sufficiency Gate 급락 | 리랭커 목적을 "토큰 감소"→"관련도 향상"으로 재정의 |
| C-14 | Gate(임베딩 점수)와 리랭커(cross-encoder 점수) 판단 충돌 | 리랭커 점수를 Gate 유일 입력으로 통합 |
| C-17 | 검색 품질 베이스라인 미측정 → 리랭커 효과 비교 불가 | Phase 0에 precision@5, recall@10, MRR 측정 포함 |
| C-21 | 골드 50~100쌍으로 Gate 임계값 보정은 통계적 불충분 | 초기 시드 + 운영 피드백으로 300+쌍 점진 확대 |

### 확정된 Cloud 최적화 Phase

```
Phase 0: 베이스라인 수립 + 평가 인프라
├─ 골드 Q&A 50~100쌍 시드 (점진 확대 300+)
├─ 검색 품질 (precision@5, recall@10, MRR)
├─ 레이턴시 브레이크다운 (서버사이드 기준)
├─ 캐시 히트율 + 라우팅 비율
└─ Phase 진입/스킵 기준 (부트스트랩 95% CI)

Phase 1: 리랭커 도입 (precision@5 CI 상한 < 0.7일 때)
├─ Cohere Rerank API
├─ top-k 고정 (simple=10, complex=15)
├─ 부모 제한 (light=2, heavy=4)
├─ 폴백: 500ms 타임아웃 → 임베딩 유사도 Gate (보수적 임계값)
└─ 1~2주 운영 데이터 수집

Phase 2: 프롬프트 캐싱 (LLM 레이턴시 > 서버사이드 60%)
├─ Claude: cache_control (80% 레이턴시, 90% 비용 절감)
└─ OpenAI: cached input pricing (50% 할인)

Phase 3: 모델 업그레이드 (진입 시 최신 벤치마크 재확인)
Phase 4: Gate 재보정 (Phase 1 운영 데이터 기반)
Phase 5: 캐시 히트율 최적화 (히트율 < 40% → 목표 60%+)
```

### 라운드별 모순 추이

| 라운드 | 새 모순 | VALID | WEAK | INVALID | Challenger 상태 |
|:---:|:---:|:---:|:---:|:---:|---|
| R1 | 10 | 10 | 0 | 0 | sustained_disagreement |
| R2 | 10 | 8 | 1 | 0 | sustained_disagreement |
| R3 | 8 | 6 | 2 | 0 | sustained_disagreement |
| R4 | 0 | - | - | - | **rational_consensus** |

---

## 검증 2차: Local 모드 모델 교체 및 알고리즘 개선

### 메타데이터

| 항목 | 값 |
|------|---|
| 날짜 | 2026-04-15 |
| Executor | Claude (Opus 4.6) |
| Challenger | Codex (GPT-5.2) |
| Arbiter | Gemini (3 Pro) |
| 라운드 | 3 |
| 모순 | 13 |
| 해소 | 13 (100%) |
| VALID 비율 | 100% (13/13) |
| CRITICAL | 3 (모두 해소) |
| 종료 | Rational Consensus |
| Collapse | 없음 |
| Frame Shift | R1에서 감지 (R2에서 해소) |

### CRITICAL 발견 사항

| ID | 내용 | 해소 방법 |
|----|------|----------|
| C-01 | BGE-M3(1024차원)로 교체 시 벡터DB 인덱스 전체 재생성 비용 미산정 | Matryoshka 384차원 축소 + 재인덱싱 스텝 포함 |
| C-02 | MTEB 범용 벤치마크로 도메인 특화 교체 순서 확정은 논리 비약 | Phase 0 한국어 면접 도메인 실측 기반으로 전환 |
| C-03 | Cloud(384차원)와 Local(1024차원) 불일치로 품질 격차 확대 | 384차원 통일로 해소 |

### 기술적으로 중요한 발견

**1. 차원 수가 같아도 벡터 공간이 다르면 호환 불가 (C-09)**

BGE-M3와 text-embedding-3-small 모두 384차원을 출력할 수 있지만, 서로 다른 임베딩 공간에 매핑하므로 기존 인덱스에 새 모델 벡터를 넣으면 검색이 무의미해진다. **전체 재인덱싱 필수.**

**2. Matryoshka는 출력만 잘라냄, 모델 가중치는 그대로 (C-12)**

BGE-M3 384차원 모드는 출력 벡터를 384차원으로 truncate할 뿐, 모델 자체는 568M 파라미터 전체를 로딩한다. 메모리 사용량은 ~1.1GB(FP16)이지 ~300MB가 아니다.

**3. 한국어 토큰화 차이 (C-13)**

500자 한국어 텍스트는 영어 기준 ~170토큰이 아닌 300~400토큰(XLM-RoBERTa 토크나이저). BGE-M3 max_seq_length 512토큰에 여유가 100~200토큰밖에 없으므로 청크 크기 설정 시 주의.

### 확정된 Local 모델 교체 계획

| 구분 | 현재 | 제안 | 메모리 |
|------|------|------|:---:|
| LLM | gemma2:9b (~5GB) | gemma4:e4b Q4 (~3GB) | -2GB |
| 임베딩 | all-MiniLM-L6-v2 (~80MB) | BGE-M3 384차원 (~1.1GB) | +1GB |
| 리랭커 | 없음 | ms-marco-MiniLM-L-6-v2 (~80MB) | +80MB |
| 합계 | ~5.1GB | ~4.2~4.8GB | 비슷~경량 |

### 알고리즘 개선 옵션

| 기법 | 효과 | 추가 비용 | 면접 위키 적합성 |
|------|------|----------|:---:|
| 리랭커 | precision 향상 | CPU 100-300ms | 높음 |
| Query Expansion | recall 향상 (동의어/영한 혼용) | 낮음 | 높음 |
| HyDE | 시맨틱 갭 해소 | LLM 추가 호출 1회 | 중간 |
| RAPTOR | 계층적 요약 검색 | 사전 요약 비용 큼 | 낮음 (Q&A 구조 비적합) |

### Local 적용 Phase

각 Phase마다 A/B 비교. 게이트: precision@5 >= 베이스라인 + 5%p.

```
Phase 0: 골드 데이터셋 + 베이스라인 (한국어 면접 도메인 실측)
Phase 1: 임베딩 교체 (BGE-M3 384차원) + ChromaDB 재인덱싱 → A/B
Phase 2: LLM 교체 (gemma4:e4b) → A/B
Phase 3: 리랭커 (ms-marco-MiniLM-L-6-v2) → A/B
Phase 4: Query Expansion (한국어 동의어) → A/B
Phase 5: HyDE/RAPTOR (효과 측정 후 선택적)
```

### 라운드별 모순 추이

| 라운드 | 새 모순 | VALID | WEAK | INVALID | Challenger 상태 |
|:---:|:---:|:---:|:---:|:---:|---|
| R1 | 8 | 8 | 0 | 0 | sustained_disagreement + frame_shift |
| R2 | 5 | 5 | 0 | 0 | sustained_disagreement + frame_shift |
| R3 | 0 | - | - | - | **rational_consensus** |

---

## 생태계 스냅샷 (2026-04-15 시점)

### 모델 현황

| 모델 | 상태 | 비고 |
|------|------|------|
| GPT-4o | 2026-02 ChatGPT 은퇴 | API 레거시 유지 |
| GPT-4o-mini | 현역 | GPT-5-mini가 후계 |
| GPT-5-mini/nano | 최신 | 저비용 고성능 |
| Claude Sonnet 4 | 현역 | 프롬프트 캐싱 80% 레이턴시 감소 |
| Gemma 4 E4B | 최신 | 140+ 언어 네이티브, Ollama 공식 지원 |
| all-MiniLM-L6-v2 | MTEB 56.3 | production RAG 부적합 권고 |
| BGE-M3 | 최신 권장 | 100+ 언어, 한국어 강세 |

### 리랭커 현황

| 모델 | 크기 | CPU 성능 | 비고 |
|------|------|---------|------|
| ms-marco-MiniLM-L-6-v2 | 33M | 2-5ms/pair | 최경량 |
| ms-marco-MiniLM-L-12-v2 | ~80M | ~130ms/16pair | 속도/품질 균형 |
| BGE-reranker-v2-m3 | 278M | ~130ms/16pair | 다국어 최고 |
| FlashRank | 경량 | 빠름 | 설치 간편 |
| Cohere Rerank v3 | API | 빠름 | 프로덕션 안정 |

---

## 교훈

1. **전제를 먼저 검증하라** — "추론 성능 최적화"라는 주제에서 양자화/경량 모델이 1순위로 논의되었으나, Cloud API 환경에서는 적용 자체가 불가능했다. Challenger가 R1에서 즉시 포착.

2. **"드롭인 교체"를 의심하라** — 차원 수가 같아도 임베딩 모델이 다르면 벡터 공간이 다르다. Executor가 "384차원 동일 → 재인덱싱 불필요"라고 주장했으나, Challenger가 논파.

3. **메모리 계산에서 출력과 가중치를 혼동하지 마라** — Matryoshka 384차원은 출력 truncation이지 모델 경량화가 아니다.

4. **범용 벤치마크로 도메인 특화 결정을 내리지 마라** — MTEB 점수로 한국어 면접 위키의 검색 품질을 예측할 수 없다. 도메인 실측이 필수.
