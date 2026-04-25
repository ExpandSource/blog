---
title: "Gitea를 터미널에서 — tea CLI 설치와 활용 가이드"
date: 2026-04-25
draft: false
tags: ["gitea", "tea", "cli", "git", "self-hosted", "devops"]
categories: ["devops", "tools"]
description: "Gitea 공식 CLI 툴 tea 설치부터 로그인, 레포지토리·이슈·PR 관리까지. 셀프호스팅 Gitea를 터미널에서 다루는 방법"
---

[이전 글](/posts/gitea-self-hosting-guide)에서 Docker Compose로 Gitea 서버를 올리는 방법을 다뤘다. 서버가 떴으면 이제 웹 UI 말고 터미널에서 다루는 방법을 알아볼 차례다.

GitHub을 쓸 때 `gh` CLI가 있는 것처럼, Gitea에는 **tea**라는 공식 CLI 툴이 있다. 레포지토리 생성, 이슈 조회, PR 열기, 릴리즈 생성까지 터미널 안에서 처리할 수 있다. 브라우저를 열지 않아도 되니 워크플로우가 훨씬 빨라진다.

## tea가 뭔가

tea는 Gitea 팀이 공식으로 관리하는 CLI 클라이언트다.

- Gitea 인스턴스에 로그인해 API 토큰으로 인증
- 레포지토리, 이슈, PR, 릴리즈, 알림 등을 터미널에서 조작
- 여러 Gitea 인스턴스를 동시에 등록하고 전환 가능
- `gh` CLI와 사용법이 비슷해서 GitHub 사용자에게 익숙한 편

GitHub.com 공식 `gh`가 GitHub에 묶여 있다면, tea는 셀프호스팅 Gitea 인스턴스에 연결한다는 점이 다르다.

## 설치

### 바이너리 직접 설치 (Linux)

