# 멀티 에이전트 적대적 검증 시스템 — 전체 여정 요약

> 변증법(테제-안티테제-진테제) 기반 AI 설계 검증 시스템의 설계, 실험, 그리고 실제 프로덕트 적용까지의 전체 기록

---

## 한 줄 요약

AI 3개(Claude/Codex/Gemini)가 변증법 구조로 설계를 공격·반박·판정하여, 단독 설계에서 발견 불가능한 구조적 결함을 체계적으로 드러내는 시스템.

---

## 수치로 보는 성과

| 지표 | 값 |
|------|-----|
| 총 실험 | 4회 (RAG 설계, 전광판 동기화, RAG 구현, 호스팅/LLM) |
| 총 라운드 | 35+ R |
| 총 모순 제기 | 90+ 건 |
| VALID 비율 | 96%+ |
| 3모델 vs 단일모델 발견량 | +263% |
| 설계 변경 | 파이프라인 3회, 청크 ID 7회, 캐시 키 5회 |
| 최종 프로덕트 | 면접위키 RAG → Cloudflare Pages 배포 |

---

## 실험 타임라인

### 1차: RAG 시스템 설계 검증 (2026-04-06)
- **모델**: Claude(Executor) + Codex(Challenger) + Gemini(Arbiter)
- **소주제**: 청크 분할, 검색-생성 아키텍처, 캐싱/인덱싱
- **결과**: 21R, 47건 모순, VALID 98%
- **핵심 발견**: 청크 ID 7번 재설계, 인용 정합성 span 3단계 폴백, 3계층 캐시 설계
- **상세**: [docs/experiments/2026-04-06-RAG-시스템-검증/](experiments/2026-04-06-RAG-시스템-검증/)

### 2차: 전광판 영상 동기화 검증 (2026-04-07)
- **소주제**: 실시간 동기화, 오차 측정, UDP 전환
- **결과**: 7R, 42건 모순, VALID 81%, Executor Collapse 감지
- **상세**: [docs/experiments/2026-04-07-전광판-영상싱크/](experiments/2026-04-07-전광판-영상싱크/)

### 3차: RAG 구현 검증 — 3모델 정식 (2026-04-08)
- **모델**: Claude(Executor) + Codex GPT-5.4(Challenger) + Gemini 3 Pro(Arbiter)
- **소주제**: 아키텍처 정합성 + 비용 제로 배포
- **결과**: 7R, 40건 모순, VALID 97.5%
- **핵심 발견**: 
  - 단일모델(11건) 대비 263% 더 많은 결함 발견
  - match_chunks 삭제, citation 비강제, 라우터 SPOF, 캐시 키 정합성 등 14건 신규 발견
  - P0~P2 의존성 그래프 확정
- **상세**: [docs/experiments/2026-04-08-RAG-구현-검증/](experiments/2026-04-08-RAG-구현-검증/)

### 4차: 배포 전략 + LLM 선택 검증 (2026-04-09)
- 무료 호스팅 8개 서비스 검증 → 전부 탈락 → Cloudflare Workers AI 확정
- 무료 LLM API 6개 검증 → Cloudflare Workers AI (335회/일) 최적
- 토큰 절감 6가지 전략 우선순위 확정
- Ollama vs vLLM vs llama.cpp vs LocalAI 런타임 비교

---

## 변증법이 바꾼 것

| 변증법 없었으면 | 변증법으로 |
|----------------|-----------|
| 차원 1536으로 배포 → 런타임 크래시 | SQL/Config 384 통일 |
| citation 무시하고 캐시 → 환각 유통 | 재생성/거절 로직 강제 |
| Ollama 자체 호스팅 고집 → 배포 불가 | Cloudflare Workers AI |
| Gemini API 소진 → 서비스 중단 | 면접위키 실시간 연동 + 토큰 절감 |
| 3-tier 유지보수 → 조합 폭발 | 단일 경로 Cloudflare 올인원 |
| 캐시 키 query만 → 잘못된 응답 | complexity/model/설정 포함 |

---

## 시스템 아키텍처

### 적대적 검증 시스템
```
Executor (Claude) — 설계 제안 + 반박 대응
    ↓ 테제
Challenger (Codex GPT-5.4) — 코드 직접 읽기 + 모순 탐색
    ↓ 안티테제
Arbiter (Gemini 3 Pro) — 대칭 판정 (VALID/WEAK/INVALID)
    ↓ 진테제
→ 수렴 시 conditional_consensus로 종료
```

### 면접위키 RAG (최종 프로덕트)
```
사용자 → Cloudflare Pages (Next.js)
  → API Route
    → 면접위키 Supabase 실시간 검색 (18토픽)
    → 섹션 추출 (800자, 토큰 절감)
    → Cloudflare Workers AI (Llama 3.1 8B)
    → 스트리밍 응답
```

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| 검증 시스템 | Claude Opus 4.6 + Codex GPT-5.4 + Gemini 3 Pro |
| 프론트엔드 | Next.js 16, Tailwind CSS, TypeScript |
| 백엔드 | Cloudflare Workers AI, Pages |
| LLM | Llama 3.1 8B Instruct (Cloudflare) |
| 데이터 | 면접위키 Supabase (실시간 연동) |
| 검색 | 키워드 매칭 + 섹션 추출 (토큰 절감) |

---

## 포트폴리오 가치

### 이 프로젝트가 보여주는 역량
1. **시스템 설계** — RAG 파이프라인 전체를 설계하고 검증
2. **비판적 사고** — 자기 설계를 적대적으로 검증하는 구조 설계
3. **멀티모델 오케스트레이션** — 3개 AI를 역할 분담하여 조율
4. **프로덕트 완성** — 설계→검증→구현→배포 전체 사이클
5. **비용 최적화** — $0 무료 배포, 토큰 절감 전략

### 핵심 메시지
> "AI를 잘 쓰는 것이 아니라, AI 결과를 신뢰 가능하게 만드는 시스템을 설계했습니다."
