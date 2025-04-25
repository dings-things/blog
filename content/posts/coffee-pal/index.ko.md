---
date: "2024-10-26"
draft: false
author: dingyu
categories: ["dx"]
tags:
- slack-bot
- coffee-chat
title: '[DX] 사내 커피 챗 슬랙 봇 개발기'
summary: 나 개발잔데 옆에 사람이랑 슬랙으로 대화한다.. 우리 이제 친해져요
cover:
    image: img/slack.png
    relative: true
comments: true
description: Developers often work alone, deeply diving into specific techniques. However, as members of a company, we need a space to discuss and brainstorm together. Inspired by the spirit of Ubuntu, I created my own Slack app Slack Bolt Coffee Chat, Coffee Pal
---

> 소스 코드 : https://github.com/dings-things/coffee_pal

# 목적
---
일을 하는 것에서 그치는 것이 아닌, 일을 잘 하고 싶은 조직에서 좋은 문화를 구성하는 것은 일의 생산성이나 품질에도 여러 이점을 가져온다

필자가 개발하고자 하는 "CoffeePal"은 문제에 막혀 길을 잃어버린 사람이나 소소한 스몰 토크로 일에도 능률을 키울 수 있는 그러한 앱을 만들 고자 `커피 친구`라는 의미에서 커피팔을 기획하게 되었다


# 개발에 앞서...
---
먼저 사내 커뮤니케이션 툴인 `Slack`에서 상호작용을 통해, **쉽고 빠르게** 커피 친구를 매칭해줘야 하는 요구사항이 있다

