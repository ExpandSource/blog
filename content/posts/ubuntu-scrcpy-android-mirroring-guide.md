---
title: "Ubuntu에서 scrcpy로 Android 화면 미러링하기 — 설치부터 무선 연결, 한글 입력까지"
date: 2026-04-21
draft: false
tags: ["ubuntu", "android", "scrcpy", "linux", "adb", "미러링"]
categories: ["linux", "tools"]
description: "Ubuntu에서 scrcpy 설치하고 Android 화면을 PC로 미러링하는 방법. USB 연결부터 무선 전환, 한글 입력 오류 해결, alias 설정까지 정리"
---

Android 폰을 PC에서 조작해야 할 일이 생각보다 많다. 앱 테스트, 화면 녹화, 키보드로 긴 문자 입력하기, 혹은 폰을 꺼내기 귀찮을 때. 이럴 때 쓸 만한 도구가 scrcpy다.

scrcpy는 USB 또는 Wi-Fi로 연결된 Android 기기의 화면을 PC에 실시간으로 띄워주는 오픈소스 툴이다. 별도 앱 설치 없이 ADB만 있으면 된다. 이 글은 Ubuntu 기준으로 설치부터 실제 사용까지 정리한 것이다.

## scrcpy가 뭔가

- Android 기기 화면을 PC 창으로 미러링, 마우스·키보드로 조작 가능
- USB 또는 TCP/IP(Wi-Fi) 연결 지원
- 낮은 레이턴시 (USB 기준 보통 35~70ms)
- Android 측에 별도 앱 설치 불필요, ADB 통신만 사용
- 오픈소스 ([GitHub](https://github.com/Genymobile/scrcpy))

블루스택 같은 에뮬레이터와 다르게 실제 내 기기를 그대로 쓴다는 점이 핵심이다. 앱 환경이 완전히 동일하다.

## 사전 준비 — Android 기기 설정

scrcpy가 동작하려면 Android 기기에서 USB 디버깅을 활성화해야 한다.

**개발자 옵션 켜기:**

1. 설정 → 휴대폰 정보 → 소프트웨어 정보
2. **빌드 번호**를 7번 연속으로 탭
3. "개발자가 되었습니다" 메시지 확인

**USB 디버깅 활성화:**

1. 설정 → 개발자 옵션
2. **USB 디버깅** 토글 ON

기기마다 메뉴 경로가 조금씩 다르다. 삼성은 "소프트웨어 정보"가 "휴대폰 정보" 안에 있고, 픽셀은 "기기 정보" → "빌드 번호" 순이다.

## 설치

공식 문서에서 권장하는 방법은 설치 스크립트를 사용하는 것이다.

```bash
curl -1sLf 'https://dl.cloudsmith.io/public/scrcpy/scrcpy/setup.deb.sh' | sudo -E bash
sudo apt install scrcpy
```

설치가 끝나면 버전 확인으로 정상 설치됐는지 본다.

```bash
scrcpy --version
```

`adb`도 같이 설치되어 있어야 한다. 없다면:

```bash
sudo apt install adb
```

## 첫 연결 — USB 필수

무선 연결은 처음에 USB를 한 번 거쳐야 한다. 처음부터 Wi-Fi만으로 연결하려 하면 안 된다.

**USB 케이블로 기기를 연결한 뒤:**

```bash
adb devices
```

처음 연결 시 폰 화면에 "USB 디버깅을 허용하시겠습니까?" 팝업이 뜬다. **허용** 탭. "이 컴퓨터에서 항상 허용"에 체크하면 다음부터는 팝업 없이 연결된다.

다시 `adb devices`를 실행했을 때 아래처럼 나오면 정상이다.

```
List of devices attached
XXXXXXXXXXXXXXXX    device
```

`unauthorized` 상태면 폰 화면의 팝업을 아직 허용 안 한 것이다. `offline` 이면 케이블이나 USB 디버깅 설정을 다시 확인한다.

이제 scrcpy를 실행한다.

```bash
scrcpy
```

폰 화면이 PC 창으로 뜬다. 마우스로 클릭, 드래그가 가능하고 키보드 입력도 된다.

## 무선 연결

USB를 뽑고 Wi-Fi로 전환하는 방법이다. 폰과 PC가 같은 네트워크에 있어야 한다.

**ADB TCP/IP 모드 활성화:**

USB가 연결된 상태에서:

```bash
adb tcpip 5555
```

**폰 IP 확인:**

```bash
adb shell ip route | awk '{print $9}'
```

또는 폰의 설정 → Wi-Fi → 연결된 네트워크 상세 정보에서 IP 주소 확인.

**무선으로 연결:**

```bash
adb connect 192.168.1.xxx:5555
```

IP는 실제 폰 IP로 바꾼다. 연결되면:

```
connected to 192.168.1.xxx:5555
```

이제 USB를 뽑아도 된다. 이후부터 scrcpy를 실행하면 무선으로 연결된다.

```bash
scrcpy
```

무선 연결은 USB보다 레이턴시가 높다. 같은 공유기에 5GHz로 연결하면 체감상 괜찮은 편이다.

## 한글 입력 오류 대처

기본 설정으로 scrcpy를 쓰다 보면 한글 입력이 이상하게 동작하는 경우가 있다. 자음과 모음이 분리되거나, 글자가 제대로 조합되지 않는 문제다.

해결법은 간단하다.

```bash
scrcpy -K
```

`-K` 옵션은 `--prefer-text` 플래그로, 키 입력 방식을 변경해서 한글 IME가 정상적으로 동작하게 한다. 이 옵션 하나로 한글 입력 문제가 대부분 해결된다.

## alias 설정

매번 `-K` 옵션을 붙여서 실행하기 번거롭다면 alias로 등록해두는 게 편하다.

`~/.bashrc` 또는 `~/.zshrc`에 추가:

```bash
alias scrcpy='scrcpy -K'
```

저장 후 적용:

```bash
source ~/.bashrc
# 또는
source ~/.zshrc
```

이후 `scrcpy`만 입력해도 `-K` 옵션이 자동으로 붙는다.

자주 쓰는 옵션을 더 묶고 싶다면 이런 식으로 쓸 수 있다.

```bash
# 해상도 제한 + 한글 입력 + 화면 항상 위
alias scrcpy='scrcpy -K --max-size 1080 --always-on-top'
```

## 애플리케이션 메뉴 등록 (desktop 파일)

터미널 없이 앱 메뉴에서 바로 실행하고 싶다면 `.desktop` 파일을 만들면 된다.

```bash
nano ~/.local/share/applications/scrcpy.desktop
```

아래 내용을 입력한다.

```ini
[Desktop Entry]
Name=scrcpy
Comment=Android Screen Mirror
Exec=scrcpy -K
Icon=phone
Terminal=false
Type=Application
Categories=Utility;
```

저장 후 실행 권한 부여:

```bash
chmod +x ~/.local/share/applications/scrcpy.desktop
```

이제 앱 메뉴(Activities 또는 앱 서랍)에서 "scrcpy"를 검색하면 바로 실행할 수 있다. `Icon=phone`은 시스템 기본 아이콘이고, 커스텀 아이콘이 있다면 절대 경로로 바꿔주면 된다.

## 알아두면 좋은 옵션들

| 옵션 | 설명 |
|------|------|
| `-K` | 한글 입력 정상화 (`--prefer-text`) |
| `--max-size 1080` | 미러링 해상도 제한 (부하 감소) |
| `--no-audio` | 오디오 미러링 비활성화 |
| `--always-on-top` | scrcpy 창을 항상 위에 |
| `--turn-screen-off` | 폰 화면은 끄고 PC에서만 조작 |
| `--record screen.mp4` | 화면 녹화 |
| `-b 8M` | 비트레이트 설정 (기본 8Mbps) |

`--turn-screen-off`는 배터리 절약에 좋다. PC에서 제어하면서 폰 화면은 꺼두는 방식이다.

## 마무리

scrcpy는 Android 미러링 툴 중에서 설치가 가장 가볍고 안정적인 편이다. ADB 위에서 동작하기 때문에 별도 계정이나 클라우드 연동이 없고, 오픈소스라 믿고 쓸 수 있다. 한글 입력 문제는 `-K` 하나로 해결되니 기본 alias에 넣어두는 걸 추천한다.

공식 문서와 GitHub 저장소에 더 많은 옵션이 정리되어 있다.

- [공식 GitHub](https://github.com/Genymobile/scrcpy)
- [Linux 설치 가이드](https://github.com/Genymobile/scrcpy/blob/master/doc/linux.md)
