---
name: news-collector
description: Google News collection and analysis. Searches Korean news via SerpAPI/Tavily, extracts full articles with Playwright stealth, cleans 50+ outlet noise, saves CSV/Excel. Triggers on "뉴스 수집", "뉴스 검색", "구글 뉴스", "knews", "뉴스 모아줘", "news collect".
---

# 구글 뉴스 수집기 (News Collector)

키워드 기반 구글 뉴스 검색 → Playwright 본문 추출 → CSV 저장 → (선택) 분석까지 자동 수행합니다.

## 역할

당신은 뉴스 리서처이자 트렌드 분석가입니다.
사용자의 관심 주제를 파악하고, 최적의 검색 전략으로 뉴스를 수집한 뒤, 핵심 인사이트를 도출합니다.
사용자와 한국어로 대화합니다.

---

## 전체 흐름

```
Phase 0: SETUP (최초 1회)
  └── API 키 설정 → ~/.env

Phase 1: INTAKE (정보 수집)
  └── 사용자 요청에서 키워드/건수/기간 자연스럽게 파악

Phase 2: COLLECT (뉴스 수집)
  ├── PROJECT_DIR 생성
  ├── news_collect.py 작성 (아래 코드)
  └── python3 news_collect.py 실행 → CSV + JSON 저장

Phase 3: ANALYZE (선택적 분석)
  └── JSON을 Read 도구로 읽고 Claude가 직접 분석
```

> **중요:** 매 작업 시작 시 **키워드+날짜 기반 고유 폴더**를 생성합니다.
> ```bash
> PROJECT_DIR="/mnt/c/Users/daniel/Desktop/뉴스수집/[keyword_slug]_$(date +%Y%m%d_%H%M)"
> mkdir -p "$PROJECT_DIR"
> ```

---

# Phase 0: SETUP (최초 1회)

API 키가 `~/.env`에 있는지 확인합니다. 없으면 사용자에게 안내:

```
API 키가 필요합니다:
1. SerpAPI 키 (구글 뉴스 검색) — https://serpapi.com (무료 100회/월)
2. Tavily API 키 (폴백 검색) — https://app.tavily.com (무료 1,000회/월)
```

키를 받으면 저장:
```bash
echo "SERPAPI_API_KEY=받은키" >> ~/.env
echo "TAVILY_API_KEY=받은키" >> ~/.env
```

---

# Phase 1: INTAKE

사용자 요청을 자연스럽게 파악하여 파라미터로 변환합니다.

| 사용자 표현 | keywords | count | days | 기타 |
|------------|----------|-------|------|------|
| "AI 뉴스 모아줘" | AI | 10 | 7 | |
| "BTS 최근 3일 20건 최신순" | BTS | 20 | 3 | latest |
| "AI, 반도체, 경제 각 15건" | AI,반도체,경제 | 15 | 7 | |
| "스타트업 뉴스 분석해줘" | 스타트업 | 10 | 7 | → Phase 3 |

기본값: 10건, 7일, auto 백엔드

---

# Phase 2: COLLECT

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

## 실행

`$PROJECT_DIR/news_collect.py`로 아래 코드를 저장하고 실행합니다.

```bash
# 의존성 (최초 1회)
pip install -q tavily-python httpx beautifulsoup4 lxml lxml-html-clean rich python-dotenv openpyxl playwright playwright-stealth newspaper3k 2>/dev/null
playwright install chromium 2>/dev/null

# 실행
cd "$PROJECT_DIR" && python3 news_collect.py --keywords "AI,반도체" --count 10 --days 7
```

## news_collect.py

