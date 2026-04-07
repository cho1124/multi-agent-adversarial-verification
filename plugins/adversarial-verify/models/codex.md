---
id: codex
name: Codex GPT-5.2
type: cli
command: 'node "C:/Users/WINTEK/.claude/plugins/marketplaces/openai-codex/plugins/codex/scripts/codex-companion.mjs" task --fresh'
---

# 호출 방법

codex-plugin-cc의 codex-companion 스크립트를 통해 호출.

```bash
node "<codex-companion 경로>" task --fresh "<프롬프트>"
```

## 필수 설치

```bash
npm install -g @openai/codex
```

그리고 Claude Code에서:
```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
```

## 인증

```bash
codex login
```

또는 환경변수: `OPENAI_API_KEY`

## 참고

- 매 호출 시 `--fresh` 플래그로 새 스레드 생성 (이전 컨텍스트 미유지)
- codex:codex-rescue 서브에이전트로도 호출 가능
- 출력은 stdout으로 반환
