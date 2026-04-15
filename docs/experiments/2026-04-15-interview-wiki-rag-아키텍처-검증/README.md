# interview-wiki-rag 아키텍처 적대적 검증 (2026-04-15)

> 프로젝트의 아키텍처와 모델 선택이 최선인지 검증 + 모델 벤치마크 7개 비교

---

## 결론

### 실무 결론: "무료 RAG의 근본 모순 발견 + 모델 교체로 해소"

| 지표 | 검증 전 (Llama 3.1 8B) | 검증 후 (Qwen3-30B-A3B) |
|------|----------------------|------------------------|
| 인용 [N] 준수 | 미준수 (4차 검증에서 발견) | A+ (벤치마크 검증) |
| 한국어 품질 | 약함 | A+ (CJK 특화) |
| 추론 속도 | - | 3.7초 (NIM 기준) |
| 뉴런 효율 | 8B dense 전체 활성 | 3B만 활성 (MoE) |
| 무료 티어 지속성 | 위험 | 개선 (뉴런 소비 감소) |

### 핵심 발견

1. **무료 티어 순환 의존성 (C-19)** - 4라운드에 걸쳐 발견. 모든 개선안이 Cloudflare 무료 한도에 종속되어 순환 의존
2. **Python 백엔드와 Web 배포의 완전 분리** - 정교한 RAG 파이프라인이 프로덕션에서 전혀 사용되지 않음
3. **모델 생태계 최신화 실패** - Llama 8B만 있다고 가정했으나, 실제로는 Gemma 4, Qwen3가 Workers AI에 탑재되어 있었음
4. **프로젝트 목표 미확정** - 포트폴리오 vs 실서비스 판단이 선행되어야 기술 결정 가능

---

## 실험 설계

| 항목 | 값 |
|------|-----|
| 검증 주제 | interview-wiki-rag 아키텍처와 모델 선택의 최적성 |
| Executor | Claude (Opus 4.6) |
| Challenger | Codex (GPT-5.2) |
| Arbiter | Gemini (rate limit으로 Claude 대행) |
| 라운드 수 | 4 |
| 총 모순 | 22건 |
| VALID | 14건 |
| WEAK | 5건 |
| INVALID | 1건 |
| Collapse | 없음 |
| Frame Shift | C-20에서 감지 |

---

## 라운드별 진행

### Round 1: Challenger 7건 제기

| ID | 유형 | 대상 | 심각도 | 판정 |
|----|------|------|:------:|:----:|
| C-01 | assumption_flaw | Workers 번들 크기 vs 임베딩 JSON | critical | VALID |
| C-02 | missing_case | Supabase API 키 하드코딩 | high | VALID |
| C-03 | logic_error | hybrid->vector-only 전환 시 BM25 누락 | high | VALID |
| C-04 | side_effect | 인용 재생성 Workers 타임아웃 | high | VALID |
| C-05 | missing_case | 충분성 게이트 임계값 캘리브레이션 | medium | VALID |
| C-06 | scalability_risk | O(n) 선형 스캔 | medium | WEAK |
| C-07 | frame_shift | 대화 이력 3개 제한 | low | WEAK |

### Round 2: 새 모순 7건 + 이전 미해소 추적

| ID | 유형 | 대상 | 심각도 | 판정 |
|----|------|------|:------:|:----:|
| C-08 | assumption_flaw | 무료 티어 한도 충돌 | critical | VALID |
| C-09 | missing_case | 인용 검증 뉴런 소비 | high | WEAK |
| C-10 | side_effect | BM25 Workers 내 구현 한계 | high | VALID |
| C-11 | logic_error | Supabase 서버사이드 접근 | high | WEAK |
| C-12 | missing_case | Llama 8B 한국어 faithfulness 미검증 | critical | VALID |
| C-13 | scalability_risk | LangGraph 상태 격리 | critical | INVALID |
| C-14 | frame_shift | Rate limiting 부재 | high | VALID |

### Round 3: 근본 원인 수렴

| ID | 유형 | 대상 | 심각도 | 판정 |
|----|------|------|:------:|:----:|
| C-15 | frame_shift | "양립 불가" 프레임 재정의 | critical | VALID |
| C-16 | implementation_gap | 한국어 BM25 Supabase 미지원 | high | VALID |
| C-17 | verification_insufficiency | 골드셋 50쌍 통계적 부족 | medium | WEAK |
| C-18 | circular_dependency | KV eventual consistency rate limit 우회 | high | VALID |
| C-19 | dependency_chain | 무료 티어 순환 의존성 | critical | VALID |

