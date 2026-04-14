# 05. Unity 에디터 연동 강화

## 문제 (초기 검증 H-03, CK-26)

1. SpriteSheetPostprocessor.cs의 수동 JSON 파서(string indexOf)가 취약 — 음수값, 공백 변동, 중첩 키에서 오파싱
2. 슬라이싱 좌표가 padding을 미고려 — padding > 0인 시트에서 프레임 위치 어긋남

## 최종 해소

### JSON 파서 교체
- `JsonConvert.DeserializeObject<SpriteSheetMetadata>()` (Newtonsoft.Json)
- `[JsonProperty]` 어트리뷰트로 Python 메타데이터 키명과 정확히 매핑
- `Dictionary<string, AnimationDef>` 역직렬화 지원 (Unity JsonUtility 불가 → Newtonsoft 필요)

### Padding 좌표 반영
- `cellW = FrameWidth + pad * 2`, `cellH = FrameHeight + pad * 2`
- `x = col * cellW + pad`
- `y = textureHeight - (row * cellH + pad + FrameHeight)`

## 개선된 점

- JSON 파싱 안정성 대폭 향상
- padding 유/무 모두 정확한 슬라이싱
- Python 메타데이터에 `padding` 필드 추가 → Unity 측 자동 인식
