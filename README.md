# SecuNews Normalizer - 굿모닝, 9시 보안 뉴스 (Good Morning, 9 o'clock Security News)

RSS로 모은 보안 기사들을 **정규화**해서, 브라우저에서 즉시 **정렬/분류/검색**해 보여주는 **가벼운 정적 웹페이지**입니다.  
프런트는 정적 JSON만 읽고, 데이터 갱신은 서버(crontab) 또는 GitHub Actions가 맡습니다.

- **데모**: *http://binpage.kr/news*  
- **주요 소스**: 보안뉴스(https://www.boannews.com/media/news_rss.xml), 데일리시큐(https://www.dailysecu.com/rss/allArticle.xml)

---



## ✨ 기능  

- **시간 경계**: 전날 09:00 ~ 오늘 09:00 기사 표시 + 오늘분 이후 소식(최신순) 블록 분리

- **필터/탐색**: 소스(전체/보안뉴스/데일리시큐), 카테고리(전체/해양/포렌식/AI), 검색

- **정렬/정제**: 항상 최신순, 중복 링크 제거

- **테마**: Sunrise (살구→하늘 그라데이션), 카테고리/소스 칩 색상

- **완전 정적**: 서버/DB 없이 정적 JSON fetch만으로 렌더

<br><br>

## 🧠 어떻게 동작하나요?
```
[RSS 제공사]
      ↓ (파싱/정규화: Python 등)
   feed.json (+ daily.json, archive/…)
      ↓ (정적 파일 커밋/업데이트)
   GitHub Pages가 서빙
      ↓
index.html / all.html에서 fetch → 필터 → 렌더
```

- **UI(프런트)**: `index.html`, `all.html`, `assets/sunrise.css`

- **데이터**:

    - `feed.json` (실시간 최신)

    - `daily.json` (오늘 09→09)

    - `archive/daily-YYYY-MM-DD.json`

- **업데이트 파이프라인**: 내 서버 **crontab** 또는 **GitHub Actions**가 주기적으로 JSON 생성→커밋

- **프로젝트 핵심은 정규화(normalize)** 입니다.
      - 보안 뉴스와 데일리 시큐의 **RSS 항목 필드와 형식**(예: 제목, 링크, 요약, 발행시각, 작성사, 소스, 카테고리등)을 **단일 스키마**로 통일해 `feed.json` / `daily.json` / `archive/*.json`에 저장하였습니다.
      - 발생시각 포맷/ 타임존 일치/ 불필요한 HTML 제거, 중복 링크 제거 등을 고려하였습니다.

<br><br>

## 📁 디렉터리
```
/
├─ index.html            # 아침판(09→09) + '이후 소식' 섹션
├─ all.html              # 전체 피드 탐색(검색/필터/무한로드)
├─ feed.json             # 최신 정규화 데이터(자동 갱신 대상)
├─ daily.json            # 오늘(09→09) 집계
├─ archive/
│   └─ daily-YYYY-MM-DD.json
├─ assets/
│   └─ sunrise.css       # Sunrise 테마 CSS
└─ .nojekyll             # Jekyll 비활성화(그대로 서빙)

```
> **.nojekyll**: GitHub Pages에서 Jekyll 처리를 막아 폴더/언더스코어 파일 등을 있는 그대로 서빙하게 합니다.

<br><br>

## 정규화 스키마
```
{
  "title": "스트링",
  "link": "https://example.com/article",
  "summary": "간단 요약(HTML 제거/트림)",
  "published": "2025-10-02T07:12:00Z",     // RFC3339 UTC 권장
  "author": "작성자명 (없으면 생략 가능)",
  "source": "보안뉴스 | 데일리시큐 등",
  "cats": ["OCEAN", "FORENSICS", "AI"]     // 선택적; 있으면 프런트가 최우선 사용
}
```
> **권장**: published는 **UTC ISO-8601(RFC3339)** 로 저장하고, 필요 시 `published_kst`(KST 변환값)를 추가 저장할 수 있어요.

<br><br>

## 핵심 기능

### 시간 경계(09→09)

하루 경계를 **아침 9시**로 잡아, “아침에 전체를 훑기” 좋게 구성했습니다.

오늘 날짜일 때는 **09시 이후**의 새 기사들을 “이후 소식(최신순)” 블록으로 시각적으로 분리합니다.

<br>

### 분류 로직

- 기본은 제목 + 요약에 대한 키워드 매칭(문자열)입니다.

- 해양(OCEAN) 은 오검출을 줄이려 엄격 포함어 + 제외어 필터를 사용합니다.

- 소스 버튼(전체/보안뉴스/데일리시큐)으로 먼저 거르고, 그다음 카테고리/검색을 적용합니다.

- 항목에 서버 분류(item.cats) 가 있으면 그 값을 최우선으로 사용합니다.

<br><br>


## 🔄데이터 자동 갱신 (택 1)

### A) 내 서버(crontab)
`update_feed.sh`:
```
#!/usr/bin/env bash
set -e
cd /path/to/repo

python make_feed.py             # RSS → feed.json 생성
git add feed.json daily.json archive/
git -c user.name="cron-bot" -c user.email="bot@example.com" commit -m "chore: update $(date -u +'%Y-%m-%dT%H:%MZ')" || true
git push origin main
```
크론 등록(매 30분):
```
*/30 * * * * /path/to/update_feed.sh >> /var/log/update_feed.log 2>&1
```

<br>

### B) GitHub Actions(서버 없이)
`.github/workflows/update-feed.yml`:
```
name: Update feed.json
on:
  schedule:
    - cron: "*/30 * * * *"
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install feedparser
      - name: Build feed.json
        run: python make_feed.py
      - name: Commit & push
        run: |
          git config user.name "gh-actions"
          git config user.email "actions@users.noreply.github.com"
          git add feed.json daily.json archive/
          git commit -m "chore: update feed.json (CI)" || echo "No changes"
          git push
```

<br>

### 로컬 미리보기
```
python -m http.server 8000
#http://localhost:8000 에서 index.html / all.html 확인
```

<br><br>

## ➕ 새 소스 추가 가이드(요약)

1. RSS에서 **title/link/summary/published/author/source** 를 추출

2. **타임존/포맷 정규화**: `published` → **UTC ISO-8601**

3. **HTML/공백 정리, 중복 링크 제거(Set)**

4. 필요 시 `cats`(["OCEAN", "FORENSICS", "AI"]) 태깅

5. 결과를 `feed.json / daily.json / archive/*.json` 에 합쳐 저장

> 정규화는 **필드 이름**만 맞추는 게 아니라 **값의 형식/시각/문자열 정리**까지 포함합니다.

<br><br>

## 🎨 테마 적용

- HTML 상단에 테마를 불러옵니다: <br>
`<link rel="stylesheet" href="assets/sunrise.css">`

- `index.html`의 <body>에 테마 클래스를 주면 헤더 배경까지 활성화됩니다: <br>
`<body class="sunrise">`

<br><br>

## ❓FAQ(요약)

- Q. 서버 코드는 왜 없나요? <br>
A. GitHub Pages는 정적 호스팅입니다. 대신 crontab/Actions가 JSON을 만들어서 커밋합니다.

- Q. 데이터가 계속 쌓이면 무거워지지 않나요? <br>
A. feed.json은 최신만, 과거는 archive/로 분리해 용량을 관리합니다.

- Q. 이 프로젝트가 “크롤링”을 안 해서 의미가 없나요? <br>
A. 아닙니다. 이 프로젝트의 가치는 이기종 RSS를 하나의 스키마로 정규화하고, 동일한 UI/UX로 빠르게 탐색하게 만드는 데 있습니다.

<br><br>

## 🤔 그외
필요하면 나중에 make_feed.py 예시도 넣을지 고민중입니다만, 지금은 “정적 프런트 + 백그라운드로 JSON 자동 갱신” 콘셉트가 또렷하게 보이도록 최소∙명확 구성을 넣었습니다. 
<br>
그외 문의할 내용이 있는 경우 <6427d3010@gmail.com>으로 연락주세요. ^_^ <br>
이슈/PR 환영합니다.
