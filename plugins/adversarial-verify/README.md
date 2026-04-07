# Adversarial Verify Plugin

멀티 에이전트 적대적 검증 시스템 — 3개 모델이 설계를 적대적으로 검증합니다.

## 설치

```bash
# 1. 마켓플레이스 추가
/plugin marketplace add cho1124/multi-agent-adversarial-verification

# 2. 플러그인 설치
/plugin install adversarial-verify@adversarial-verification

# 3. (선택) 플러그인 새로고침
/reload-plugins
```

## 필수 의존성

| 도구 | 설치 | 역할 |
|------|------|------|
| Codex CLI | `npm install -g @openai/codex` | Challenger (기본) |
| Gemini CLI | `npm install -g @google/gemini-cli` | Arbiter (기본) |

Codex/Gemini 없이도 Claude만으로 2모델 구조 사용 가능:
```
/adversarial-verify "주제" --challenger=claude --arbiter=claude
```

## 사용법

```bash
# 기본 (Claude=Executor, Codex=Challenger, Gemini=Arbiter)
/adversarial-verify "실시간 채팅 메시지 저장 전략"

# 역할 커스텀
/adversarial-verify "DB 스키마 설계" --executor=codex --challenger=claude --arbiter=gemini

# v1 placebo 모드 (비교 실험용, 프레임 전환 검증 없음)
/adversarial-verify "API 인증" --v1

# 최대 라운드 지정
/adversarial-verify "캐싱 전략" --rounds=20
```

## 포함된 에이전트

| 에이전트 | 역할 |
|---------|------|
| `executor` | 설계 제안 + 반박 대응 |
| `challenger` | 모순 탐색 + Collapse 판별 기준 + 프레임 전환 검증 |
| `arbiter` | 대칭적 검증 + VALID/WEAK/INVALID 판정 + 프로세스 주도 |
| `orchestrator` | Phase 0~3 오케스트레이션 프로토콜 |

## 핵심 기능

- **프레임 전환 검증**: 논점 돌리기/논점 흐리기를 구조적으로 차단
- **독립 체크리스트**: Challenger가 제안 보기 전에 필수 항목 생성 (앵커링 방지)
- **Collapse 감지**: 4단계 조작적 정의 (SD/RC/CA/CS)
- **대칭적 검증**: Arbiter가 Challenger 반박 + Executor 대응 모두 평가
- **동적 종료**: Collapse 감지 시 즉시 종료, 합의 도달 시 종료, 최대 50라운드