[Gitea 공식 코드 저장소](https://gitea.com/gitea/tea/releases)에서 최신 릴리즈를 받는다.

```bash
# 아키텍처 확인
uname -m
```

`x86_64`라면 `tea-linux-amd64`를, ARM이라면 `tea-linux-arm64`를 받는다.

```bash
# 예시: linux-amd64
wget https://gitea.com/gitea/tea/releases/download/v0.9.2/tea-linux-amd64 -O tea
chmod +x tea
sudo mv tea /usr/local/bin/
```

설치 확인:

```bash
tea --version
```

### Homebrew (macOS / Linux)

```bash
brew tap gitea/tap https://gitea.com/gitea/homebrew-gitea
brew install gitea/tap/tea
```

## 로그인 — Gitea 인스턴스 연결

설치 후 가장 먼저 할 일은 Gitea 서버에 로그인하는 것이다.

```bash
tea login add
```

대화형 프롬프트가 뜬다.

```
URL of Gitea instance:
```

여기에 Gitea 서버 주소를 입력한다. 이전 글 기준으로 설정했다면 `https://git.yourdomain.com` 또는 `http://서버IP:3000`.

```
Name of the login (only used locally):
```

여러 인스턴스를 등록할 때 구분하는 이름이다. `mygitea` 같이 짧게 쓰면 된다.

```
Do you have an access token?
```

토큰으로 인증할지 묻는다. 토큰이 없으면 `n`을 입력하면 계정/비밀번호로 발급해준다.

**토큰을 직접 발급하는 방법:**

Gitea 웹 UI → 우상단 프로필 → **Settings → Applications → Generate Token**에서 발급.  
권한은 `repository`, `issue`, `user` 정도면 충분하다.

```bash
tea login add --url https://git.yourdomain.com --token YOUR_TOKEN --name mygitea
```

한 줄로 비대화형으로 등록하는 방법이다. 스크립트로 자동화할 때 편하다.

등록된 인스턴스 목록 확인:

```bash
tea login list
```

기본 인스턴스를 바꾸고 싶으면:

```bash
tea login default mygitea
```

## 레포지토리 관리

### 목록 조회

```bash
tea repo list
```

내 계정의 레포지토리 목록이 나온다.

### 레포지토리 생성

```bash
tea repo create --name my-project --description "설명" --private
```

`--private`을 빼면 퍼블릭으로 생성된다.

### 레포지토리 포크

```bash
tea repo fork username/repo
```

### 현재 디렉토리 기준으로 작업

`git remote`에 Gitea 인스턴스가 등록된 디렉토리 안에서 `tea` 명령을 실행하면 자동으로 해당 레포지토리를 컨텍스트로 잡는다. `-l mygitea`나 레포지토리 인자를 매번 붙일 필요가 없다.

## 이슈 관리

### 이슈 목록 조회

```bash
tea issue list
```

기본으로 열려 있는 이슈만 나온다. 닫힌 이슈까지 보려면:

```bash
tea issue list --state closed
```

### 이슈 생성

```bash
tea issue create --title "버그 제목" --body "재현 방법 설명"
```

대화형 에디터를 열고 싶다면:

```bash
tea issue create
```

제목과 본문을 입력하는 프롬프트가 순서대로 나온다.

### 이슈 상세 보기

```bash
tea issue view 42
```

이슈 번호를 넣으면 제목, 본문, 댓글 목록이 출력된다.

### 이슈 닫기

```bash
tea issue close 42
```

## PR 관리

### PR 목록 조회

```bash
tea pr list
```

### PR 생성

현재 브랜치 기준으로 PR을 연다.

```bash
tea pr create --title "기능 추가" --body "변경 내용 설명" --base main
```

`--base`는 머지 대상 브랜치다. 생략하면 기본 브랜치로 잡힌다.

### PR 체크아웃

리뷰할 때 유용하다.

```bash
tea pr checkout 15
```

PR 번호에 해당하는 브랜치를 로컬로 체크아웃한다.

### PR 머지

```bash
tea pr merge 15
```

머지 방식을 지정할 수 있다.

```bash
tea pr merge 15 --style squash
```

`merge`, `squash`, `rebase` 중 선택.

## 릴리즈 관리

### 릴리즈 목록

```bash
tea release list
```

### 릴리즈 생성

```bash
tea release create --tag v1.0.0 --title "v1.0.0" --note "첫 릴리즈"
```

파일을 첨부하려면:

```bash
tea release create --tag v1.0.0 --title "v1.0.0" --asset ./build/app.tar.gz
```

## 알림 확인

```bash
tea notifications
```

읽지 않은 알림 목록이 나온다.

```bash
tea notifications --all
```

전체 알림 조회.

## 알아두면 좋은 것들

### 여러 인스턴스 전환

회사 Gitea와 개인 Gitea를 같이 쓴다면 `-l` 플래그로 인스턴스를 지정한다.

```bash
tea repo list -l work-gitea
tea issue list -l personal-gitea
```

### 출력 포맷 지정

```bash
tea repo list --output json
```

`json`, `yaml`, `csv`, `table` 중 선택. 스크립트에서 파싱할 때는 `json`이 편하다.

### 페이지네이션

목록이 길면 `--limit`과 `--page`로 페이지를 나눌 수 있다.

```bash
tea issue list --limit 20 --page 2
```

### 설정 파일 위치

tea는 로그인 정보를 `~/.config/tea/config.yml`에 저장한다. 토큰이 평문으로 들어있으니 해당 파일 권한은 확인해두는 게 좋다.

```bash
chmod 600 ~/.config/tea/config.yml
```

## 마무리

tea는 Gitea 서버가 있다면 바로 붙여 쓸 수 있는 CLI다. 이슈를 만들거나 PR을 여는 정도는 브라우저보다 터미널이 훨씬 빠르다. 특히 레포지토리 디렉토리 안에서 git 명령과 함께 쓰면 컨텍스트 스위칭 없이 흐름을 이어갈 수 있다.

처음에는 `tea issue list`와 `tea pr create` 두 개만 써도 충분하다. 나머지 명령은 필요할 때 `tea --help`로 찾아가면 된다.

- 공식 저장소: [gitea.com/gitea/tea](https://gitea.com/gitea/tea)
- Gitea 서버 설치: [이전 글 참고](/posts/gitea-self-hosting-guide)
