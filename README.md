# 굿모닝, 9시 보안 뉴스 (Good Morning, 9 o'clock Security News)

RSS로 모은 보안 기사들을 **정규화**해서, 브라우저에서 즉시 **정렬/분류/검색**해 보여주는 **가벼운 정적 웹페이지**입니다.  
프런트는 정적 JSON만 읽고, 데이터 갱신은 서버(crontab) 또는 GitHub Actions가 맡습니다.

- **데모**: *(GitHub Pages URL)*  
- **주요 소스**: 보안뉴스, 데일리시큐

---

## 정규화 스키마
```
json
{ "title": "", "link": "", "summary": "", "published": "", "author": "", "source": "", "cats": [] }
```

## ✨ 기능  

- **시간 경계**: 전날 09:00 ~ 오늘 09:00 기사 표시 + 오늘분 이후 소식(최신순) 블록 분리

- **필터/탐색**: 소스(전체/보안뉴스/데일리시큐), 카테고리(전체/해양/포렌식/AI), 검색

- **정렬/정제**: 항상 최신순, 중복 링크 제거

- **테마**: Sunrise (살구→하늘 그라데이션), 카테고리/소스 칩 색상

- **완전 정적**: 서버/DB 없이 정적 JSON fetch만으로 렌더

<br><br>

## 🧠 How it works?
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

- **업데이트**: 내 서버 **crontab** 또는 **GitHub Actions**가 주기적으로 JSON 생성→커밋

- 프로젝트 핵심은 **정규화(normalize)** 입니다.
(보안뉴스와 데일리시큐 뉴스의 rss 필드를 하나의 스키마로 통일)

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

<br>

### 시간 경계(09→09)

하루 경계를 **아침 9시**로 잡아, “아침에 전체를 훑기” 좋게 구성했습니다.

오늘 날짜일 때는 **09시 이후**의 새 기사들을 “이후 소식(최신순)” 블록으로 따로 보여줍니다.

<br>

### 분류(카테고리/소스/검색)

기사 title + summary를 키워드로 문자열 매칭하여 카테고리를 판단합니다.

**해양(OCEAN)** 은 오검출을 줄이려고 **엄격 포함어 + 제외어** 필터를 사용합니다.

**소스 버튼**(전체/보안뉴스/데일리시큐)으로 먼저 거르고, 그다음 **카테고리/검색**을 적용합니다.

만약 항목에 서버 분류`item.cats`가 있으면 그 값을 최우선으로 사용합니다.

<br>

## 🔄데이터 자동 갱신 (둘 중 택 1)

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

<br>

## ❓FAQ(요약)

Q. 서버 코드는 왜 없나요?
A. GitHub Pages는 정적 호스팅이라 실행 환경이 없습니다. 대신 `crontab/Actions`가 JSON을 생성해 커밋합니다.

Q. 데이터가 계속 쌓이면?
A. `feed.json`은 최신만 유지하고, 과거는 `archive/daily-YYYY-MM-DD.json`로 분리하여 용량을 관리합니다.

<br><br>

## 🎨 테마 적용

HTML 상단에 테마를 불러옵니다:

`<link rel="stylesheet" href="assets/sunrise.css">`


`index.html`의 <body>에 테마 클래스를 주면 헤더 배경까지 활성화됩니다:

`<body class="sunrise">`

<br>

## 🔧 데이터 갱신

외부에서 RSS를 수집/파싱하여 `feed.json`, `daily.json`, `archive/`를 업데이트합니다.

프런트는 정적 파일만 읽으므로, 파일만 교체해도 화면이 자동 반영됩니다.

<br>

## 🤔 문의사항
필요하면 나중에 make_feed.py 예시도 넣을 예정입니다만, 지금은 “정적 프런트 + 백그라운드로 JSON 자동 갱신” 콘셉트가 또렷하게 보이도록 최소∙명확 구성을 넣었습니다.
그외 문의할 내용이 있는 경우 `6427d3010@gmail.com`으로 연락주세요 ^_^
