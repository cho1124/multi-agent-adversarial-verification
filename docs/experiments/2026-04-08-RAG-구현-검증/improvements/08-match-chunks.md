# match_chunks RPC 삭제 미복원

> 소주제1 R1 — C2, VALID, **3모델만 발견**

## 문제

`setup_v3.sql`이 기��� `match_chunks` 함수를 DROP하지만 재생성하지 않는다. `vector_search.py`는 여전히 `match_chunks` RPC를 호출하므로, v3 스키마 기준으로 quiz/explain/compare 에이전트가 깨진다.

## 단일모델에서는

미발견. 차원 불일치(C1)만 지적하고, SQL 내부의 DROP/CREATE 불일치는 놓침.

## 3모델에서 발견된 과정

Codex(GPT-5.4)가 `setup_v3.sql`을 직접 읽고:
- 7~11행: `DROP FUNCTION IF EXISTS match_chunks` 확인
- 전체 파일에서 `CREATE FUNCTION match_chunks` 부재 확인
- `vector_search.py:27-35`의 RPC 호출과 교차 대조

## 영향

quiz/explain/compare ��이전트가 `vector_search` 도구를 바인딩하므로, v3 배포 시 이 3개 경로 전부 실패. search만 hybrid_search를 직접 호출하여 우회.

## 수정

`setup_v3.sql`에 `match_chunks_hybrid` 또는 `match_chunks` 함수를 v3 스키마에 맞게 재생성. 또는 vector_search.py가 v3의 `match_chunks_hybrid`를 호출하도록 변경.
