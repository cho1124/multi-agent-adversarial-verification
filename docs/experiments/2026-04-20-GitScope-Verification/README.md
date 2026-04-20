# GitScope 설계 검증

Code Forensics 시각화를 내장한 Git GUI 프로젝트의 설계를 3개 모델(Claude/Codex)이 Executor-Challenger-Arbiter 역할로 검증. 설계 검증 3라운드 + 구현 검증 1라운드 수행.

## 결론

**"파일/클래스 단위 히스토리 추적 Git GUI"라는 원안은 3라운드의 적대적 검증을 거치며 "Code Forensics 시각화를 내장한 경량 Git GUI"로 정제되었다.**

### 결과 요약

| 항목 | 값 |
|------|-----|
| 설계 검증 라운드 | 3 |
| 구현 검증 라운드 | 1 |
| 총 모순 제기 | 22건 (설계 15건 + 구현 7건) |
| 해소 | 19건 |
| 미해소 | 1건 |
| VALID | 7건 (47%) |
| WEAK | 6건 (40%) |
| INVALID | 2건 (13%) |
| Collapse | 없음 |
| Frame Shift | R1, R2 발생 → R3에서 LEGITIMATE 판정 |

### 설계 검증 단계별 발견

| 라운드 | 핵심 발견 | Challenger 상태 |
|--------|----------|----------------|
| R1 | MVP 스코프 과다, git2-rs/CLI 구조 모순, 독립 앱 vs "한 번의 클릭" 모순 | rational_consensus |
| R2 | GitLens 직접 경쟁, 심볼 추적 한계 vs 가치 제안 모순, 기존 도구 대체 가능 | — |
| R3 | CodeScene 커뮤니티 종료 확인 → 무료 대체재 부재, PMF 사전 검증 한계 | EXHAUSTED |

### 구현 검증 발견

| ID | 유형 | 심각도 | 내용 | 조치 |
|----|------|--------|------|------|
| I-01 | Security | High | 경로 검증 없이 사용자 입력 경로 사용 | `validatePath()` 추가 |
| I-03 | Performance | High | `readdirSync` 동기 파일트리 → 대규모 레포 블로킹 | 비동기 + 깊이 제한(5) |
| I-05 | Data | Medium | `\|` 구분자 → 작성자명에 `\|` 포함 시 파싱 오류 | `\x1f` (Unit Separator) 변경 |
| I-06 | Data | Medium | 동명이인 기여자 병합 | 이메일 기반 키로 변경 |
| I-07 | Concurrency | High | 글로벌 `gitService` 경합 | 1인 사용 전제로 수용 |

### 검증된 것

- 기술 스택 선택 (Express + React + simple-git) 타당성
- Code Forensics가 기존 무료 Git GUI에 없는 차별점임 (CodeScene 유료 전환 확인)
- git log --numstat 파싱 기반 히트맵/핫스팟의 데이터 신뢰성 (파일 단위)
- MVP 스코프 (기본 Git + Forensics 대시보드) 현실성
- GIT_OPTIONAL_LOCKS=0 + 읽기 전용 접근으로 lock 충돌 회피 가능

### 검증되지 않은 것

- 10만+ 커밋 대규모 레포에서의 실제 성능 (벤치마크 미실시)
- 함수 단위 히스토리 추적의 실용성 (MVP 범위 외)
- 실제 사용자 수요 (개인 사이드 프로젝트로 수용)

## 설계 진화 과정

```
원안                              R1 수정                        R3 최종
──────────────────────────────────────────────────────────────────────────
Tauri 독립 앱                   → 하이브리드(VSCode+Tauri)     → Express+React 웹앱
심볼 단위 완벽 추적              → 리네임 3단계 중 1단계만      → 파일 단위 핵심
"한 번의 클릭으로 확인"          → "심볼 단위 인사이트"         → "Code Forensics 시각화"
MVP 3-4개월                     → 스코프 축소                  → Git 기본 + Forensics
Electron 대비 50MB              → 수치 철회                    → Express 경량 서버
```

## WEAK 항목

| ID | 내용 | 이유 |
|----|------|------|
| C-03 | 50MB 메모리 주장 근거 부재 | 수치 철회 후 상대적 비교만 유지 — 실질적 영향 없음 |
| C-07 | Progressive Disclosure 프레임 전환 | UX/기술 전략 분리로 충분히 정리 |
| C-08 | GitLens 대비 차별점 | Code Forensics(전체 레포 히트맵/트렌드)로 명시 |
| C-13 | PMF 검증 계획 구체성 부족 | 개인 프로젝트로 프레임 재설정 |
| C-15 | Git GUI + Forensics 스코프 | 기본 Git 범위 명시적 제한으로 대응 |
| I-04 | isGitRepo() 에러 구분 부재 | 개인 사용에서 실질적 영향 미미 |

## 프로젝트 정보

| 항목 | 값 |
|------|-----|
| 레포 | [cho1124/GitScope](https://github.com/cho1124/GitScope) |
| 스택 | Express + React + TypeScript + simple-git |
| Executor | Claude (Opus 4.6) |
| Challenger | Codex |
| Arbiter | Claude (별도 에이전트) |
| 검증 일자 | 2026-04-20 |
| 스킬 버전 | adversarial-verify v2 |