# 구글 뉴스 수집기 (News Collector)

Claude Code / Antigravity 스킬 — 키워드만 말하면 구글 뉴스를 자동으로 수집합니다.

## 설치

`SKILL.md` 파일을 Claude Code 스킬 폴더에 복사하세요:

```bash
mkdir -p ~/.claude/skills/news-collector
cp SKILL.md ~/.claude/skills/news-collector/
```

## API 키 설정 (최초 1회)

1. [SerpAPI](https://serpapi.com) 가입 → API 키 발급 (무료 100회/월)
2. [Tavily](https://app.tavily.com) 가입 → API 키 발급 (무료 1,000회/월)

```bash
echo "SERPAPI_API_KEY=여기에_키_입력" >> ~/.env
echo "TAVILY_API_KEY=여기에_키_입력" >> ~/.env
```

## 사용법

Claude Code에서 자연어로 말하면 됩니다:

```
"AI 뉴스 모아줘"
"반도체, 경제 뉴스 각 20건 수집해줘"
"BTS 최근 3일 뉴스 최신순으로"
"스타트업 뉴스 모아서 분석해줘"
```

## 수집 파이프라인

```
SerpAPI → 구글 뉴스 검색 (제목/URL)
  ↓
Playwright (stealth) → URL에서 본문 추출 (JS 렌더링, 36+ 한국 매체)
  ↓ 실패 시
newspaper3k → 정적 HTML 폴백
  ↓ SerpAPI 키 없을 때
Tavily → 검색+본문 통합 (최후 보루)
```

## 결과

바탕화면 `뉴스수집` 폴더에 자동 저장:

- `news_키워드_날짜.csv` — Excel에서 바로 열 수 있는 CSV
- `news_키워드_날짜.json` — Claude 분석용 JSON

## 분석 기능

수집 후 "분석해줘"라고 하면 Claude가 직접:
- 핵심 요약
- 트렌드 분석
- 감성 분류 (긍정/부정/중립)
- 인사이트 도출

## 호환성

- Claude Code (Anthropic)
- Antigravity
- WSL / macOS / Linux

## 라이선스

MIT