이에 앞서 간단하게 [Slack Workflow](https://slack.com/intl/ko-kr/features/workflow-automation)를 통하여, 전반적인 흐름을 파악해 본다

---

1. **Trigger Point 생성**
   테스트 용이기 때문에, Trigger는 `:커피:` 이모지로 반응하였을 때 워크플로가 실행 되도록 한다

2. **활성화 하기**
   반응한 메시지를 전송한 사용자에게 메시지를 보내어, 커피팔 매칭을 시작할 지 여부를 확인한다

3. **양식에서 정보 수집**
   개인화된 서비스를 위해 MBTI, 생년월일을 입력 받는다.
   해당 데이터의 기입은 Optional 하게 만든다

4. **대상, 일시, 주제를 지정한다**
   커피팔 대상과 커피 일정, 그리고 전반적인 주제를 사전에 정의한다.

5. **커피팔 대상에게 초대 메시지 보내기**
   앞서 지정한 데이터를 통해 커피팔에게 간략하게 초대 메시지를 보낸다

---
![](image.png)

---

![](image-1.png)


개인적으로 `Slack 워크플로`를 처음 사용해 보았는데, UI가 깔끔하고 사용성이 괜찮음을 느꼈다

다만, 사용자와 "상호작용"이라기 보다는, 일방적인 소통에 가깝기 때문에 slack app으로 확장 해보도록 한다

먹고 싶지 않은 음식을 억지로 먹여주는 느낌... 이랄까요?

![](image-2.png)


# 요구사항
---
1. 미리 저장된 데이터를 기반으로 사용자 기반의 매칭을 할 수 있어야 한다

   - 개인정보 입력 시, 매칭 시스템 Trigger

2. App을 사용한 상호작용을 위해 컴퓨팅 리소스를 확보한다
   - [x] 사내 서버 
      - 방화벽 관리 어려움
      - 소켓 통신을 통한 빠른 응답
   - [ ] AWS Lambda 
      - 프리티어 기준 무료
      - 서버리스
      - AWS API GW와 연계 가능
      - 프리티어 종료 후, 유지 비용 고려
3. 사용자에게 한눈에 봐도 알 수 있는 UI 제공
   - [Slack Block kit](https://app.slack.com/block-kit-builder) 활용


# 구축하기
---
## Lambda로 구축하기
- Slack APP 생성
- Lambda 설정
- Socket Mode 설정

### Slack APP 생성 1. App 생성하기

[slack API](https://api.slack.com/apps)에서 App 생성하기
![](image-3.png)


### Slack APP 생성 2. `From scratch`를 클릭
![](image-4.png)


### Slack APP 생성 3. 앱 workspace와 이름 지정하기
![](image-5.png)


### Slack APP 생성 4. 유저 상호작용을 위한 봇 설정
![](image-6.png)


### Slack APP 생성 5. 이벤트 활성화
![](image-7.png)


### Lambda 설정 1. [Lambda 페이지](https://ap-northeast-2.console.aws.amazon.com/lambda/home?region=ap-northeast-2#/begin)에서 함수 생성하기
![](image-8.png)


### Lambda 설정 2. 함수 설정, 런타임 및 아키텍처 설정
![](image-9.png)


### Lambda 설정 3. API Gateway 설정

API Gateway는 HTTP 요청을 Lambda 함수로 라우팅 할 수 있는 매우 유용한 트리거입니다. Slack Events API와 Lambda를 연동하기 위해서는 API Gateway를 통한 설정이 필요합니다.

![](image-10.png)

라고 생각하던 와중... 서버가 이미 있다면 굳이 람다로 구현을 해야하는가? 
이미 있는 서버를 안쓰는 것이 더 낭비가 아닌가? 🤔 고민에 빠졌습니다

서버는 남는 개발 서버를 활용하여 처리를 하고 소켓 모드로 일괄 처리하는 방안을 생각하게 되었습니다


## Socket Mode 설정
> **소켓 모드란?**
> 소켓 모드를 켜면 이러한 페이로드를 공개 HTTP 앤드포인트인 요청 URL로 보내는 대신 WebSocket 연결을 통해 앱의 상호 작용과 이벤트를 라우팅합니다.
> ![](https://velog.velcdn.com/images/wjddn3711/post/642277c3-713d-427d-9bad-79010dff6847/image.png)

앤드 포인트를 사용하지 않음으로서 여러 요청에 대한 보안적인 위협을 피할 수 있습니다.
또한 여러 Event에 대해 일일이 구독을 하지 않아도 됩니다


### 1. 애플리케이션 전역 토큰 생성
   전역 토큰에 권한을 부여 → 서버 내에서 권한을 사용하여 슬랙 제어
   ![width:500px](https://velog.velcdn.com/images/wjddn3711/post/bf79d0d7-806b-4b11-8b9c-76c3f5874c98/image.png)


### 2. 토큰의 Scope 지정
   ![bg right 90%](https://velog.velcdn.com/images/wjddn3711/post/a561cd66-446b-4f3b-8f8b-e189e9cedbd0/image.png)

- `channels:read` : 채널의 사용자 정보 취득
- `chat:write` : 채널에서 쓰기 권한
- `chat:write.public` : 채널에 invite 되어 있지 않아도 쓰기 권한
- `groups:read` : private 채널 정보 취득
- `groups:write` : private 채널에 쓰기 권한
- `im:write` : DM 발송 권한
- `mpim:write` : group DM 발송 권한
- `reminders:write` : 리마인더 설정 권한


## 개발
### Slack Bolt

> Slack 애플리케이션을 개발하기 위한 프레임워크
> 주로 Node.js, Python, JavaScript와 같은 언어로 작성되었으며, Slack 플랫폼과 통합된 애플리케이션을 쉽게 만들 수 있도록 도와줍니다.

굳이...? 싶지만...

안쓰면 불필요한 작업이 너무 많아집니다. 이는 곧 낮은 생산 속도, 유지 보수도 힘들어지죠 


#### 쓰지 않을 때
```python
from slack_sdk import WebClient
from slack_sdk.errors import SlackApiError
import os
import json
import base64
import urllib.parse

def lambda_handler(event, context):
    bot_token = os.getenv('BOT_KEY')
    # 토큰으로 인증
    client = WebClient(token=bot_token)
    # Base64 인코딩 여부 확인 및 디코딩
    if event.get('isBase64Encoded', False):
        try:
            # Base64 인코딩된 본문을 디코딩하여 문자열로 변환
            decoded_body = base64.b64decode(event['body']).decode('utf-8')
            event_body = urllib.parse.unquote_plus(decoded_body)[8:]
        except Exception as e:
            print(f"Error decoding base64 body: {e}")
            return {
                'statusCode': 400,
                'body': json.dumps({'message': 'Invalid base64 encoded body'})
            }
    ele:
        event_body = event['body']
    print(event_body)
    # 디코딩된 본문을 JSON으로 파싱
    try:
        event_data = json.loads(event_body)
    except json.JSONDecodeError:
        print("JSONDecodeError occurred. The body may not be in valid JSON format.")
        return {
            'statusCode': 400,
            'body': json.dumps({'message': 'Invalid request body'})
        }

    # Event 타입 확인
    event_type = event_data.get('event', {}).get('type')

    # Event 타입에 따라 분기로 처리
```

---

#### 쓸 때

```python
app = App(token=settings.SLACK_BOT_TOKEN)
logger = logging.getLogger(__name__)


# app_home_opened 이벤트 핸들러 등록
@app.event("app_home_opened")
def handle_app_home_opened(
    event: Union[
        str,
        Pattern,
        Dict[str, Optional[Union[str, Sequence[Optional[Union[str, Pattern]]]]]],
    ],
    client: WebClient = None,
) -> SlackResponse:
    ...

# 버튼 액션 핸들러 등록
@app.action("suggest_coffee_chat_button")
def handle_random_coffee_chat_button(
    ack, body, client: WebClient = None
) -> SlackResponse:
    ...

```


특정 이벤트 / 액션 / 뷰 를 디버깅 할 때에 Bolt를 활용하면 인증, event, action에 따른 분기 처리, 미들웨어 설정 등등...

서비스 로직을 수행하기전 거쳐야 했던 중복되는 과정들을 생략하고 서비스 로직에 집중할 수 있습니다.

- 유지보수 ↑
- 가독성 ↑


Bolt에서 제공되는 기능 중 크게 5가지를 사용하게 됩니다.

- middleware
  - 앱 전반의 로깅 및 인증 관리
- action
  - 버튼 클릭, Select 메뉴 등의 상호작용 시 발생
- view
  - 모달과 같은 복잡한 인터페이스를 생성 및 관리
- event
  - Slack 워크스페이스 내에서 발생하는 다양한 이벤트. 가령 채널에 참가 / 사용자의 메시지 전송 등등


### Block Kit

슬랙 앱 개발의 꽃은 역시 Block Kit이라 말할 수 있습니다.
![](image-11.png)


각 화살표는 Block을 나타내며 type에 따라 구분되는데 이를 Stacking 하여 하나의 View를 형성합니다.

ex.
``` python
HOME_OPENED = {
    "type": "home",
    "blocks": [
        ...
    ],
}
```

---

### HOME 구성 요소
![](image-12.png)



### Section
섹션을 활용하여 직관적인 UI로 사용자 경험을 최적화 합니다

가로 섹션
![](image-13.png)


---
세로 섹션
![](image-14.png)


---

### Input

Slack API는 Restful API 답게 Stateless (무상태성)을 띕니다.
각 요청이 독립적이며, 서버가 클라이언트의 이전 요청에 대한 정보를 유지하지 않는것이죠

Input 없이는 유저의 이전 View에서의 상태를 알 수 없기 때문에 상호작용이 불가합니다.
![](image-15.png)
![](image-16.png)

---

랜덤 커피챗을 예로 들어 보겠습니다
![](image-17.png)

---

서버가 만약 순간적인 단절이 생겼을 때에도
다시 요청 시, 이전 Input 값을 토대로 무상태성 요청을 보낼 수 있어

**안정성**이 보장됩니다


## 개선 사항
---
- Slack DataStore를 이용한 유저의 상태 값 저장
  - TTL을 지정하여 예약된 커피챗 일정의 조회 / 수정 / 삭제 기능
- 인사정보를 조회하여 나만의 맞춤형 커피팔 선정
  - 인사 API 권한을 얻어 직급 / 나이 / 직군 별 맞춤형 커피팔을 추첨하는 기능
- UI 개선
  - 딱 봐도 이거네~ 누구나 알기 쉽게 사용자 편의성을 고려한 View 구성
- CI/CD 및 인프라 구성
  - 허락을 구한다면.. 별도 인프라로 full time 사용가능한 앱 구성
  - CI CD 파이프라인으로 누구나 기여하여 새로운 기능을 추가하도록 오픈소스화!!


# 마치며
---
개발자로 살아오며 “우분투" 정신에 깊게 공감하고  있었습니다

> '우리가 함께 있기에 내가 있다(I am because you are)'

우리가 함께할 수 있는 환경을 구성하는 것도 중요하다 생각됩니다.
모든 이가 소통에 두려움을 느끼지 않고 "함께"가 되었으면 하는 마음에

#### 커피팔 을 만들게 되었습니다
![](image-18.png)

기술적인 어려움에 부딪히거나, 보다 긴밀한 협업을 위해 아이스 브레이킹이 필요하시다면 여러분도 슬랙봇을 만들어 보세요 :)