### Round 4: 해소 재검증

| ID | 유형 | 대상 | 심각도 | 판정 |
|----|------|------|:------:|:----:|
| C-20 | escalation_evasion | 에스컬레이션 타이밍 | critical | VALID |
| C-21 | premature_resolution | C-09 해소 철회 (regex != faithfulness) | moderate | VALID |
| C-22 | incomplete_resolution | C-11 latency 미해소 | moderate | WEAK |

---

## 해소/미해소 분류

### 해소 (5건)
- C-01: Cloudflare Vectorize / KV 분리 저장
- C-02: Cloudflare Env Variables로 이동
- C-03: search.ts BM25 + 벡터 결합
- C-04: 재생성 포기, 검증만 수행
- C-07: 의도적 제약

### 미해소 (11건) - 사람 판단 필요
- **C-19**: 무료 티어 순환 의존성 (critical)
- **C-12**: Llama 8B 한국어 faithfulness (critical) -> 모델 교체로 해소
- **C-20**: 에스컬레이션 타이밍 (critical)
- C-08, C-10, C-14, C-16, C-18: 무료 한도 관련
- C-05, C-17, C-21: 검증 방법론

---

## 후속 조치: 모델 벤치마크

검증에서 발견된 C-12(Llama 8B 품질)를 해소하기 위해 7개 모델 비교.

### 서버 모델 (API)

| 순위 | 모델 | 속도 | 인용 | 한국어 | 플랫폼 |
|:----:|------|-----:|:----:|:------:|--------|
| 1 | **Qwen3-80B-A3B** | 3.7s | A+ | A+ | OpenRouter, NIM |
| 2 | Gemma 4 26B-A4B | 6.2s | A+ | A+ | OpenRouter, Gemini CLI |
| 3 | Kimi K2 | 10.1s | A+ | A | NIM |
| 4 | DeepSeek V3.2 | 36.5s | A | A | NIM |
| 5 | Nemotron Nano 30B | 3.4s | F | A | NIM |
| 6 | Nemotron 120B | 40.3s | A | A | OpenRouter |
| X | GLM-5 | timeout | - | - | NIM |

### CPU 모델 (Ollama 로컬)

| 모델 | 크기 | 속도 | 인용 | 한국어 |
|------|-----:|-----:|:----:|:------:|
| Qwen2.5-7B | ~5GB | 59s | 4개 | OK |
| Qwen2.5-3B | ~2GB | 3.7s | 0개 | 어색 |
| Phi-4-mini | ~2.5GB | 6.8s | 3개 | 약함 |
| Gemma 4 E4B | ~10GB | 60s | 23개 | 영어 thinking |
| Gemma 4 E2B | ~5GB | 91s | 12개 | 영어 thinking |
| SmolLM2 1.7B | ~1.8GB | 6.2s | 0개 | 깨짐 |

---

## 프로세스 개선 도출

이번 실험에서 6가지 프로세스 개선을 도출하여 adversarial-verify 스킬 v2.1에 반영.

| 개선 | 출처 | 효과 |
|------|------|------|
| --min-rounds | 사용자 요청 | 조기 종료 방지 |
| Phase 0.5 생태계 최신화 검증 | Gemma 4/Qwen3 미인지 사태 | 구식 전제 방지 |
| Phase 2.5 해소 재검증 | C-09, C-11 해소 철회 | 과대평가 방지 |
| Phase 2.6 의존성 체인 탐지 | C-19 순환 종속 | 근본 원인 조기 발견 |
| Phase 2.7 에스컬레이션 타이밍 검증 | C-20 REACTIVE 판정 | 프레임 전환 추가 방지 |
| Arbiter 폴백 체인 | Gemini rate limit | 모델 장애 대응 |
| Challenger 관점 8번 | 모델 생태계 변화 | 구식 전제 탐지 |

상세: [adversarial-verify.md](https://github.com/cho1124/multi-agent-adversarial-verification/blob/main/.claude/commands/adversarial-verify.md)

---

## 한계

1. **Arbiter 모델 장애** - Gemini rate limit으로 Claude가 Arbiter 겸임. 이해 충돌 가능성
2. **모델 벤치마크 불완전** - OpenRouter 무료 한도 소진으로 Qwen3 TC2-4 미완 (NIM으로 보완)
3. **Cloudflare Workers AI 실환경 미검증** - 벤치마크는 OpenRouter/NIM 경유. 실제 Workers AI 성능과 차이 가능
