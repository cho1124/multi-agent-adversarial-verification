# 03. 공유 팔레트 — 프레임 간 색상 일관성

## 문제 (초기 검증 H-01)

각 프레임을 독립적으로 팔레트 양자화하여 같은 애니메이션 내에서 프레임마다 서로 다른 팔레트 → 재생 시 색상 플리커링. 픽셀 아트에서 치명적.

## 논의 과정

| 라운드 | 제안/반박 | 결과 |
|--------|----------|------|
| 초기 | build_shared_palette + apply_shared_palette 구현 | 전체 프레임 → 단일 팔레트 |
| R1 C-04 | fallback build_shared_palette가 입력 무시 | Executor 코드 오독 확인 → 클램핑 추가 |
| 테스트 | putdata TypeError (list[list] vs tuple) | tuple 변환 수정 |

## 최종 해소

- `build_shared_palette()`: 모든 프레임의 불투명 픽셀 수집 → strip 이미지 → MEDIANCUT 양자화
- `apply_shared_palette()`: 공유 팔레트를 각 프레임에 적용, 알파 채널 보존
- `num_colors` 클램핑: `max(2, min(256))`
- 전체 투명 프레임 시 더미 팔레트: `Image.new("RGB").quantize(colors=2)`

## 개선된 점

- 애니메이션 전체에서 동일 색상 팔레트 사용 → 플리커링 제거
- 테스트: 공유 팔레트 일관성 + 알파 보존 자동 검증
