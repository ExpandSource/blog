---
title: "Claude Code Statusline 설정하기: 모델·컨텍스트·사용량을 한눈에"
date: 2026-04-10
draft: false
tags: ["claude-code", "claude", "ai", "terminal", "productivity"]
categories: ["tools", "ai"]
description: "Claude Code 하단에 모델명, 컨텍스트 사용량, 5h/7d rate limit을 표시하는 statusline을 Mac/Linux/Windows에 설정하는 방법"
---

Claude Code를 터미널에서 쓰다 보면 슬슬 궁금해지는 것들이 있다. 지금 컨텍스트가 얼마나 찼는지, 오늘 5시간 limit이 얼마나 남았는지, 어떤 모델이 붙어 있는지. 매번 확인하러 나가기도 번거롭고, 그렇다고 모르고 쓰다 갑자기 리셋되면 당황스럽다.

**Statusline**은 Claude Code 화면 하단에 고정 표시되는 커스텀 바다. 쉘 스크립트 하나로 원하는 정보를 실시간으로 띄울 수 있다. 이 글은 Mac/Linux/Windows 각각의 설정 방법을 단계별로 정리한다.

## Statusline이 뭔가

Statusline은 `settings.json`에 커맨드를 등록하면 동작한다. Claude Code가 컨텍스트 변경, 비용 업데이트, 세션 전환 같은 이벤트가 발생할 때마다 등록한 커맨드를 실행하고, 그 출력을 화면 하단에 표시한다.

- **stdin으로 JSON 수신**: Claude Code가 세션 데이터를 JSON 형태로 스크립트에 넘겨준다
- **출력값 그대로 표시**: 스크립트가 출력하는 텍스트가 statusline에 그대로 나온다 (멀티라인 가능)
- **ANSI 컬러 지원**: 색상 코드를 넣으면 터미널에서 색상이 적용된다
- **로컬 실행**: API 토큰을 소모하지 않는다
- **자동 숨김**: 자동완성, 권한 요청 프롬프트가 뜰 때는 잠시 가려진다

## 준비물

### jq 설치

bash 스크립트에서 JSON을 파싱할 때 `jq`를 쓴다. 없으면 설치부터 한다.

```bash
# Mac
brew install jq

# Ubuntu/Debian
sudo apt-get install jq

# Windows (Chocolatey)
choco install jq
```

`jq --version`으로 설치 확인.

### settings.json 위치

Claude Code 설정 파일은 `~/.claude/settings.json`이다. 없으면 새로 만들면 된다. 프로젝트별로 적용하려면 `.claude/settings.json` (프로젝트 루트 기준)을 사용한다.

## Mac / Linux 설정

### 1단계. 스크립트 파일 생성

`~/.claude/statusline.sh`를 만든다.

```bash
#!/bin/bash
input=$(cat)

MODEL=$(echo "$input" | jq -r '.model.display_name')
PCT=$(echo "$input" | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
FIVE_H=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
SEVEN_D=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // empty')

# 컨텍스트 바 생성
FILLED=$((PCT / 10))
BAR=""
for i in $(seq 1 10); do
  if [ "$i" -le "$FILLED" ]; then
    BAR="${BAR}#"
  else
    BAR="${BAR}-"
  fi
done

# rate limit 파트 (Pro/Max 플랜에서만 표시)
RATE=""
if [ -n "$FIVE_H" ]; then
  FIVE_H_INT=$(echo "$FIVE_H" | cut -d. -f1)
  RATE=" | 5h: ${FIVE_H_INT}%"
fi
if [ -n "$SEVEN_D" ]; then
  SEVEN_D_INT=$(echo "$SEVEN_D" | cut -d. -f1)
  RATE="${RATE} 7d: ${SEVEN_D_INT}%"
fi

echo "[${MODEL}] ctx: ${BAR} ${PCT}%${RATE}"
```

실행 권한을 부여한다.

```bash
chmod +x ~/.claude/statusline.sh
```

### 2단계. settings.json에 등록

```json
{
  "statusLine": {
    "type": "command",
    "command": "/Users/your-username/.claude/statusline.sh"
  }
}
```

경로는 절대경로로 써야 한다. `~` 같은 쉘 확장은 동작하지 않는 경우가 있으니 풀 경로를 쓰는 게 안전하다.

### 3단계. 확인

