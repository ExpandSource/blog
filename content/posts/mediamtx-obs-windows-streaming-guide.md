---
title: "MediaMTX + OBS로 내부 네트워크 라이브 스트리밍 구축하기 — Windows 기준"
date: 2026-05-09
draft: false
tags: ["mediamtx", "obs", "streaming", "rtmp", "rtsp", "hls", "windows", "homelab"]
categories: ["tools", "homelab"]
description: "MediaMTX를 Windows에 설치하고 OBS로 RTMP 송출, 같은 네트워크 안에서 VLC·브라우저로 수신하는 방법. 녹화 설정까지."
---

인터넷 라이브 스트리밍은 YouTube나 Twitch로 넘기면 그만이다. 그런데 내부 네트워크 안에서만 화면을 공유하거나, 방송 테스트 환경을 인터넷 없이 구성하고 싶을 때는 이야기가 달라진다. 외부 서비스를 끼면 지연이 생기고, 사내 정책상 외부로 내보낼 수 없는 화면도 있다.

이 글에서는 **MediaMTX**를 Windows에 미디어 서버로 올리고, **OBS**로 RTMP 송출 → 같은 네트워크 안의 기기에서 수신하는 흐름을 처음부터 설정한다. 스트림을 파일로 저장하는 녹화 설정도 같이 다룬다.

## MediaMTX가 뭔가

MediaMTX(구 rtsp-simple-server)는 Go로 작성된 경량 미디어 서버다. 별도 설치 없이 단일 실행 파일로 동작한다.

- **RTMP 수신** → RTSP / HLS / WebRTC / SRT 등 여러 프로토콜로 재송출
- 단일 바이너리, 의존성 없음. Windows에서도 exe 파일 하나로 실행
- 설정 파일 하나(yaml)로 경로별 스트림, 인증, 녹화를 모두 제어
- Nginx나 ffmpeg 없이 HLS를 내장 HTTP 서버로 바로 제공

Nginx-RTMP처럼 복잡한 설정이 필요 없고, ffmpeg 파이프라인을 별도로 구성하지 않아도 된다.

## 사전 준비

