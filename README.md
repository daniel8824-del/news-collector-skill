# News Collector Skill

AI가 키워드만으로 구글 뉴스를 자동 수집하고 본문까지 추출합니다.

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.com/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## Features

- 키워드만 말하면 **구글 뉴스 자동 수집** (멀티 키워드 지원)
- SerpAPI(구글 뉴스) + Tavily(범용) 듀얼 백엔드
- **Playwright stealth** 본문 추출 (한국 뉴스 36+ 매체 JS 렌더링)
- newspaper3k 폴백 (정적 사이트)
- 한국 뉴스 50+ 매체 노이즈 클리닝 (기자정보, 광고, 푸터 자동 제거)
- 최종 결과물: **CSV + JSON** (Excel 바로 열기 가능)
- Claude가 직접 분석 (요약, 트렌드, 감성, 인사이트)

## 사용 방법

### 1. Claude Code 스킬

```bash
git clone https://github.com/daniel8824-del/news-collector-skill.git ~/.claude/skills/news-collector
```

Claude Code에서:
```
"AI, 반도체 뉴스 20건 모아줘"
"BTS 최근 3일 뉴스 최신순으로"
"스타트업 뉴스 수집해서 분석해줘"
```

### 2. Google Antigravity

```bash
git clone https://github.com/daniel8824-del/news-collector-skill.git
cd news-collector-skill
```

`.agent/` 폴더가 자동 인식됩니다.
Antigravity에서 "구글 뉴스 모아줘"라고 입력하면 스킬이 실행됩니다.

## API Keys

`~/.env` 파일에 설정:
```
SERPAPI_API_KEY=필수 (구글 뉴스 검색, 무료 100회/월)
TAVILY_API_KEY=선택 (폴백 검색, 무료 1,000회/월)
```

API 키 발급:
- SerpAPI: https://serpapi.com
- Tavily: https://app.tavily.com

## 수집 파이프라인

```
SerpAPI → 구글 뉴스 검색 (제목/URL)
  ↓
Playwright (stealth) → URL에서 본문 추출 (JS 렌더링, 36+ 매체)
  ↓ 실패 시
newspaper3k → 정적 HTML 폴백
  ↓ SerpAPI 키 없을 때
Tavily → 검색+본문 통합 (최후 보루)
```

## 분석 기능

수집 후 "분석해줘"라고 하면 Claude가 직접:
- 핵심 요약 (5줄)
- 트렌드 분석 (키워드 빈도, 매체별 분포)
- 감성 분류 (긍정/부정/중립 비율)
- 인사이트 도출 (핵심 발견 5개)

## License

MIT
