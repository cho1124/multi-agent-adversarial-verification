---
id: gemini
name: Gemini 3 Pro
type: cli
command: "gemini -p"
---

# 호출 방법

gemini-cli를 통해 단발 프롬프트 모드로 호출.

```bash
gemini -p "<프롬프트>"
```

## 필수 설치

```bash
npm install -g @google/gemini-cli
```

## 인증

```bash
# 방법 A: Google 계정 로그인 (권장)
gemini   # 브라우저에서 Google 로그인

# 방법 B: API 키
export GEMINI_API_KEY=your-key
```

## 참고

- 단발 프롬프트 모드 (`-p`) 사용 — 대화 히스토리 미유지
- 무료 티어: 60 req/min, 1000 req/day
- stdout으로 응답 반환
- 타임아웃 120초 권장
