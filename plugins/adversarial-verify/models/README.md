# 모델 어댑터

각 모델은 어댑터 파일로 정의됩니다. 기본 제공 3개 외에 커스텀 모델을 추가할 수 있습니다.

## 기본 제공 모델

| 모델 ID | 파일 | 호출 방식 |
|---------|------|----------|
| `claude` | [claude.md](claude.md) | 메인 세션 직접 생성 |
| `codex` | [codex.md](codex.md) | codex-plugin-cc CLI |
| `gemini` | [gemini.md](gemini.md) | gemini-cli |

## 커스텀 모델 추가

`models/` 폴더에 `<모델ID>.md` 파일을 추가하면 됩니다.

### 어댑터 파일 형식

```markdown
---
id: my-model
name: 표시 이름
type: cli | api | mcp
command: "실행 명령어 (type=cli일 때)"
---

# 호출 방법

(Claude가 이 모델을 호출할 때 따라야 할 구체적 지시)
```

### type 설명

| type | 설명 | 예시 |
|------|------|------|
| `cli` | 터미널 명령어로 호출 | gemini, codex, ollama |
| `api` | HTTP API 직접 호출 | OpenAI API, Anthropic API |
| `mcp` | MCP 서버 도구로 호출 | MCP 연결된 모델 |
| `session` | 현재 Claude 세션에서 직접 | claude (기본) |

### 예시: Ollama 로컬 모델 추가

```markdown
---
id: ollama-llama
name: Ollama Llama 3.1
type: cli
command: "ollama run llama3.1"
---

# 호출 방법

프롬프트를 echo로 파이프하여 호출:
\`\`\`bash
echo "<프롬프트>" | ollama run llama3.1
\`\`\`

출력은 stdout으로 반환됨. JSON 포맷 강제가 안 될 수 있으므로
응답에서 JSON 블록을 추출하여 파싱.
```

## 사용법

```bash
# 커스텀 모델을 Challenger로 사용
/adversarial-verify "주제" --challenger=ollama-llama

# 커스텀 모델을 Arbiter로 사용
/adversarial-verify "주제" --arbiter=my-model
```
