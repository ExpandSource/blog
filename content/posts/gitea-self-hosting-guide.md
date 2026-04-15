---
title: "Gitea 셀프호스팅 서버 설치하기 — 나만의 Git 서버 구축"
date: 2026-04-15
draft: false
tags: ["gitea", "self-hosted", "git", "docker", "devops"]
categories: ["devops", "tools"]
description: "Docker Compose로 Gitea를 셀프호스팅하는 방법. 설치부터 SSH/HTTPS 설정, Nginx 리버스 프록시까지 한 번에 정리"
---

GitHub, GitLab에 코드를 올리는 게 당연해졌지만, 상황에 따라 직접 Git 서버를 운영하고 싶을 때가 있다. 외부에 올리기 곤란한 사내 코드, 인터넷이 안 되는 폐쇄망 환경, 또는 그냥 내 데이터를 내 서버에서 관리하고 싶은 이유로. 이럴 때 가장 먼저 꺼내볼 만한 선택지가 **Gitea**다.

Gitea는 설치가 간단하고 메모리를 거의 안 먹는다. Raspberry Pi 같은 저사양 기기에서도 잘 돌아간다. 이 글은 Docker Compose로 Gitea를 설치하고, SSH와 Nginx 리버스 프록시까지 붙이는 과정을 정리한다.

## Gitea가 뭔가

Gitea는 Go로 작성된 경량 오픈소스 Git 서비스다. GitHub과 유사한 웹 인터페이스를 제공하면서도 단일 바이너리로 동작한다.

- **경량**: 최소 RAM 64MB, 권장 128MB. GitLab과 비교하면 비교가 안 될 정도로 가볍다
- **셀프호스팅 친화적**: Docker, 단일 바이너리, 패키지 매니저 등 다양한 설치 방법 지원
- **GitHub 유사 기능**: 이슈, PR, 위키, Actions(CI/CD), 패키지 레지스트리 포함
- **MIT 라이선스**: 제한 없이 상업적으로도 사용 가능
- **낮은 의존성**: 기본 설정은 SQLite만으로 동작, 필요하면 MySQL/PostgreSQL로 교체 가능

GitLab이 "기능은 많지만 무겁다"라면, Gitea는 "핵심 기능만 있지만 가볍고 빠르다"는 쪽이다. 팀 규모가 크지 않거나 단순히 코드 저장소만 필요하다면 Gitea가 더 현실적인 선택이다.

## 준비물

- Docker, Docker Compose가 설치된 서버 (Linux 권장)
- 외부 접근이 필요하다면 도메인 혹은 고정 IP
- (선택) Nginx — 리버스 프록시를 붙이려면 필요

Docker 설치가 안 되어 있다면 먼저 진행한다.

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

로그아웃 후 재로그인하면 `docker` 명령을 sudo 없이 쓸 수 있다.

## Docker Compose로 설치 (추천)

### 1단계. 디렉토리 준비

```bash
mkdir -p ~/gitea && cd ~/gitea
```

### 2단계. docker-compose.yml 작성

```yaml
version: "3"

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    environment:
      - USER_UID=1000
      - USER_GID=1000
    volumes:
      - ./data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"   # 웹 UI
      - "2222:22"     # SSH (호스트 22번 포트가 비어 있으면 "22:22"도 가능)
```

데이터베이스를 SQLite가 아닌 PostgreSQL로 쓰고 싶다면 서비스를 추가한다.

```yaml
version: "3"

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea_password
    volumes:
      - ./data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"
    depends_on:
      - db

  db:
    image: postgres:16
    container_name: gitea_db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_PASSWORD=gitea_password
      - POSTGRES_DB=gitea
    volumes:
      - ./postgres:/var/lib/postgresql/data
```

### 3단계. 컨테이너 실행

```bash
docker compose up -d
```

로그 확인:

```bash
docker compose logs -f gitea
```

`Gitea: Running` 같은 메시지가 뜨면 정상이다.

## 초기 설정

브라우저에서 `http://서버IP:3000`을 열면 설치 마법사가 나온다.

**주요 설정 항목:**

- **데이터베이스 타입**: SQLite3 (간단), PostgreSQL (운영 권장)
- **사이트 URL**: 외부에서 접근할 URL을 입력한다. 나중에 바꾸기 번거로우니 처음부터 정확히.
  - 로컬 전용이면 `http://localhost:3000`
  - 도메인이 있으면 `https://git.yourdomain.com`
