---
title: "n8n 처음 시작하기: 코드 없이 만드는 자동화 워크플로우"
date: 2026-04-08
draft: false
tags: ["n8n", "automation", "workflow", "self-hosted", "no-code", "low-code"]
categories: ["automation", "tools"]
description: "n8n이 처음인 사람을 위한 입문 가이드. 설치부터 첫 워크플로우 실행까지 한 번에 따라하기"
---

반복적인 작업에 지친 적이 있다면 한 번쯤 "이거 자동화하면 안 되나?" 하는 생각을 해봤을 것이다. Zapier, Make(구 Integromat) 같은 서비스가 이 시장을 오래 차지해왔지만, 최근 몇 년 사이 오픈소스 진영에서 가장 주목받는 프로젝트가 바로 **n8n**이다.

n8n은 코드 없이 드래그 앤 드롭으로 워크플로우를 만들 수 있으면서도, 필요하면 JavaScript로 자유롭게 확장할 수 있다. 무엇보다 **셀프호스팅이 가능하다**는 점이 가장 큰 매력이다. 이 글은 n8n을 처음 접하는 사람을 위한 입문 가이드다.

## n8n이 뭔가

n8n(엔에이트엔, "nodemation"의 줄임말)은 오픈소스 워크플로우 자동화 도구다. 서로 다른 서비스와 API를 노드(node)로 연결해서 데이터가 흐르는 파이프라인을 만든다.

- **400개 이상의 통합 노드**: Slack, Gmail, Notion, Google Sheets, GitHub, OpenAI 등
- **셀프호스팅 가능**: 내 서버에서 직접 실행, 데이터 주권 확보
- **Fair-code 라이선스**: 개인/회사 내부용으로는 자유롭게 사용
- **코드 확장**: Function 노드에서 JavaScript로 자유롭게 로직 작성
- **AI 친화적**: LangChain 기반 AI 에이전트 노드 내장

Zapier가 "쉬운 대신 제한적"이라면, n8n은 "조금 더 배울 게 있는 대신 제한이 거의 없다"고 보면 된다.

## 설치하기

n8n을 실행하는 방법은 크게 세 가지다.

### 1. npx로 바로 실행 (가장 빠른 체험)

Node.js만 있으면 된다.

```bash
npx n8n
```

첫 실행 시 패키지를 다운로드하고 `http://localhost:5678`에서 웹 UI가 열린다. 아무 설치 없이 바로 써보고 싶다면 이 방법이 제일 간단하다.

### 2. Docker로 실행 (추천)

실사용에는 Docker가 가장 안정적이다.

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

볼륨을 마운트해두면 워크플로우와 자격증명이 컨테이너를 재시작해도 유지된다.

### 3. n8n Cloud

