# 01. Timer 아키텍처 재설계

## 문제 (R1 C-01, R4 C-25, R5 C-30)

초기 설계에서 Thread 내부에서 bpy API를 호출할 수 있다는 암묵적 전제. Blender의 메인 스레드 전용 API 정책과 충돌. R4에서 on-demand Timer를 제안했으나 `bpy.app.timers.register()`도 메인 스레드 전용으로 무효화.

## 논의 과정 (6라운드에 걸친 점진적 개선)

| 라운드 | 제안 | 반박 | 결과 |
|--------|------|------|------|
| R1 | Thread + Timer (경계 미정의) | bpy 메인 스레드 전용, segfault 위험 | Thread→Queue→Timer→bpy 패턴 |
| R2 | Timer 실행 주기 미정의 | burst 시 대량 bpy 호출 | 최대 1건/tick |
| R4 | on-demand Timer | **register() 메인 스레드 전용** | **Frame Shift: 상시 Timer** |
| R5 | 상시 Timer 500ms | Queue 배압 없음 | 2개 Queue 분리 + 용량 제한 |
| R6 | Queue(maxsize=10) put() | 메인 스레드 블로킹 | put_nowait + 용량 제한 |
| R8 | Timer 500ms 고정 | 재진입 위험 | _is_processing 플래그 |

## 최종 해소

- 2개 Queue: request_queue(maxsize=5, put_nowait) + result_queue(무제한)
- Worker Thread: 순수 네트워크 I/O만. bpy 호출 완전 차단
- 상시 Timer 500ms: 재진입 방지(_is_processing), 최대 1건/tick
- 30초 주기 유지보수: time.monotonic() 기반, 최대 5건 아카이브