- **SSH 서버 포트**: docker-compose에서 `2222`로 매핑했으면 여기도 `2222`
- **관리자 계정**: 설치 마법사 하단에서 바로 생성 가능. 나중에도 만들 수 있다.

설정을 마치고 "Gitea 설치"를 누르면 잠시 후 로그인 화면으로 넘어간다.

## SSH 설정

SSH로 git push/pull을 하려면 클라이언트에서 공개키를 등록해야 한다.

### 키 생성 (이미 있으면 건너뜀)

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

### Gitea에 공개키 등록

우상단 프로필 → **Settings → SSH / GPG Keys → Add Key** 에서 `~/.ssh/id_ed25519.pub` 내용을 붙여넣는다.

### SSH 접속 확인

docker-compose에서 포트를 `2222:22`로 매핑한 경우:

```bash
ssh -T git@서버IP -p 2222
```

`Hi username! You've successfully authenticated` 메시지가 나오면 정상이다.

### clone/push 사용

Gitea 웹에서 레포지토리 주소를 복사할 때 SSH 탭을 선택한다. 포트가 22가 아닌 경우 주소가 `ssh://git@서버IP:2222/user/repo.git` 형식으로 나온다.

`~/.ssh/config`에 등록해두면 편하다.

```
Host gitea
  HostName 서버IP
  User git
  Port 2222
  IdentityFile ~/.ssh/id_ed25519
```

등록 후에는 `git clone gitea:user/repo.git` 형태로 쓸 수 있다.

## Nginx 리버스 프록시 설정

도메인을 붙이고 HTTPS를 쓰려면 Nginx와 Let's Encrypt를 연결한다.

### Nginx 설정 파일

`/etc/nginx/sites-available/gitea`:

```nginx
server {
    listen 80;
    server_name git.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name git.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/git.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/git.yourdomain.com/privkey.pem;

    client_max_body_size 100m;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

### Let's Encrypt 인증서 발급

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d git.yourdomain.com
```

이메일 입력, 약관 동의 후 자동으로 인증서가 발급되고 Nginx 설정도 업데이트된다. 인증서는 90일마다 자동 갱신된다.

## 알아두면 좋은 것들

### app.ini로 세부 설정

Gitea의 모든 설정은 `./data/gitea/conf/app.ini`에 저장된다. 웹 마법사 이후에도 이 파일을 직접 수정해서 설정을 바꿀 수 있다.

자주 쓰는 설정 몇 가지:

```ini
[server]
DOMAIN           = git.yourdomain.com
ROOT_URL         = https://git.yourdomain.com/
SSH_DOMAIN       = git.yourdomain.com
SSH_PORT         = 2222

[service]
DISABLE_REGISTRATION = true   ; 가입 차단 (초대제로 운영할 때)

[repository]
DEFAULT_PRIVATE = true        ; 새 레포지토리를 기본 private으로
```

수정 후 컨테이너를 재시작해야 적용된다.

```bash
docker compose restart gitea
```

### 백업

데이터는 모두 `./data` 디렉토리 안에 있다. 이 폴더만 백업하면 된다.

```bash
docker compose stop gitea
tar -czf gitea-backup-$(date +%Y%m%d).tar.gz ./data
docker compose start gitea
```

또는 Gitea 내장 백업 명령을 쓸 수도 있다.

```bash
docker exec -it gitea bash -c "gitea dump -c /data/gitea/conf/app.ini"
```

### 업데이트

```bash
docker compose pull
docker compose up -d
```

`latest` 태그 대신 버전 고정을 원하면 `image: gitea/gitea:1.22` 처럼 태그를 명시한다.

## 마무리

Gitea는 Docker Compose 파일 하나로 10분 안에 띄울 수 있다. GitHub Actions와 유사한 Gitea Actions도 지원하기 때문에 CI/CD까지 내재화하고 싶다면 Actions Runner도 함께 설치해볼 만하다. 처음에는 SQLite + HTTP로 간단하게 시작하고, 팀이 붙거나 데이터가 쌓이면 PostgreSQL + HTTPS로 올리는 것이 자연스러운 순서다.

- 공식 문서: [docs.gitea.com](https://docs.gitea.com)
- GitHub: [github.com/go-gitea/gitea](https://github.com/go-gitea/gitea)
- Docker Hub: [hub.docker.com/r/gitea/gitea](https://hub.docker.com/r/gitea/gitea)