설치가 번거롭다면 [n8n.io](https://n8n.io)에서 제공하는 클라우드 버전을 쓸 수도 있다. 14일 무료 체험 후 유료다. 학습 용도면 셀프호스팅을 추천한다.

## 첫 실행과 계정 생성

브라우저에서 `http://localhost:5678`을 열면 소유자 계정 생성 화면이 뜬다. 이메일, 이름, 비밀번호를 입력하면 끝. 이 계정은 로컬에만 저장되니 외부 전송 걱정은 없다.

대시보드에 들어오면 왼쪽에 **Workflows**, **Credentials**, **Executions** 같은 메뉴가 보인다. 개념은 단순하다.

- **Workflow**: 만드는 자동화 파이프라인 하나
- **Credentials**: 외부 서비스 API 키, OAuth 토큰 등
- **Executions**: 워크플로우가 실제로 실행된 기록

## 핵심 개념: 노드와 트리거

n8n의 워크플로우는 **노드(Node)** 들의 연결로 이루어진다. 노드는 두 종류로 나뉜다.

1. **트리거 노드 (Trigger)**: 워크플로우의 시작점. 예를 들어 "매일 오전 9시에" (Schedule), "이메일이 오면" (Email Trigger), "웹훅이 호출되면" (Webhook) 등.
2. **일반 노드 (Regular)**: 데이터를 처리하거나 외부 서비스를 호출. HTTP Request, Slack, Google Sheets, Function 등.

데이터는 노드 사이를 **JSON 객체 배열** 형태로 흐른다. 이전 노드의 출력이 다음 노드의 입력이 되는 구조다.

## 첫 워크플로우 만들기: "매일 날씨 알림"

개념만 설명하면 감이 안 오니 실제로 하나 만들어보자. 매일 아침 특정 도시의 날씨를 가져와서 콘솔에 출력하는 워크플로우다.

### 1단계. 새 워크플로우 생성

왼쪽 메뉴에서 **Workflows → Add workflow**를 클릭한다. 빈 캔버스가 열린다.

### 2단계. Schedule Trigger 추가

캔버스 가운데의 **+** 버튼을 누르고 `Schedule Trigger`를 검색해서 선택한다.

- **Trigger Interval**: `Days`
- **Trigger at Hour**: `9`
- **Trigger at Minute**: `0`

이렇게 설정하면 매일 오전 9시에 워크플로우가 실행된다.

### 3단계. HTTP Request 노드 추가

Schedule Trigger의 오른쪽 **+**를 누르고 `HTTP Request`를 추가한다.

- **Method**: `GET`
- **URL**: `https://wttr.in/Seoul?format=j1`

`wttr.in`은 API 키 없이 바로 쓸 수 있는 날씨 API다. `?format=j1`은 JSON 응답을 요청하는 옵션이다.

노드 오른쪽 상단의 **Execute step**을 누르면 실제 응답이 오는지 바로 확인할 수 있다. n8n의 가장 큰 장점 중 하나가 **노드 단위 즉시 실행**이다. 만들면서 바로 테스트할 수 있다.

### 4단계. 데이터 가공 (Edit Fields 노드)

날씨 API 응답은 복잡하다. 필요한 값만 뽑아보자. HTTP Request 뒤에 `Edit Fields (Set)` 노드를 추가한다.

- **Mode**: `Manual Mapping`
- 새 필드 추가:
  - **Name**: `temperature`
  - **Value**: `={{ $json.current_condition[0].temp_C }}`
  - **Name**: `description`
  - **Value**: `={{ $json.current_condition[0].weatherDesc[0].value }}`

`{{ }}` 는 n8n의 표현식(expression) 문법이다. 이전 노드의 데이터에 접근할 때 사용한다.

### 5단계. 실행하기

캔버스 하단의 **Execute workflow** 버튼을 누르면 전체 워크플로우가 한 번 실행된다. 각 노드의 입출력을 확인할 수 있다.

마지막 노드에서 `temperature`와 `description`이 깔끔하게 나오면 성공이다. 이 뒤에 Slack 노드, Telegram 노드, 이메일 노드를 붙이면 실제 알림이 된다.

## 알아두면 좋은 것들

### 표현식(Expression) 문법

n8n에서 이전 노드 데이터를 참조하는 방법은 `{{ $json.필드명 }}` 이다.

- `$json`: 현재 아이템의 JSON 데이터
- `$node["노드이름"].json`: 특정 노드의 출력 참조
- `$now`, `$today`: 현재 시간/날짜
- 내부에서 JavaScript 표현식 전체가 동작한다: `{{ $json.price * 1.1 }}`

### Function 노드

드래그 앤 드롭으로 해결이 안 되는 복잡한 로직은 `Code` 노드(구 Function)에서 JavaScript로 처리할 수 있다.

```javascript
const items = $input.all();
return items.map(item => ({
  json: {
    ...item.json,
    processed: true,
    uppercased: item.json.name?.toUpperCase()
  }
}));
```

### Credentials 관리

Slack, Gmail, Notion 같은 서비스에 연결할 때는 **Credentials**에 미리 API 키나 OAuth 토큰을 등록한다. 여러 워크플로우에서 같은 자격증명을 재사용할 수 있고, 암호화되어 저장된다.

### Webhook 트리거

외부 서비스가 n8n을 호출하게 만들려면 `Webhook` 트리거를 사용한다. 고유 URL이 생성되고, 그 URL로 POST/GET 요청이 오면 워크플로우가 실행된다. GitHub 이벤트, 폼 제출, Stripe 결제 등 이벤트 기반 자동화의 핵심이다.

## 실전에서 써볼 만한 아이디어

처음 배우면 뭘 자동화할지 막막하다. 몇 가지 예시:

- **RSS 피드 → Notion 데이터베이스**: 관심 블로그 새 글을 자동 수집
- **Gmail 첨부파일 → Google Drive 자동 저장**
- **GitHub 이슈 생성 → Slack 알림**
- **OpenAI + 스프레드시트**: 행마다 AI로 요약 생성
- **매주 월요일 → 지난 주 지표 이메일 발송**
- **특정 키워드 트윗/뉴스 모니터링 → Telegram 전송**

## 마무리

n8n은 "조금만 배우면 웬만한 건 다 자동화할 수 있다"는 감각을 빠르게 준다. 처음에는 Schedule + HTTP Request + Slack 같은 단순한 조합으로 시작하고, 익숙해지면 Webhook, Function 노드, AI 에이전트까지 확장해보면 된다.

문서와 커뮤니티도 활발하다.

- 공식 문서: [docs.n8n.io](https://docs.n8n.io)
- 워크플로우 템플릿: [n8n.io/workflows](https://n8n.io/workflows)
- GitHub: [github.com/n8n-io/n8n](https://github.com/n8n-io/n8n)

일단 `npx n8n` 한 줄로 띄워놓고, 평소에 귀찮았던 반복 작업 하나를 골라서 워크플로우로 옮겨보는 게 가장 빠른 학습법이다.