- **Windows 10 / 11** (64비트)
- **OBS Studio** 설치 완료 — [obsproject.com](https://obsproject.com)
- 방화벽에서 포트를 열 수 있는 권한
- 수신 기기(같은 Wi-Fi 또는 유선 LAN에 연결된 PC, 스마트폰 등)

## 1단계 — MediaMTX 다운로드 및 실행

[GitHub Releases](https://github.com/bluenviron/mediamtx/releases)에서 최신 버전을 받는다.

파일명 예시:
```
mediamtx_v1.x.x_windows_amd64.zip
```

압축을 풀면 두 파일이 나온다.

```
mediamtx.exe
mediamtx.yml
```

`mediamtx.exe`를 더블클릭하거나 터미널에서 실행한다.

```powershell
.\mediamtx.exe
```

기본 설정으로 다음 포트가 열린다.

| 프로토콜 | 포트 |
|----------|------|
| RTMP     | 1935 |
| RTSP     | 8554 |
| HLS      | 8888 |
| WebRTC   | 8889 |

처음 실행하면 콘솔에 다음처럼 출력된다.

```
INF MediaMTX v1.x.x
INF [RTMP] listener opened on :1935
INF [RTSP] listener opened on :8554
INF [HLS] listener opened on :8888
INF [WebRTC] listener opened on :8889
```

이 상태가 되면 서버가 준비된 것이다.

## 2단계 — Windows 방화벽 포트 열기

외부 기기에서 접속하려면 방화벽 인바운드 규칙을 추가해야 한다. PowerShell을 관리자 권한으로 열고 실행한다.

```powershell
# RTMP (OBS 송출용)
New-NetFirewallRule -DisplayName "MediaMTX RTMP" -Direction Inbound -Protocol TCP -LocalPort 1935 -Action Allow

# HLS (브라우저 수신용)
New-NetFirewallRule -DisplayName "MediaMTX HLS" -Direction Inbound -Protocol TCP -LocalPort 8888 -Action Allow

# RTSP (VLC 수신용)
New-NetFirewallRule -DisplayName "MediaMTX RTSP" -Direction Inbound -Protocol TCP -LocalPort 8554 -Action Allow
```

내부 네트워크에서만 쓴다면 `WebRTC(8889)`도 필요에 따라 같은 방식으로 추가한다.

## 3단계 — OBS 스트림 설정

OBS에서 MediaMTX로 RTMP 송출 설정을 한다.

1. OBS → **설정(Settings)** → **방송(Stream)**
2. 서비스(Service): **사용자 지정(Custom)**
3. 서버(Server):

```
rtmp://127.0.0.1/live
```

MediaMTX가 같은 PC에서 실행 중이면 `127.0.0.1`, 다른 PC에 올렸다면 그 PC의 IP를 쓴다.

4. 스트림 키(Stream Key): `stream` (임의로 정해도 된다)

결과적으로 스트림 주소는 `rtmp://127.0.0.1/live/stream`이 된다. 경로(path)가 `live/stream` 형태다.

**확인:** 설정 저장 후 OBS에서 **방송 시작**을 누르면 MediaMTX 콘솔에 다음 메시지가 출력된다.

```
INF [RTMP] [conn 127.0.0.1] is publishing to path 'live/stream'
```

이 메시지가 나오면 송출이 정상적으로 연결된 것이다.

## 4단계 — 수신 기기에서 스트림 시청

송출 서버의 IP를 확인한다.

```powershell
ipconfig
# IPv4 주소 확인. 예: 192.168.1.100
```

### VLC로 RTSP 수신

VLC → **미디어** → **네트워크 스트림 열기** → 주소 입력:

```
rtsp://192.168.1.100:8554/live/stream
```

### 브라우저로 HLS 수신

같은 네트워크의 어떤 기기에서든 브라우저 주소창에 입력:

```
http://192.168.1.100:8888/live/stream
```

HLS는 세그먼트 방식이라 VLC RTSP보다 수초 정도 지연이 더 크다. 지연을 줄이고 싶다면 **LL-HLS(Low Latency HLS)** 또는 **WebRTC**를 사용한다.

### WebRTC로 저지연 수신

```
http://192.168.1.100:8889/live/stream
```

브라우저에서 WebRTC 플레이어 페이지가 열린다. HLS보다 지연이 훨씬 짧다. 화상회의 수준(1초 미만)을 원하면 이 방식이 맞다.

## 5단계 — 스트림 녹화 설정

MediaMTX는 수신한 스트림을 그대로 파일로 저장할 수 있다. `mediamtx.yml`을 열고 `paths` 섹션을 수정한다.

```yaml
paths:
  live/stream:
    record: yes
    recordPath: ./recordings/%path/%Y-%m-%d_%H-%M-%S-%f
    recordFormat: fmp4
    recordSegmentDuration: 1h
```

설정 설명:

- `record: yes` — 이 경로의 스트림을 파일로 저장
- `recordPath` — 저장 경로. `%path`는 스트림 경로명, `%Y-%m-%d_%H-%M-%S-%f`는 시작 시각
- `recordFormat` — `fmp4`(MP4) 또는 `mpegts`(TS). 재생 호환성은 fmp4가 낫다
- `recordSegmentDuration` — 분할 간격. `1h`이면 1시간 단위로 파일이 나뉜다

설정 변경 후 MediaMTX를 재시작하면 다음 송출부터 `recordings/live/stream/` 폴더에 파일이 쌓인다.

## 알아두면 좋은 것들

### 경로를 여러 개 운용하기

`mediamtx.yml`에서 경로를 여러 개 정의하면 스트림을 분리할 수 있다.

```yaml
paths:
  cam1:
    record: yes
    recordPath: ./recordings/cam1/%Y-%m-%d_%H-%M-%S-%f
  cam2:
    record: no
  all:
```

OBS 스트림 키를 `cam1`으로 설정하면 `rtmp://서버IP/cam1`으로 송출된다.

### 인증 추가

외부에 노출될 가능성이 있다면 최소한 인증을 걸어둔다.

```yaml
authInternalUsers:
  - user: admin
    pass: yourpassword
    permissions:
      - action: publish
      - action: read
```

### 스트림 재인코딩 없이 중계

MediaMTX는 기본적으로 스트림을 재인코딩하지 않고 그대로 전달한다. CPU를 거의 쓰지 않는다. 저사양 미니PC나 서버에 올려도 무리없이 동작한다.

### 백그라운드 서비스 등록

매번 exe를 수동으로 실행하기 번거롭다면 Windows 서비스로 등록한다.

```powershell
sc.exe create mediamtx binPath= "C:\mediamtx\mediamtx.exe" start= auto
sc.exe start mediamtx
```

서비스로 등록하면 PC 부팅 시 자동으로 시작된다.

## 마무리

MediaMTX + OBS 조합은 구성이 단순하고 추가 의존성이 없다는 게 장점이다. OBS에서 RTMP로 밀어 넣으면 MediaMTX가 RTSP, HLS, WebRTC로 변환해 뿌려준다. 녹화도 설정 파일 몇 줄로 해결된다. 내부 네트워크 강의 녹화, 화면 공유, 방송 테스트 환경 구축에 바로 쓸 수 있다.

처음에는 기본 설정 그대로 실행해보고, 동작이 확인되면 `mediamtx.yml`에서 경로와 녹화 설정을 조금씩 추가해 나가는 식으로 확장하면 된다.

- MediaMTX 공식 저장소: [github.com/bluenviron/mediamtx](https://github.com/bluenviron/mediamtx)
- OBS Studio: [obsproject.com](https://obsproject.com)