```python
#!/usr/bin/env python3
"""구글 뉴스 수집기 - SerpAPI/Tavily 검색 + Playwright 본문 추출 + CSV 저장"""

import argparse, asyncio, csv, json, os, re, sys
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
from urllib.parse import urlparse

import httpx
from bs4 import BeautifulSoup
from dotenv import load_dotenv
from rich.console import Console
from rich.panel import Panel

console = Console()

EXCLUDED_DOMAINS = {"news.nate.com", "nate.com", "msn.com", "www.msn.com"}

@dataclass
class NewsResult:
    title: str; url: str; snippet: str = ""; content: str = ""
    source: str = ""; published_date: str = ""; score: float = 0.0

# ── 검색 ──

def search_serpapi(query, max_results=10, days=7, site=None):
    api_key = os.getenv("SERPAPI_API_KEY", "")
    if not api_key:
        return []
    tbs_map = {1: "qdr:d", 3: "qdr:d3", 7: "qdr:w", 30: "qdr:m"}
    tbs = "qdr:w"
    for t in sorted(tbs_map):
        if days <= t: tbs = tbs_map[t]; break
    q = f"site:{site} {query}" if site else query
    params = {"engine": "google_news", "q": q, "gl": "kr", "hl": "ko",
              "num": str(min(max_results, 100)), "tbs": tbs, "api_key": api_key}
    data = httpx.get("https://serpapi.com/search", params=params, timeout=15.0).json()
    results = []
    for item in data.get("news_results", []):
        url = item.get("link", "")
        src = urlparse(url).netloc.replace("www.", "") if url else ""
        if not url and "stories" in item:
            for s in item["stories"][:1]:
                url = s.get("link", ""); src = s.get("source", {}).get("name", "")
        if url and not any(d in url for d in EXCLUDED_DOMAINS):
            results.append(NewsResult(title=item.get("title",""), url=url, snippet=item.get("snippet","")[:300],
                source=src or item.get("source",{}).get("name",""), published_date=item.get("date","")))
    return results[:max_results]

def search_tavily(query, max_results=10, days=7, site=None):
    from tavily import TavilyClient
    api_key = os.getenv("TAVILY_API_KEY", "")
    if not api_key:
        console.print("[red]TAVILY_API_KEY 미설정. ~/.env에 추가하세요.[/red]"); sys.exit(1)
    resp = TavilyClient(api_key=api_key).search(
        query=query, search_depth="advanced", topic="news", days=days,
        max_results=min(max_results, 20), include_raw_content=True,
        include_domains=[site] if site else None)
    results = []
    for item in resp.get("results", []):
        raw = item.get("raw_content","") or item.get("content","")
        u = item.get("url",""); src = urlparse(u).netloc.replace("www.","") if u else ""
        if any(d in u for d in EXCLUDED_DOMAINS): continue
        results.append(NewsResult(title=item.get("title",""), url=u,
            snippet=item.get("content","")[:300], content=clean_news_body(raw, u) if raw else "",
            source=src, published_date=item.get("published_date",""), score=item.get("score",0.0)))
    return results

def search_news(query, max_results=10, days=7, site=None, backend="auto"):
    if backend == "auto":
        backend = "serpapi" if os.getenv("SERPAPI_API_KEY") else "tavily"
    return search_serpapi(query, max_results, days, site) if backend == "serpapi" else search_tavily(query, max_results, days, site)

# ── 클리닝 (한국 뉴스 50+ 매체 노이즈 제거) ──

def clean_news_body(raw, url=""):
    if not raw or not isinstance(raw, str) or len(raw) < 50: return raw or ""
    if re.search(r"contrary to the Web firewall|Hold tight|보안 검사|CAPTCHA", raw, re.I): return ""
    t = raw
    t = re.sub(r"\[[^\]]+\]\([^\)]+\)", "", t)
    t = re.sub(r"!\[.*?\]\(.*?\)", "", t)
    t = re.sub(r"\n?/?[가-힣]{2,4}\s*기자\s*[a-zA-Z0-9._-]+@[^\s\n]+", "", t)
    t = re.sub(r"\[[가-힣]+(?:데일리|투데이|뉴스|타임즈|경제|일보|신문)\s*[가-힣]*(?:기자|리포터)\]", "", t)
    t = re.sub(r"\([^)]*=\s*[^)]+\)\s*[가-힣\s]+기자\s*[=]*\s*", "", t)
    t = re.sub(r"[가-힣]+기자\s+구독\s+구독중", "", t)
    t = re.sub(r"\[[^\]]*기자\]", "", t)
    t = re.sub(r"◎공감언론\s*뉴시스", "", t)
    t = re.sub(r"\n?[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.(co\.kr|com|net)\s*$", "", t, flags=re.M)
    t = re.sub(r"입력\s*\d{4}\.\d{2}\.\d{2}\.\s*\d{2}:\d{2}", "", t)
    t = re.sub(r"\[By Taboola\][^\n]*|\[AD\][^\n]*", "", t)
    filtered = []
    for line in [l.strip() for l in t.split("\n") if l.strip()]:
        if len(line) < 2 or re.match(r"^\d+\.?\s*$", line): continue
        if re.match(r"^(정치|사회|경제|국제|IT|과학|문화|연예|스포츠)$", line, re.I): continue
        if re.match(r"^(관련기사|최신기사|추천기사|많이 본 뉴스)", line): break
        if re.match(r"^(SNS 공유하기|Copyright|AD$)", line, re.I): break
        if re.search(r"저작권자|무단\s*(전재|복제|배포)|All Rights Reserved", line, re.I): continue
        if re.search(r"사업자등록번호|Taboola|Sponsored", line, re.I): continue
        filtered.append(line)
    t = "\n".join(filtered)
    t = re.sub(r"https?://[^\s)]+", "", t)
    t = re.sub(r"[▶▷●◆■★※▲▼→←↑↓#♥♡✓✔☞◎|│┃]", "", t)
    t = re.sub(r"\*\*", "", t)
    t = re.sub(r"[\U0001F300-\U0001F9FF\u2600-\u26FF\u2700-\u27BF]", "", t)
    t = re.sub(r"\n{3,}", "\n\n", t); t = re.sub(r" {2,}", " ", t)
    return t.strip()

# ── 본문 추출 (Playwright + newspaper3k) ──

JS_RENDER_SITES = {
    "jtbc.co.kr","news.jtbc.co.kr","chosun.com","biz.chosun.com","ichannela.com",
    "mbc.co.kr","imnews.imbc.com","sbs.co.kr","news.sbs.co.kr","tvchosun.com",
    "ytn.co.kr","news.zum.com","v.daum.net","n.news.naver.com","vogue.co.kr",
    "news1.kr","mk.co.kr","news.kbs.co.kr","etnews.com","nocutnews.co.kr",
    "joongang.co.kr","donga.com","khan.co.kr","hani.co.kr","sedaily.com",
    "mt.co.kr","biz.heraldcorp.com","edaily.co.kr","asiae.co.kr","fnnews.com",
    "newsis.com","ddaily.co.kr","inews24.com","mbn.co.kr","yna.co.kr","maxmovie.com",
}

@dataclass
class Article:
    title: str; url: str; content: str; content_length: int; method: str
    thumbnail: str = ""; success: bool = True; error: str = ""

def parse_html(html, url):
    soup = BeautifulSoup(html, "lxml")
    for tag in soup(["script","style","nav","header","footer","aside","iframe","noscript"]): tag.decompose()
    for el in soup.find_all(class_=re.compile(r"\bad\b|advertisement|sidebar|related|comment|share", re.I)): el.decompose()
    title = ""; og = soup.find("meta", property="og:title")
    if og and og.get("content"): title = og["content"]
    if not title:
        tt = soup.find("title"); title = tt.get_text(strip=True) if tt else ""
    thumb = ""; oi = soup.find("meta", property="og:image")
    if oi and oi.get("content","").startswith("http"): thumb = oi["content"]
    content = ""
    for sel in ["article",".article-body",".article-content",".entry-content",".article_body",
                ".news-body",'div[itemprop="articleBody"]',"#articleBodyContents","#newsEndContents",
                ".news_cnt_detail_wrap",".article_txt",".newsct_article",".viewer_article",
                ".article_view",".news_view","#articleTxt","#article-view-content-div"]:
        el = soup.select_one(sel)
        if el:
            ps = [p.get_text(strip=True) for p in el.find_all("p") if len(p.get_text(strip=True)) > 20]
            if ps: content = "\n\n".join(ps)
            if len(content) < 100: content = el.get_text(separator="\n", strip=True)
            if len(content) > 100: break
    if len(content) < 100:
        content = "\n\n".join(p.get_text(strip=True) for p in soup.find_all("p")
                              if len(p.get_text(strip=True)) > 30
                              and not any(k in p.get_text(strip=True).lower() for k in ("cookie","로그인","copyright","저작권")))
    return title, clean_news_body(content, url), thumb

async def extract_playwright(url):
    try:
        from playwright.async_api import async_playwright; from playwright_stealth import Stealth
    except ImportError:
        return Article(title="",url=url,content="",content_length=0,method="playwright",success=False,error="playwright 미설치")
    try:
        async with async_playwright() as p:
            br = await p.chromium.launch(headless=True, args=["--no-sandbox","--disable-dev-shm-usage","--disable-gpu"])
            ctx = await br.new_context(viewport={"width":1920,"height":1080},
                user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/131.0.0.0 Safari/537.36",
                locale="ko-KR", timezone_id="Asia/Seoul")
            page = await ctx.new_page(); await Stealth().apply_stealth_async(page)
            await page.route("**/*.{png,jpg,jpeg,gif,svg,woff,woff2,ttf,eot}", lambda r: r.abort())
            try: await page.goto(url, wait_until="domcontentloaded", timeout=20000); await page.wait_for_timeout(3000)
            except: pass
            html = await page.content(); await br.close()
        title, content, thumb = parse_html(html, url)
        ok = len(content) >= 50
        return Article(title=title,url=url,content=content,content_length=len(content),method="playwright",thumbnail=thumb,success=ok,error="" if ok else f"본문 {len(content)}자")
    except Exception as e:
        return Article(title="",url=url,content="",content_length=0,method="playwright",success=False,error=str(e))

async def extract_article(url):
    result = await extract_playwright(url)
    if result.success and result.content_length >= 200: return result
    try:
        from newspaper import Article as NA
        a = NA(url, language="ko"); a.download(); a.parse()
        c = clean_news_body(a.text or "", url)
        if len(c) > (result.content_length or 0):
            return Article(title=a.title or "",url=url,content=c,content_length=len(c),method="newspaper3k",thumbnail=a.top_image or "")
    except: pass
    return result

# ── 메인 ──

def main():
    load_dotenv(); load_dotenv(Path.home()/".env")
    ap = argparse.ArgumentParser()
    ap.add_argument("--keywords", required=True); ap.add_argument("--count", type=int, default=10)
    ap.add_argument("--days", type=int, default=7); ap.add_argument("--backend", default="auto")
    ap.add_argument("--latest", action="store_true"); ap.add_argument("--output", default=None)
    args = ap.parse_args()

    queries = [q.strip() for q in args.keywords.split(",") if q.strip()]
    console.print(f"\n[bold blue]뉴스 검색[/bold blue] {len(queries)}개: {', '.join(queries)}")
    console.print(f"  최근 {args.days}일 / 키워드당 {args.count}건 / 백엔드: {args.backend}", style="dim")

    results, seen = [], set()
    for qi, query in enumerate(queries, 1):
        if len(queries) > 1: console.print(f"\n  [cyan][{qi}/{len(queries)}][/cyan] '{query}'...")
        for r in search_news(query, args.count, args.days, backend=args.backend):
            if r.url not in seen: seen.add(r.url); r._keyword = query; results.append(r)
    if not results: console.print("[yellow]검색 결과 없음[/yellow]"); return
    console.print(f"\n  [green]총 {len(results)}건[/green]")

    articles, tasks = [], []
    for i, r in enumerate(results):
        d = {"keyword":getattr(r,"_keyword",queries[0]),"title":r.title,"url":r.url,"content":r.content,
             "content_length":len(r.content),"source":r.source,"published_date":r.published_date,"method":"tavily","success":True}
        if len(r.content) < 200: tasks.append((i, r))
        articles.append(d)

    if tasks:
        console.print(f"  [cyan]{len(tasks)}건 본문 추출 중...[/cyan]")
        async def _all():
            sem = asyncio.Semaphore(5)
            async def _one(u):
                async with sem: return await extract_article(u)
            return await asyncio.gather(*[_one(r.url) for _, r in tasks])
        for (idx, r), ext in zip(tasks, asyncio.run(_all())):
            if ext.success and len(ext.content) > len(r.content):
                articles[idx].update(content=ext.content, content_length=ext.content_length, method=ext.method)
            elif not ext.success and len(r.content) < 50:
                articles[idx].update(success=False, error=ext.error)

    if args.latest: articles.sort(key=lambda a: a.get("published_date",""), reverse=True)

    slug = queries[0].replace(" ","_")[:20]; ts = datetime.now().strftime("%Y%m%d_%H%M")
    fn = args.output or f"news_{slug}_{ts}.csv"
    with open(fn, "w", encoding="utf-8-sig", newline="") as f:
        w = csv.writer(f); w.writerow(["키워드","제목","출처","URL","본문길이","성공","본문"])
        for a in articles:
            w.writerow([a.get("keyword",""),a.get("title",""),a.get("source",""),a.get("url",""),
                        a.get("content_length",0),"O" if a.get("success") else "X",a.get("content","")])

    ok = [a for a in articles if a.get("success")]; fail = [a for a in articles if not a.get("success")]
    console.print(Panel(f"[bold]검색어:[/bold] {', '.join(queries)}\n[bold]결과:[/bold] {len(articles)}건  "
        f"[green]성공 {len(ok)}[/green]  [red]실패 {len(fail)}[/red]\n[bold]저장:[/bold] {fn}",
        title="[bold blue]뉴스 수집 결과[/bold blue]", border_style="blue"))
    for i, a in enumerate(articles[:10], 1):
        s = "[green]OK[/green]" if a.get("success") else "[red]FAIL[/red]"
        console.print(f"\n  {s} {i}. {a.get('title','')} ({a.get('source','')})")
        console.print(f"     {a.get('url','')}")
        if a.get("success") and a.get("content"):
            console.print(f"     [dim]{a['content'][:150]}...[/dim]")
            console.print(f"     [cyan]{a.get('content_length',0):,}자[/cyan]")

    jf = fn.replace(".csv",".json")
    with open(jf,"w",encoding="utf-8") as f:
        json.dump({"query":queries,"count":len(articles),"success":len(ok),"failed":len(fail),"file":fn,"articles":articles},f,ensure_ascii=False,indent=2)
    console.print(f"\n[dim]JSON: {jf}[/dim]")

if __name__ == "__main__": main()
```

---

# Phase 3: ANALYZE (선택적)

사용자가 "분석해줘", "요약해줘" 등을 요청하면 실행합니다.

1. Read 도구로 JSON 파일 읽기
2. Claude가 직접 분석
3. 마크다운으로 보고

| 유형 | 사용자 표현 | Claude가 하는 일 |
|------|-----------|----------------|
| 요약 | "요약해줘" | 핵심 내용 5줄 요약 |
| 트렌드 | "트렌드 알려줘" | 키워드 빈도, 매체별 분포, 시간 흐름 |
| 감성 | "긍정/부정 분류" | 긍정/부정/중립 비율 |
| 인사이트 | "인사이트 뽑아줘" | 핵심 발견 5개 + 시사점 |
| 종합 | "분석해줘" | 위 전부 간략 수행 |

---

# 에러 대응

| 에러 | 해결 |
|------|------|
| API 키 미설정 | `~/.env`에 SERPAPI_API_KEY, TAVILY_API_KEY 추가 |
| playwright 미설치 | `playwright install chromium` |
| 429 rate limit | `--backend tavily`로 변경 |
| CSV 0건 | 키워드 변경 또는 `--days 30`으로 기간 확대 |
