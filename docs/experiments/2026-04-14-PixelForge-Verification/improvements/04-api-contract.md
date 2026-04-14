# 04. API 타입 계약 정직성

## 문제 (Meta 검증 M-03)

`generate_frames(frame_size: int | tuple[int, int])` 시그니처가 tuple을 명시적으로 허용하면서, 내부에서 `frame_size[0]`만 취하고 height를 폐기. 비정사각형 (64, 128) 호출 시 묵시적으로 64x64 생성.

## 논의 과정

| 라운드 | 제안/반박 | 결과 |
|--------|----------|------|
| R1 C-01 | non-square 불가능하면서 tuple 수용은 기만 | WEAK — UI가 정사각형만 제공 |
| Meta M-03 | UI 우회가 아니라 public API 계약 위반 | **VALID — 수정 필요** |

## 최종 해소

- 비정사각형 tuple 입력 시 **명시적 ValueError** → 묵시적 데이터 손실 방지
- 에러 메시지에 제약 사유 명시 ("ComfyUI generates square frames only")
- 테스트: `test_non_square_frame_size_raises` 자동 검증

## 개선된 점

- API가 지원하지 않는 입력을 조용히 잘라내지 않고 명확히 거부
