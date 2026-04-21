---
title: "Ubuntu VM에서 Logitech 마우스 인식시키기 — Proxmox USB Passthrough부터 Solaar 권한 문제 해결까지"
date: 2026-04-21
draft: false
tags: ["ubuntu", "logitech", "proxmox", "solaar", "self-hosted", "linux"]
categories: ["devops", "linux"]
description: "Proxmox VM 우분투에서 Logi Bolt 수신기를 인식시키는 방법. USB 패스스루 설정부터 Solaar sudo 권한 문제 해결까지 시행착오 포함"
---

Proxmox 위에 Ubuntu VM을 올려서 쓰다 보면 한 가지 불편한 점이 생긴다. 마우스나 키보드 같은 USB 장치가 호스트에 붙어있어서 VM 안에서는 보이지 않는다는 것. 특히 로지텍 MX Master 같은 마우스를 Logi Bolt 수신기로 연결해서 쓴다면 설정이 하나 더 필요하다.

처음에는 단순하게 생각했다. Proxmox에서 USB 장치 추가하고 Solaar 설치하면 끝 아닌가. 실제로는 블루투스 착각, sudo 권한 문제, udev 규칙 누락까지 서너 번 헛발질을 했다. 이 글은 그 과정을 정리한 것이다.

## Logi Bolt가 뭔가 — 블루투스가 아니다

첫 번째 함정이 여기 있다. Logi Bolt 수신기를 꽂고 Ubuntu 설정에서 블루투스 메뉴를 열었는데 아무것도 안 뜬다. 그래서 "블루투스가 꺼져 있는 건가?" 하고 토글을 켜보려 했지만 켜지지도 않았다.

당연한 결과다. **Logi Bolt는 블루투스가 아니다.**

- Logi Bolt와 Logitech Unifying 수신기는 독자적인 2.4GHz RF 프로토콜을 사용한다
- 수신기가 OS에 전달하는 방식은 `"나는 USB 마우스다"` — 블루투스 어댑터처럼 등록되지 않는다
- 따라서 Ubuntu 설정의 블루투스 메뉴는 완전히 무관하다. 건드릴 필요 없다

블루투스가 아니기 때문에 별도 드라이버 없이도 커서 이동 정도는 바로 된다. 다만 배터리 상태 확인, DPI 조절, 제스처 설정 같은 기능은 별도 툴이 필요하다.

## 1단계. Proxmox USB Passthrough 설정

VM은 기본적으로 호스트 하드웨어를 직접 보지 못한다. Logi Bolt 수신기를 VM으로 넘겨줘야 한다.

**Proxmox 웹 UI 기준:**

1. 해당 VM 선택 → **Hardware** 탭
2. **Add → USB Device** 클릭
3. **Use USB Vendor/Device ID** 선택
4. 목록에서 `Logitech, Inc. Logi Bolt Receiver` 찾아 추가
5. VM 재시작

### USB 3.0 호환성 문제

여기서 하나 주의할 점이 있다. USB 장치 추가 시 **USB3 체크박스를 해제**하는 것이 안전하다. xHCI(USB 3.0) 컨트롤러와 로지텍 수신기가 가끔 충돌을 일으킨다. USB 2.0 모드로 넘기면 인식률이 올라간다.

패스스루가 제대로 됐는지는 Ubuntu 터미널에서 확인한다.

```bash
lsusb
```

출력 중에 아래처럼 로지텍 장치가 보이면 성공이다.

```
Bus 001 Device 003: ID 046d:c548 Logitech, Inc. Logi Bolt Receiver
```

드라이버까지 붙었는지 확인하려면:

```bash
lsusb -t
```

`Driver=usbhid` 가 해당 장치 옆에 있어야 한다. 없다면 Ubuntu가 장치를 어떻게 처리할지 모르는 상태다.

## 2단계. Solaar 설치

리눅스에서 로지텍 수신기를 관리하는 도구는 여러 가지가 있지만, Bolt와 Unifying 수신기를 모두 지원하면서 GUI까지 제공하는 **Solaar**가 가장 범용적이다.

```bash
sudo apt update
sudo apt install solaar -y
```

설치 후 udev 규칙을 로드한다. 이 단계를 빠뜨리면 권한 문제가 생긴다.

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## 3단계. 시행착오 — sudo로는 되는데 일반 실행이 안 된다

