# Multi-Agent Adversarial Verification

멀티 에이전트 적대적 검증 시스템 — 변증법 기반 AI 에이전트 오케스트레이션

## 개요

AI 에이전트가 설계/구현 제안을 할 때, **안티테제(반박)를 구조적으로 강제**하여 결과물의 완성도를 높이는 시스템입니다.

핵심 전제: **안티테제가 강력할수록 테제의 논리도 단단해진다.**

## 구조

```
[자율 운영 라운드]

Executor ⇄ Challenger
        ↕
    Arbiter (AI)
    - 서기: 공방 기록 정리
    - 임시 중재: 논점 정리, 방향 제시
    - 감지: 공방 품질 모니터링

        ↓ 리포트 (요약본 + 중재 이력)

[최종: 사람 개입]

    사람 (최종 결정권자)
    - Arbiter 요약본 기반 결정
    - 교착 상태 또는 프로젝트 급변 시에만 개입
```

## 핵심 발견: 프레임 전환 검증

Challenger에 아래 규칙 **1줄을 추가**하는 것만으로 Disagreement Collapse를 구조적으로 방지할 수 있습니다:

> Executor가 미해소 모순을 '제약조건', '수용 가능한 리스크', '본질적 한계' 등으로 재정의할 경우, 그 재정의의 근거와 권한이 타당한지 검증한다. 문제의 프레임을 바꾸는 것은 해소가 아니다.

쉽게 말하면, 사람 간 토론에서의 **논점 돌리기/논점 흐리기를 구조적으로 차단**하는 규칙입니다.

## 실험 결과

6회의 실증 실험을 통해 검증:

| 실험 | 주제 | v1 (규칙 없음) | v2 (프레임 전환 검증) |
|------|------|---------------|---------------------|
| 원본 | LangChain 최적화 | Collapse (R5~7) | 없음 |
| A1 | DB 스키마 설계 | Collapse (R4) | 없음 |
| A2 | API 인증 아키텍처 | Collapse (R1~7) | 없음 |
| A3 | UI 상태관리 | Collapse (R1~7) | 없음 |
| B | 통제 실험 (동일 입력) | 부분적 Collapse | 없음 |
| C | 역할 역전 (Codex=Executor) | - | 없음 |

- **v1 Collapse 재현율: 100% (5/5)**
- **v2 Collapse 발생율: 0% (0/6)**
- **Executor 편향 배제**: 통제 실험(동일 입력)으로 확인
- **모델 역할 무관**: 역할 역전에서도 유효

## 파일 구조

```
agents/
  executor.md      # Executor (테제) 에이전트 정의
  challenger.md    # Challenger (안티테제) 에이전트 정의
  arbiter.md       # Arbiter (중재) 에이전트 정의

docs/
  멀티 에이전트 적대적 검증 시스템 설계.md   # 전체 이론 + 실험 결과
```

## 설계 원칙

1. **포맷 강제** — 행동을 금지하는 것이 아니라, 행동할 수 있는 틀 자체를 제한
2. **모델 다양성** — 같은 편향 공유 방지를 위해 서로 다른 모델 사용
3. **이중 방어** — 구조적 예방(포맷, 역할 제한, 라운드 제한) + 감지(품질 모니터링)
4. **사람 개입 최소화** — 교착/품질 저하/프로젝트 급변 시에만

## 실험 환경

- **Executor**: Claude Opus 4.6
- **Challenger**: Codex GPT-5.2 (codex-plugin-cc)
- **Arbiter**: Gemini 3 Pro (미연결, 추후 통합 예정)

## 참고 연구

- [Peacemaker or Troublemaker: How Sycophancy Shapes Multi-Agent Debate](https://arxiv.org/html/2509.23055v1)
- [Mitsubishi Electric: Multi-agent AI for Adversarial Debate](https://us.mitsubishielectric.com/en/pr/global/2026/0120/)
- [Devil's Advocacy and Dialectical Inquiry](https://www.nationalforum.com/Electronic%20Journal%20Volumes/Lunenburg,%20Fred%20C.%20Devil's%20Advocacy%20&%20Dialectical%20Inquiry%20IJSAID%20V14%20N1%202012.pdf)
- [Talk Isn't Always Cheap: Failure Modes in Multi-Agent Debate](https://arxiv.org/pdf/2509.05396)

## 라이선스

MIT License
