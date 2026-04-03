---
title: "NanoClaw로 나만의 Claude 어시스턴트 만들기"
date: 2026-04-03
draft: false
tags: ["nanoclaw", "claude", "whatsapp", "ai", "homelab", "self-hosted"]
categories: ["ai", "homelab"]
description: "WhatsApp에서 @Claw를 부르면 Claude가 대답한다. 설치부터 채널 연결까지 NanoClaw 셋업 가이드"
---

ChatGPT나 Claude를 웹 브라우저에서만 쓰다가, 어느 순간 이런 생각이 든다. *그냥 카카오톡처럼 메신저에서 바로 물어보면 안 되나?*

NanoClaw는 그 아이디어를 현실로 만들어주는 오픈소스 프로젝트다. WhatsApp, Telegram, Slack 같은 메신저에서 트리거 키워드를 입력하면 Claude가 Docker 컨테이너 안에서 실행되고 응답한다. 클라우드 서비스에 대화 내용을 맡기지 않고, 내 서버에서 직접 돌린다.

## NanoClaw가 뭔가

GitHub: [qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw)

아키텍처를 한 줄로 요약하면 이렇다:

```
메신저 채널 → SQLite → 폴링 루프 → Docker 컨테이너 (Claude Agent SDK) → 응답
```

단일 Node.js 프로세스가 전체를 관리한다. 채널은 스킬로 추가하고, 메시지가 오면 컨테이너를 띄워서 Claude를 실행한 뒤 결과를 돌려보낸다.

보안 면에서 눈에 띄는 설계가 두 가지다:

1. **컨테이너 격리**: 에이전트는 명시적으로 마운트된 디렉터리만 볼 수 있다. 호스트 파일시스템에 직접 접근하지 못한다.
2. **크리덴셜 격리**: API 키가 컨테이너 안으로 들어가지 않는다. OneCLI 게이트웨이가 요청 시점에 인증을 주입한다.

## 필요한 것

- macOS, Linux, 또는 Windows (WSL2)
- Node.js 20+
- [Claude Code](https://claude.ai/download)
- Docker (macOS라면 Apple Container도 가능)
- Anthropic API 키

## 설치

### 1. 포크하고 클론

NanoClaw는 직접 클론해서 쓰는 게 아니라, **포크해서 내 레포로 만들어** 쓰는 방식이다. 커스터마이징이 코드 수정으로 이루어지기 때문이다.

```bash
# GitHub CLI가 있다면
gh repo fork qwibitai/nanoclaw --clone
cd nanoclaw

# 없다면
# 1. GitHub에서 qwibitai/nanoclaw 포크
# 2. git clone https://github.com/<내-아이디>/nanoclaw.git
# 3. cd nanoclaw
```

### 2. Claude Code 실행

```bash
claude
```

Claude Code CLI가 열리면, 여기서부터는 터미널 명령어가 아니라 `/` 로 시작하는 스킬 명령어를 사용한다.

### 3. /setup 실행

```
/setup
```

Claude Code가 대화하듯 진행한다. 의존성 설치, API 키 설정, Docker 컨테이너 빌드, 서비스 등록까지 안내해준다.

진행 과정에서 물어보는 것들:

- Anthropic API 키
- 어떤 채널을 쓸 것인지 (WhatsApp, Telegram 등)
- 트리거 키워드 (기본값: `@Andy`)
- 메인 채널 등록 (어시스턴트를 관리할 내 개인 채팅)

설치 중 문제가 생기면 Claude가 직접 진단하고 수정한다. 별도의 디버깅 도구가 없어도 된다.

---

## 채널 추가

`/setup`으로 기본 구성이 끝나면 채널을 붙인다.

### WhatsApp

```
/add-whatsapp
```

QR 코드 또는 페어링 코드로 WhatsApp 계정을 연결한다. 터미널에 QR 코드가 출력되면 스마트폰 WhatsApp → 연결된 기기 → QR 코드 스캔으로 인증한다.

연결 후 셀프 채팅(나에게 보내기)에서 `/register`로 메인 채널로 등록하면 관리 명령어를 쓸 수 있다.

### Telegram

```
/add-telegram
```

Bot Token을 발급받아 입력한다. WhatsApp과 달리 봇 계정으로 동작하므로 추가 기기 연결이 필요 없다.

### 그 외

- `/add-slack` — Slack 워크스페이스 연결
- `/add-discord` — Discord 서버 연결
- `/add-gmail` — Gmail 읽기/쓰기 연동

---

## 사용법

채널이 연결되면 그룹 채팅이나 개인 채팅에서 트리거 키워드로 호출한다.

```
@Andy 오늘 서울 날씨 알려줘
@Andy 이 이미지에 뭐가 있어? [이미지 첨부]
@Andy 매주 월요일 오전 8시에 해커뉴스 요약해서 알려줘
```

메인 채널(내 셀프 채팅)에서는 관리 명령도 가능하다:

```
@Andy 모든 그룹의 예약 작업 목록 보여줘
@Andy Family Chat 그룹에 합류해
```

---

## 스킬로 기능 확장

NanoClaw는 기능을 코어에 추가하는 대신 **스킬** 단위로 확장한다. 스킬은 Claude Code에서 실행하는 명령어로, 브랜치 병합 + 코드 수정을 자동으로 처리한다.

주요 스킬들:

| 스킬 | 설명 |
|------|------|
| `/add-image-vision` | WhatsApp 이미지 인식 (멀티모달) |
| `/add-voice-transcription` | 음성 메시지 텍스트 변환 (Whisper) |
| `/add-pdf-reader` | PDF 읽기 |
| `/add-compact` | 긴 대화 컨텍스트 압축 (`/compact` 명령) |
| `/add-ollama-tool` | 로컬 모델(Ollama) 연동 |
| `/add-reactions` | WhatsApp 이모지 반응 |

스킬 추가는 `claude` 세션 안에서 실행한다:

```
/add-image-vision
```

---

## 커스터마이징

NanoClaw에는 설정 파일이 없다. 동작을 바꾸고 싶으면 Claude Code에게 말하면 된다.

```
트리거 키워드를 @Claw로 바꿔줘
응답을 더 짧고 간결하게 해줘
아침 인사에 날씨 정보도 같이 줘
```

코드베이스가 작아서 Claude가 안전하게 수정할 수 있다.

---

## 서비스 관리

설치 후 NanoClaw는 백그라운드 서비스로 실행된다.

**Linux (systemd)**:

```bash
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
```

**macOS (launchd)**:

```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

문제가 생기면 `claude` 세션에서 `/debug`를 실행하면 된다.

---

## 마치며

NanoClaw의 철학은 "충분히 작아서 이해할 수 있는 코드"다. 수십만 줄짜리 프레임워크를 그대로 갖다 쓰는 게 아니라, 내가 직접 읽고 수정할 수 있는 크기로 유지한다.

설치 비용이 조금 있지만, 한 번 구성하면 메신저에서 바로 AI를 호출할 수 있고, 스킬로 기능을 계속 붙여나갈 수 있다. 데이터가 내 서버에만 머문다는 것도 장점이다.

공식 문서: [docs.nanoclaw.dev](https://docs.nanoclaw.dev)  
커뮤니티: [Discord](https://discord.gg/VDdww8qS42)
