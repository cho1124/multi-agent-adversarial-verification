# 04. Undo 안전 임포트

## 문제 (R8 C8-3, R10 C10-4, R12 C12-3)

Timer 콜백은 Operator 컨텍스트 외부 → Undo push 미보장. bpy.data 직접 조작은 Undo와 충돌. OBJ 임포터는 Blender 4.0에서 변경.

## 최종 해소

- 자동 임포트(bpy.data 직접) 옵션 완전 제거
- 모든 임포트: Operator.execute() → bpy.ops.import_scene → Undo 보장
- 임포터 레지스트리: 런타임 hasattr 체크로 OBJ/FBX operator 감지
- 자동 오프셋: X축 바운딩박스+1m 간격
