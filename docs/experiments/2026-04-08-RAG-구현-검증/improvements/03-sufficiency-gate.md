# Sufficiency Gate 단일값 문제

> 소주제1 (아키텍처 검증) — R1에서 발견, HIGH

## 문제

Sufficiency Gate가 `max(final_score)` 하나만 체크하여, 1개 고점수 + 4개 0점인 경우에도 PASS 판정. 설계 의도("근거 충분성 검증")와 구현("최고점 통과")이 괴리.

## 1차 설계에서는

2단계 임계값 + 분포 고려로 확정:
- score < low → 거절
- low ≤ score < high → 제한 답변
- score ≥ high → 정상

## 실제 구현

```python
# tools/sufficiency_gate.py
top_score = max(chunk["final_score"] for chunk in chunks)
if top_score < low_threshold:    # 0.5
    return "REJECT"
elif top_score < high_threshold:  # 0.7
    return "LIMITED"
else:
    return "PASS"
```

2단계 임계값 자체는 구현되었으나, **max만 보는 것**이 문제. 분포를 전혀 고려하지 않음.

## 적대적 검증에서 일어난 일

**R1 — Challenger 시나리오:**

```
청크 5개 점수: [0.85, 0.12, 0.08, 0.05, 0.03]
max = 0.85 → PASS

실제: 근거가 1개뿐. 다각도 답변 불가. PASS가 아니라 LIMITED가 적절.
```

**R2 — Executor "복합 판정" 제안에 Challenger 재반박:**

"median + count 복합 판정을 도입하면 임계값이 3개로 증가. A/B 테스트 프레임워크 없이 최적값을 어떻게 찾는가?"

**R3 — Executor 최종 대응:**

A/B 없이도 보수적 조건 추가로 개선 가능:
```python
passing_chunks = [c for c in chunks if c["final_score"] >= low_threshold]
if len(passing_chunks) < 2:
    return "LIMITED"  # 근거 1개만으로는 제한 답변
```

Arbiter: VALID — 완벽하지 않아도 현재(max만)보다는 확실히 개선

## 개선된 점

기존 max 판정에 **보조 조건** 추가:
1. `count(score >= low_threshold) >= 2` — 근거 최소 2개
2. 향후 A/B 프레임워크 도입 시 median 등 추가 가능

설계의 "2단계 임계값"은 유지하되, 분포 검증을 점진적으로 보강.
