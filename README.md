# Multi-Agent Adversarial Verification

멀티 에이전트 적대적 검증 시스템 — 변증법 기반 AI 에이전트 오케스트레이션

## 이게 뭔가요?

AI한테 "이거 설계해줘"라고 하면 그럴듯한 답이 나옵니다. 근데 **그게 정말 괜찮은 건지** 어떻게 확인하나요?

이 시스템은 **AI 3명이 서로 토론**하게 만들어서 설계의 구멍을 찾습니다:

```
AI 1 (제안자): "이렇게 만들면 됩니다"
AI 2 (반박자): "그런데 이 부분이 모순이에요"
AI 3 (중재자): "반박이 타당한지 판정합니다"
```

혼자 설계하면 "이렇게 하면 되겠지"하고 넘어가는 부분들이, 반박하는 AI가 있으면 전부 드러납니다.

## 왜 필요한가요?

AI가 코드를 빠르게 짜주는 시대에, **사람의 가치는 "빨리 짜는 것"이 아니라 "제대로 판단하는 것"**으로 이동하고 있습니다.

이 시스템은 AI가 만든 시간을 **더 많이 만드는 데** 쓰는 게 아니라, **더 좋게 만드는 데** 쓰게 해줍니다.

실제로 이 시스템으로 RAG(문서 기반 AI 질의응답) 설계를 검증했더니:
- 파이프라인이 **3번 재설계**됨 (혼자 했으면 초기안 그대로)
- 핵심 데이터 구조가 **7번 변경**됨 (혼자 했으면 발견 자체가 불가능)
- 논점 돌리기를 **자동으로 감지**하여 차단

## 개요 (기술적)

AI 에이전트가 설계/구현 제안을 할 때, **안티테제(반박)를 구조적으로 강제**하여 결과물의 완성도를 높이는 시스템입니다.

쉽게 말하면, 사람 간 토론에서의 **논점 돌리기/논점 흐리기를 구조적으로 차단**하는 규칙을 AI 에이전트에 적용한 것입니다.

## 설치

### Claude Code 플러그인으로 설치 (권장)

```bash
# 1. 마켓플레이스 추가
/plugin marketplace add cho1124/multi-agent-adversarial-verification

# 2. 플러그인 설치
/plugin install adversarial-verify@adversarial-verification

# 3. 플러그인 새로고침
/reload-plugins
```

### 의존성 (선택)

풀 3모델 구조를 사용하려면:

```bash
# Challenger용 — Codex CLI + 플러그인
npm install -g @openai/codex
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex

# Arbiter용 — Gemini CLI
npm install -g @google/gemini-cli
gemini   # 브라우저에서 Google 로그인
```

Codex/Gemini 없이도 **Claude만으로 사용 가능**:

```bash
/adversarial-verify "주제" --challenger=claude --arbiter=claude
```

## 사용법

```bash
# 기본 (Claude=Executor, Codex=Challenger, Gemini=Arbiter)
/adversarial-verify "실시간 채팅 메시지 저장 전략"

# 역할 자유 배치
/adversarial-verify "DB 스키마" --executor=codex --challenger=claude --arbiter=gemini

# 커스텀 모델 사용 (models/ 폴더에 어댑터 추가)
/adversarial-verify "API 설계" --challenger=ollama-llama --arbiter=claude

# v1 placebo (비교 실험용)
/adversarial-verify "캐싱 전략" --v1

# 최대 라운드 지정
/adversarial-verify "보안 아키텍처" --rounds=20
```

## 커스텀 모델 추가

`plugins/adversarial-verify/models/` 폴더에 어댑터 파일을 추가하면 어떤 모델이든 사용 가능:

```markdown
---
id: ollama-llama
name: Ollama Llama 3.1
type: cli
command: "ollama run llama3.1"
---

# 호출 방법
echo "<프롬프트>" | ollama run llama3.1
```

지원 타입: `session`(Claude 직접), `cli`(터미널 명령어), `api`(HTTP), `mcp`(MCP 서버)

상세: [모델 어댑터 가이드](plugins/adversarial-verify/models/README.md)

---

## 문서

### 설계

- **[멀티 에이전트 적대적 검증 시스템 설계](docs/멀티%20에이전트%20적대적%20검증%20시스템%20설계.md)** — 이론, 설계 원칙, 실험 결과, 메타 검증, 남은 과제

### 실험 기록

- **[RAG 시스템 적대적 검증 (2026-04-06)](docs/experiments/2026-04-06-RAG-시스템-검증/)** — 3소주제, 21라운드, 47건 모순, VALID 98%
  - [청크 ID 안정성](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/01-청크ID-안정성.md) — 7번 재설계
  - [인용 정합성](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/02-인용-정합성.md) — span 3단계 폴백 + 2단계 IoU
  - [캐시 무효화](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/03-캐시-무효화.md) — 3계층 동시 무효화
  - [환각 방지](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/04-환각-방지.md) — sufficiency gate
  - [스트리밍 + 인용](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/05-스트리밍-인용.md) — 문장 버퍼 SSE
  - [레이아웃 보존](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/06-레이아웃-보존.md) — 텍스트+레이아웃 동시 추출
  - [비용 통제](docs/experiments/2026-04-06-RAG-시스템-검증/improvements/07-비용-통제.md) — 로컬 우선 ~$15/월

- **[전광판 영상 동기화 검증 (2026-04-07)](docs/experiments/2026-04-07-전광판-영상싱크/)** — 7라운드, 42건 모순, VALID 81%, Executor Collapse 감지
  - [단계별 계측](docs/experiments/2026-04-07-전광판-영상싱크/improvements/01-단계별-계측.md) — 290ms 고정 → 5단계 분해
  - [상태머신](docs/experiments/2026-04-07-전광판-영상싱크/improvements/02-상태머신.md) — SYNCED/DRIFTING/DESYNCED 3단계
  - [오차 원인분리](docs/experiments/2026-04-07-전광판-영상싱크/improvements/03-오차-원인분리.md) — 자기상관 분석
  - [UDP 전환](docs/experiments/2026-04-07-전광판-영상싱크/improvements/04-UDP-전환.md) — TCP → UDP 멀티캐스트
  - [자동화](docs/experiments/2026-04-07-전광판-영상싱크/improvements/05-자동화.md) — 수동 파라미터 → 적응형

### 에이전트 정의

- [Executor (테제)](agents/executor.md) — 설계 제안 + 반박 대응
- [Challenger (안티테제)](agents/challenger.md) — 모순 탐색 + Collapse 판별 기준
- [Arbiter (중재)](agents/arbiter.md) — 대칭적 검증 + VALID/WEAK/INVALID 판정
- [Orchestrator (오케스트레이션)](agents/orchestrator.md) — Phase 0~3, 동적 종료

### 모델 어댑터

- [Claude](plugins/adversarial-verify/models/claude.md) — 메인 세션 (기본 Executor)
- [Codex](plugins/adversarial-verify/models/codex.md) — GPT-5.2 CLI (기본 Challenger)
- [Gemini](plugins/adversarial-verify/models/gemini.md) — Gemini 3 Pro CLI (기본 Arbiter)
- [커스텀 모델 추가 가이드](plugins/adversarial-verify/models/README.md)

### 재현성

- [재현성 패키지](experiment/README.md) — 파라미터, 프롬프트 템플릿, 측정 지표
- [실험/문서 작성 가이드](docs/CONTRIBUTING.md) — 새 실험 추가 시 따라야 할 규칙

## 라이선스

MIT License
