# Multi-Agent Adversarial Verification

멀티 에이전트 적대적 검증 시스템 — 변증법 기반 AI 에이전트 오케스트레이션

## 개요

AI 에이전트가 설계/구현 제안을 할 때, **안티테제(반박)를 구조적으로 강제**하여 결과물의 완성도를 높이는 시스템입니다.

쉽게 말하면, 사람 간 토론에서의 **논점 돌리기/논점 흐리기를 구조적으로 차단**하는 규칙을 AI 에이전트에 적용한 것입니다.

## 문서

이론, 실험, 메타 검증 결과를 포함한 전체 설계 문서:

**[멀티 에이전트 적대적 검증 시스템 설계](docs/멀티%20에이전트%20적대적%20검증%20시스템%20설계.md)**

## 파일 구조

```
agents/
  executor.md        # Executor (테제) — 설계 제안 + 반박 대응
  challenger.md      # Challenger (안티테제) — 모순 탐색 + Collapse 판별 기준
  arbiter.md         # Arbiter (중재) — 대칭적 검증 + VALID/WEAK/INVALID 판정
  orchestrator.md    # 오케스트레이션 프로토콜 (Phase 0~3, 동적 종료)

docs/
  멀티 에이전트 적대적 검증 시스템 설계.md
    → 이론 설계 + 설계 원칙 + 실험 결과 + 메타 검증 + 남은 과제

experiment/
  README.md          # 재현성 패키지 (파라미터, 프롬프트 템플릿, 측정 지표)
```

## 실험 환경

| 역할 | 모델 | 호출 |
|------|------|------|
| Executor | Claude Opus 4.6 | 메인 세션 |
| Challenger | Codex GPT-5.2 | codex-plugin-cc |
| Arbiter | Gemini 3 Pro | gemini-cli |

## 라이선스

MIT License