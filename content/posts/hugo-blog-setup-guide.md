---
title: "Hugo로 개인 블로그 만들기 — 처음부터 배포까지"
date: 2026-03-31
draft: false
tags: ["Hugo", "블로그", "GitHub Pages", "튜토리얼"]
categories: ["IT"]
summary: "Hugo 정적 사이트 생성기와 PaperMod 테마로 개인 블로그를 만들고, GitHub Pages에 배포하는 과정을 정리합니다."
---

## 왜 Hugo인가?

블로그를 시작하려면 선택지가 많습니다. WordPress, Tistory, Notion 등 다양한 플랫폼이 있지만, 직접 만들어보고 싶었습니다.

Hugo를 선택한 이유는 간단합니다:

- **빠릅니다** — 수천 개의 글도 1초 안에 빌드
- **마크다운** 기반이라 글 쓰기가 편합니다
- **무료 호스팅** — GitHub Pages로 비용 없이 배포 가능
- **커스터마이징** 자유도가 높습니다

## 준비물

- [Hugo](https://gohugo.io/installation/) (extended 버전 권장)
- [Git](https://git-scm.com/)
- GitHub 계정
- 터미널에 대한 기본적인 이해

macOS 기준으로 Hugo 설치:

```bash
brew install hugo
```

설치 확인:

```bash
hugo version
```

## 1단계: 사이트 생성

원하는 디렉토리에서 Hugo 사이트를 생성합니다.

```bash
hugo new site blog
cd blog
git init
```

이 명령이 만들어주는 기본 구조:

```
blog/
├── archetypes/    # 새 글 생성 시 기본 템플릿
├── assets/        # CSS, JS 등 처리 대상 파일
├── content/       # 블로그 글이 들어가는 곳 (가장 중요!)
├── data/          # 사이트 데이터 파일
├── layouts/       # HTML 템플릿 커스터마이징
├── static/        # 이미지 등 정적 파일
├── themes/        # 테마 폴더
└── hugo.toml      # 사이트 설정 파일
```

> **핵심 개념:** Hugo는 `content/` 안의 마크다운 파일을 읽어서 `public/` 폴더에 HTML을 생성합니다. 이 HTML이 실제로 브라우저에 보이는 사이트입니다.

## 2단계: 테마 설치

테마 없이는 아무것도 보이지 않습니다. [PaperMod](https://github.com/adityatelange/hugo-PaperMod)를 설치합니다. Hugo 테마 중 가장 인기 있고, 깔끔합니다.

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

> **git submodule이란?** 다른 git 저장소를 내 저장소 안에 포함시키는 방법입니다. 테마를 직접 복사하는 것보다 업데이트 관리가 쉽습니다.

## 3단계: 사이트 설정

`hugo.toml` 대신 `hugo.yaml`을 사용하겠습니다. YAML이 읽기 편합니다. 기존 `hugo.toml`을 삭제하고 `hugo.yaml`을 만듭니다.

```yaml
baseURL: "https://your-username.github.io/blog/"
title: My Blog
pagination:
  pagerSize: 10
theme: PaperMod
languageCode: ko

enableRobotsTXT: true

minify:
  disableXML: true
  minifyOutput: true

# 검색 기능을 위해 JSON 출력 추가
outputs:
  home:
    - HTML
    - RSS
    - JSON

params:
  env: production        # SEO 관련 메타 태그 활성화
  description: "내 블로그"
  author: Your Name
  defaultTheme: auto     # 다크/라이트 모드 자동 전환
  ShowReadingTime: true   # 읽는 시간 표시
  ShowCodeCopyButtons: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true

  homeInfoParams:
    Title: "Welcome"
    Content: "개인 블로그입니다."

menu:
  main:
    - identifier: search
      name: 검색
      url: /search/
      weight: 10
    - identifier: tags
      name: 태그
      url: /tags/
      weight: 20
    - identifier: archives
      name: 아카이브
      url: /archives/
      weight: 30
```

주요 설정을 하나씩 살펴보면:

| 설정 | 역할 |
|------|------|
| `baseURL` | 사이트의 실제 주소. 배포 시 이 주소 기준으로 링크가 생성됩니다 |
| `theme: PaperMod` | 사용할 테마 이름 (themes/ 폴더 안의 디렉토리명) |
| `enableRobotsTXT` | 검색 엔진 크롤러 제어 파일 자동 생성 |
| `params.env: production` | OpenGraph, Schema.org 등 SEO 메타 태그 활성화 |
| `outputs.home`에 `JSON` | 검색 기능에 필수. 이걸 빼면 검색이 안 됩니다 |

## 4단계: 검색과 아카이브 페이지 추가

PaperMod의 검색은 별도 페이지가 필요합니다.

`content/search.md`:
```yaml
---
title: "검색"
layout: "search"
placeholder: "검색어를 입력하세요"
---
```

`content/archives.md`:
```yaml
---
title: "아카이브"
layout: "archives"
url: "/archives/"
---
```

## 5단계: 첫 글 작성

```bash
hugo new posts/hello-world.md
```

생성된 파일을 열어서 내용을 작성합니다:

```yaml
---
title: "첫 번째 포스트"
date: 2026-03-31
draft: false          # true면 빌드에 포함되지 않음
tags: ["블로그"]
categories: ["일반"]
summary: "첫 글입니다."
---

여기에 마크다운으로 글을 작성합니다.
```

> **draft 주의:** `draft: true`이면 `hugo` 빌드 시 포함되지 않습니다. 로컬에서 확인하려면 `hugo server -D` (-D는 draft 포함 옵션)를 사용하세요.

## 6단계: 로컬에서 확인

```bash
hugo server -D
```

브라우저에서 `http://localhost:1313/blog/`에 접속하면 사이트를 볼 수 있습니다. 파일을 수정하면 실시간으로 반영됩니다.

## 7단계: GitHub Pages 배포

### 7-1. GitHub 저장소 만들기

[GitHub](https://github.com)에서 새 저장소를 만들거나, CLI로 생성합니다:

```bash
gh repo create blog --public --source=. --remote=origin
```

### 7-2. GitHub Actions 워크플로우

`.github/workflows/deploy.yml` 파일을 만듭니다:

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive  # 테마 submodule 포함

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> **이 워크플로우가 하는 일:**
> 1. main 브랜치에 push하면 자동 실행
> 2. Hugo를 설치하고 사이트를 빌드
> 3. 빌드 결과를 GitHub Pages에 배포

### 7-3. Pages 설정

GitHub 저장소 → **Settings** → **Pages**에서:
- **Source**를 **GitHub Actions**로 변경

### 7-4. Push하면 끝

```bash
git add .
git commit -m "블로그 초기 설정"
git push -u origin main
```

몇 분 후 `https://your-username.github.io/blog/`에서 블로그를 확인할 수 있습니다.

## 앞으로의 워크플로우

이제 글을 쓸 때마다 이 흐름만 반복하면 됩니다:

```bash
# 1. 새 글 작성
hugo new posts/my-new-post.md

# 2. 로컬에서 확인
hugo server -D

# 3. 배포
git add .
git commit -m "새 글 추가"
git push
```

push만 하면 GitHub Actions가 알아서 빌드하고 배포합니다.

## 마무리

Hugo + PaperMod + GitHub Pages 조합은 무료이면서도 충분히 강력합니다. 처음에는 설정할 것이 많아 보이지만, 한번 세팅해 두면 마크다운으로 글만 쓰면 되니까 매우 편합니다.

다음 글에서는 블로그를 더 개인화하는 방법을 다뤄보겠습니다.
