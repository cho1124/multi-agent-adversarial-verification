# Agentic Loop 1회 왕복 제한

> 소주제1 R1 C3 + R3 C21, VALID, **3모델만 발견**

## 문제

두 가지 경로 모두 진정한 agentic loop이 아님:

1. **search_node (C3)**: `hybrid_search`를 직접 호출하고, `create_search_agent()`로 tool-bound LLM을 만들지만 `tool_calls`를 처리하지 않음. 생성기에 가까움.

2. **_run_agent (C21)**: quiz/explain/compare용 공통 함수. 첫 invoke → tool_calls 1번 → 두 번째 invoke → 종료. `while response.tool_calls` 루프가 아님.

## 단일모델에서는

미발견. "도구 패턴 불균일" 정도의 표면적 지적.

## 3모델에서 발견된 과정

- R1 C3: Codex가 `workflow.py:126-143`(직접 hybrid_search)과 `workflow.py:193-214`(tool-bound LLM)를 분석, tool_calls 미처리 확인
- R3 C21: Executor가 "search만 P1"로 분류하자, Codex가 `_run_agent()`도 1회 왕복임을 지적 → 공통 런타임 결함으로 격상

## 영향

- search: 도구가 바인딩되어 있지만 사용하지 않음 (장식)
- quiz/explain/compare: 2라운드 이상의 도구 사용 불가 (복잡한 질문 처리 제한)

## 수정 (P1-Phase C)

```python
def _run_agent(state, agent_name, tools, llm):
    messages = [system_msg, HumanMessage(query)]
    tool_map = {t.name: t for t in tools}
    
    while True:  # while loop으로 전환
        response = llm.invoke(messages)
        if not response.tool_calls:
            break
        for tc in response.tool_calls:
            result = tool_map[tc["name"]].invoke(tc["args"])
            messages.append(ToolMessage(content=str(result), tool_call_id=tc["id"]))
    
    return response.content
```

search_node도 동일 패턴으로 통일하여, Phase C 조건(search + 공통 경로 모두 포함)을 충족.