Solaar를 앱 메뉴에서 실행했는데 창이 뜨긴 하지만 수신기가 안 보이고 **Pair New Device** 버튼도 없는 상황이 있다. `lsusb`에는 분명히 장치가 잡히는데.

이럴 때 먼저 확인해볼 것:

```bash
sudo solaar
```

`sudo`로 실행하니 수신기가 바로 보이고 마우스도 연결된다면, **권한 문제**다. 일반 사용자 계정이 USB 입력 장치에 직접 접근할 권한이 없는 것이다.

원인은 두 가지 중 하나다.

1. Solaar 설치 시 udev 규칙 파일이 제대로 생성되지 않았다
2. 현재 사용자가 `plugdev` 그룹에 속하지 않는다

## 4단계. udev 규칙 수동 적용 + 그룹 추가

### udev 규칙 파일 확인

```bash
ls /lib/udev/rules.d/42-logitech-unify-permissions.rules
```

파일이 없다면 Solaar 공식 레포에서 직접 가져온다.

```bash
sudo wget https://raw.githubusercontent.com/pwr-ml/Solaar/master/rules.d/42-logitech-unify-permissions.rules \
  -O /etc/udev/rules.d/42-logitech-unify-permissions.rules

sudo udevadm control --reload-rules
sudo udevadm trigger
```

### plugdev 그룹에 사용자 추가

```bash
sudo usermod -aG plugdev $USER
```

명령 실행 후 **로그아웃했다가 다시 로그인**해야 그룹 변경이 적용된다.

### 수신기 재삽입

규칙과 그룹을 바꾼 뒤에는 장치를 새로 인식시켜야 한다. Proxmox에서 접근한다면 Hardware 탭에서 USB 장치를 제거했다가 다시 추가하면 된다. 물리적으로 접근 가능하다면 수신기를 뽑았다가 다시 꽂는 것이 가장 확실하다.

이제 `sudo` 없이 실행해보자.

```bash
solaar
```

왼쪽 패널에 수신기가 뜨고 마우스가 연결되어 있으면 성공이다.

## Solaar에서 할 수 있는 것들

마우스가 연결되면 목록에서 클릭해서 아래 설정들을 조작할 수 있다.

- **배터리 잔량**: 상단 트레이 아이콘에서도 확인 가능
- **Smart Shift**: 스크롤 휠이 무한 모드로 전환되는 속도 조절
- **DPI (Sensitivity)**: 커서 속도 조절
- **Fn 키 전환**: F1~F12 기능 우선순위 변경

자동 시작을 설정해두면 매 부팅 시 백그라운드에서 돌면서 배터리를 모니터링한다. Solaar 창의 톱니바퀴 메뉴 → **Autostart Solaar** 체크.

## 도구 비교: Solaar vs LogiOps vs Piper

용도에 따라 더 적합한 도구가 있다.

| 도구 | 방식 | 추천 대상 |
|------|------|-----------|
| Solaar | GUI | 페어링·배터리 확인·기본 설정이 필요한 일반 사용자 |
| LogiOps | 설정 파일(daemon) | MX Master 제스처 버튼을 워크스페이스 전환 등에 매핑하고 싶은 파워 유저 |
| Piper | GUI | 로지텍 G 시리즈 게이밍 마우스 (DPI·RGB·버튼 매핑) |

MX Master 계열을 쓰면서 제스처 버튼까지 활용하고 싶다면, Solaar로 페어링만 해두고 실제 동작 정의는 LogiOps 설정 파일(`logid.cfg`)에서 하는 조합이 정석으로 통한다. LogiOps는 공식 레포에서 빌드하거나 PPA로 설치한다.

## 정리

Proxmox VM에서 Logi Bolt 수신기를 쓰려면 세 가지가 맞아야 한다.

1. **Proxmox USB Passthrough** — USB3 옵션을 끄고 Device ID 방식으로 패스스루
2. **Solaar 설치** + udev 규칙 로드
3. **권한 설정** — udev 규칙 파일 존재 확인 + `plugdev` 그룹 추가 후 재로그인

sudo로는 되는데 일반 실행에서 수신기가 안 보인다면 거의 100%가 3번 문제다. 위 순서대로 해결하면 된다.

- Solaar GitHub: [github.com/pwr-ml/Solaar](https://github.com/pwr-ml/Solaar)
- LogiOps GitHub: [github.com/PixlOne/logiops](https://github.com/PixlOne/logiops)
