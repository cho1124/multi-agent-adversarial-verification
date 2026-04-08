# Citation Validation 비강제

> 소주제1 R1 — C6, VALID, **3모델만 발견**

## 문제

`citation.py`의 `validate_citations()`가 invalid_citations, uncited_sentences, coverage를 계산하지만, `workflow.py`의 search_node는 이 결과를 보고 **재생성/거절/강등을 하지 않고 그대로 캐시하여 반환**한다.

## 단일모델에서는

미발견. "Sufficiency Gate 단일값"(C7)만 지적하고, citation 후처리 경로는 추적하지 않음.

## 3모델에서 발견된 과정

Codex가 두 파일을 교차 분석:
- `citation.py:54-106` — validate 결과 계산
- `citation.py:138-171` — process_response: 결과를 붙여서 반환만
- `workflow.py:220-241` — 검증 결과와 무관하게 캐시 저장 + 반환

"검증은 있다"가 아니라 **"로그로 남는다" 수준**이라는 Codex의 정확한 표현.

## 영향

citation coverage가 0%여도 답변이 그대로 사용자에게 전달. "환각 방지" 주장의 근본적 무력화.

## 수정

workflow.py search_node에 citation 검증 후 제어 로직 추가:
- coverage < 0.3 → 재생성 (최대 1회 재시도)
- invalid_citations > 0 → 해당 인용 제거 후 반환
- uncited_sentences > 50% → 경고 prefix 추가