Claude Code를 재시작하거나 새 세션을 열면 하단에 statusline이 표시된다. 첫 API 응답이 오기 전까지는 일부 값이 `null`로 나올 수 있다. rate limit은 Pro/Max 플랜에서만 표시된다.

## Windows 설정

Windows에서는 Claude Code가 Git Bash를 통해 스크립트를 실행한다. PowerShell 스크립트를 쓰려면 Git Bash에서 `powershell.exe`를 호출하는 래퍼를 만들면 된다.

### 방법 1: Git Bash 스크립트 (jq 사용)

Mac/Linux와 동일한 bash 스크립트를 Git Bash에서 그대로 쓸 수 있다. jq를 Windows에 설치했다면 동일하게 동작한다.

`C:\Users\your-username\.claude\statusline.sh`:

```bash
#!/bin/bash
input=$(cat)

MODEL=$(echo "$input" | jq -r '.model.display_name')
PCT=$(echo "$input" | jq -r '.context_window.used_percentage // 0' | cut -d. -f1)
FIVE_H=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
SEVEN_D=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // empty')

RATE=""
[ -n "$FIVE_H" ] && RATE=" | 5h: $(echo "$FIVE_H" | cut -d. -f1)%"
[ -n "$SEVEN_D" ] && RATE="${RATE} 7d: $(echo "$SEVEN_D" | cut -d. -f1)%"

echo "[${MODEL}] ctx: ${PCT}%${RATE}"
```

`settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "C:\\Users\\your-username\\.claude\\statusline.sh"
  }
}
```

### 방법 2: PowerShell 직접 호출

```json
{
  "statusLine": {
    "type": "command",
    "command": "powershell.exe -NoProfile -Command \"$d = $input | ConvertFrom-Json; Write-Host \\\"[$($d.model.display_name)] ctx: $([int]$d.context_window.used_percentage)%\\\"\""
  }
}
```

인라인으로 쓰면 이스케이프가 복잡해진다. 긴 스크립트는 `.ps1` 파일로 분리하는 게 낫다.

## 사용 가능한 필드 정리

스크립트에서 `$input` JSON에 접근할 수 있는 주요 필드다.

| 필드 | 설명 |
|------|------|
| `model.display_name` | 현재 모델 이름 |
| `model.id` | 모델 ID |
| `context_window.used_percentage` | 컨텍스트 사용률 (0~100) |
| `context_window.remaining_percentage` | 컨텍스트 잔여율 |
| `context_window.context_window_size` | 최대 컨텍스트 크기 (기본 200000) |
| `rate_limits.five_hour.used_percentage` | 5시간 limit 사용률 |
| `rate_limits.seven_day.used_percentage` | 7일 limit 사용률 |
| `rate_limits.five_hour.resets_at` | 5시간 limit 리셋 시각 (unix timestamp) |
| `cost.total_cost_usd` | 세션 누적 비용 (USD) |
| `cost.total_duration_ms` | 세션 경과 시간 (ms) |
| `workspace.current_dir` | 현재 작업 디렉토리 |

`rate_limits`는 Pro/Max 플랜 구독자에게만 첫 API 응답 이후 표시된다. 값이 없을 때를 대비해 `// empty` 또는 `// 0`으로 기본값 처리를 해두는 게 좋다.

## 심화: refreshInterval

git 상태처럼 외부에서 변경되는 데이터를 반영하려면 `refreshInterval`을 추가한다.

```json
{
  "statusLine": {
    "type": "command",
    "command": "/Users/your-username/.claude/statusline.sh",
    "refreshInterval": 10
  }
}
```

`refreshInterval`의 단위는 초이고 최솟값은 1이다. 너무 짧게 설정하면 스크립트가 자주 실행되니 필요한 만큼만 설정한다.

## Statusline 제거

더 이상 필요 없으면 `/statusline delete` 커맨드를 실행하거나, `settings.json`에서 `statusLine` 필드를 직접 삭제한다.

## 마무리

Statusline 하나만 달아도 Claude Code를 쓰는 느낌이 달라진다. 컨텍스트가 얼마나 차 있는지 항상 보이니까 중간에 `/compact` 타이밍을 더 잘 잡게 되고, rate limit 수치를 보면서 페이스를 조절하게 된다. 스크립트는 jq 문법만 알면 원하는 대로 필드를 조합할 수 있으니 자신의 작업 흐름에 맞게 커스텀해보면 된다.

- 공식 문서: [code.claude.com/docs/en/statusline](https://code.claude.com/docs/en/statusline)
