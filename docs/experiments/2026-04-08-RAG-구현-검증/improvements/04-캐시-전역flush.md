# 캐시 전역 flush 문제

> 소주제1 (아키텍처 검증) — R1에서 발견, MEDIUM

## 문제

1차 설계에서 "3계층 동시 무효화"를 확정했으나, 구현은 `invalidate_for_topic(topic_id)`가 내부적으로 `invalidate_all()`을 호출하여 **전체 캐시를 날림**.

## 1차 설계에서는

```
문서 갱신 시:
  L1: 해당 topic 관련 쿼리만 무효화
  L2: 해당 category의 검색 결과만 무효화
  L3: 해당 chunk_id 포함 생성 결과만 무효화
```

## 실제 구현

```python
# cache/manager.py
def invalidate_for_topic(self, topic_id: str):
    self.invalidate_all()  # 전체 날리기
```

## 적대적 검증에서 일어난 일

**R1 — Challenger:**

주제 1개 수정 → 전체 캐시 소멸 → 비용 절감 효과 상쇄. 특히 면접위키처럼 주제가 100개 이상인 경우, 1개 수정에 99개 캐시까지 날아감.

**R2 — Challenger 재반박 (핵심):**

Executor가 "topic_id 기반 필터링 구현"을 제안했으나:

```
L1 캐시 키: SHA256(query)         → query에 topic_id가 없으면 매칭 불가
L3 캐시 키: SHA256(query+chunk_ids+model) → topic → chunk 역매핑 필요
```

"필터링"이 아니라 **역인덱스 구조 변경**이 필요. 캐시 엔트리에 `topic_ids` 메타데이터를 별도 저장해야 함.

Arbiter: VALID — SHA256 키 구조상 topic 필터링 불가

## 개선된 점

캐시 구조에 메타데이터 계층 추가:

```python
# 변경안
cache_entry = {
    "key": sha256(query),
    "value": response,
    "topic_ids": {topic_id_1, topic_id_2},  # 추가
    "timestamp": now
}

def invalidate_for_topic(self, topic_id):
    for key, entry in self._cache.items():
        if topic_id in entry["topic_ids"]:
            del self._cache[key]
```

L2/L3도 동일 패턴으로 `topic_ids` 또는 `chunk_ids` 메타데이터 추가.
