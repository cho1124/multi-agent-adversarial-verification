# 라우터 Single Point of Failure

> 소주���1 R2 C11 + R3 C16, VALID, **3모델만 발견**

## 문제

라우터가 그래프의 유일한 진입점인데:
1. **C11**: `JsonOutputParser`에 예외처리/fallback 없음 → LLM이 JSON을 어기면 전체 그래프 중단
2. **C16**: 파싱 성공 후 semantic validation 없음 → `agent_type="foo"` → 분기 맵 KeyError

## 단일모델에서는

미발견. 라우터 존재 자체는 인지했지만 예외처리 부재와 semantic validation 부재는 지적하지 않음.

## 3모델에서 발견된 과정

- R2: Codex가 `router.py:46-47`의 `JsonOutputParser` 직접 호출을 확인, try-except 부재 지적
- R3: Executor가 "파싱 fallback 추가"로 대응하자, Codex가 "파싱은 통과해도 agent_type 값 검증이 없다"고 재반박

## 수정

```python
# router.py
VALID_AGENTS = {"search", "quiz", "explain", "compare"}

try:
    result = parser.invoke(response)
except Exception:
    result = {"agent_type": "search", "complexity": "light", "reason": "fallback"}

if result.get("agent_type") not in VALID_AGENTS:
    result["agent_type"] = "search"  # 안전한 기본값
```
