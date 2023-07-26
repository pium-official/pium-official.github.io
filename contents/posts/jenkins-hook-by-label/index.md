---
title: "PR 라벨로 젠킨스 빌드유발을 구분하기"
description: "PR의 라벨로 젠킨스의 빌드 유발을 구분해봅니다."
date: 2023-07-25
update: 2023-07-25
tags:
  - 젠킨스
  - CD
---

> 이 글은 우테코 피움팀 크루 '[주노](https://github.com/Choi-JJunho)'가 작성했습니다.

## 서론

현재 피움팀의 젠킨스에서 빌드 트리거로 `GitHub hook trigger for GITScm polling`을 사용하고있다.

![](.index_images/b12e07f3.png)

해당 설정은 Github에서 유발되는 모든 WebHook에 대해 빌드 트리거가 동작하는 설정이다.

![](.index_images/ee2e6825.png)

빌드 스크립트에서 위처럼 triggers 설정을 별도로 지정할 수 있지만 현재 프로젝트 구조상 더 세분화된 설정이 필요한 상태다.

![](.index_images/7049c9ec.png)

현재 레포지토리 하나에 프론트, 백엔드 코드가 같이 관리되고있다.
PR이 Merge 되었을 때 각 트랙별로 코드가 빌드 및 배포될 수 있도록 구성해야한다.

## 어떻게 할까?

현재 하나의 레포지토리에서 두개의 코드가 관리되고 있기 떄문에 단순히 develop 브랜치의 변경감지로는 어느 트랙의 작업내용인지 구분하기가 어렵다.

![](.index_images/b1e2e898.png)

현재 피움은 라벨을 이용하여 트랙별 작업구분을 하고 있다.

이를 이용하여 라벨별로 빌드, 배포를 진행할 수 있도록 구성해보자.

## Generic Webhook Trigger 설치

![](.index_images/cce2a05b.png)

Generic Webhook Trigger는 젠킨스에서 제공하는 플러그인으로 HTTP 요청을 수신하여 JSON, XML 형태의 데이터를 추출하여 트리거를 지정할 수 있는 플러그인이다.

Generic Webhook Trigger를 사용하여 빌드 유발 상황을 구분해보자.

![](.index_images/a035c7cc.png)

플러그인을 설치했다면 파이프라인의 Build Triggers 설정에서 다음과 같은 탭을 확인 할 수 있다.

![](.index_images/43109b24.png)

해당 설정을 클릭해보면 다음과 같은 설명을 확인할 수 있다.

`http://JENKINS_URL/generic-webhook-trigger/invoke` 로 오는 요청에 대해 트리거가 수행된다는 말이 적혀져있다.

위 설명에 맞춰 GitHub에서 WebHook 설정을 변경해주자.

> 위에서 설명한 탭에 대해서는 아직 저장하지않고 깃허브로 넘어와서 진행합니다.
> 
> 젠킨스 설정은 아래에서 설명하는 `Github WebHook 설정`이후 이어서 진행합니다.

## Github WebHook 설정

![](.index_images/1862f65e.png)

`프로젝트 레포지토리` -> `Settings` -> `Webhooks 탭`에 들어와서 웹훅을 새로 만들어보자.

![](.index_images/d3d6b68d.png)

Payload URL을 위에서 확인한 `Generic Webhook Trigger`의 설정에 맞게 작성한다.

그리고 트리거가 동작하는 이벤트에 대해서 `Send me everything` 옵션을 체크한다.

![](.index_images/0aae3bb6.png)

Add webhook을 한 뒤 다시 등록한 웹 훅을 클릭하여 `Recent Deliveries` 탭을 확인해본다.

![](.index_images/ee606f5e.png)

내용을 보면 요청을 보낼 때 Payload에 다양한 정보들이 담겨있는 것을 확인할 수 있다.

### (이해하기) JSONPath 살펴보기

> 해당 과정은 원리를 이해하기 위한 과정으로 수행하지 않아도 무관합니다.

![](.index_images/5ed35347.png)
![](.index_images/37132730.png)

PR과 연관된 요청이 발생했을 때는 Payload에 pull_request에 대한 정보가 추가된다.

![](.index_images/257a3485.png)

https://jsonpath.com 에 접속하여 해당 Payload를 붙여넣어 확인해본다.

JSONPath로 표현했을 때 `$.pull_request.merged`를 확인하면 해당 요청이 PR이 merge된 요청인지 확인할 수 있다.

![](.index_images/7ca45a4f.png)

`$.pull_request.base.ref`는 해당 PR의 base 브랜치를 의미한다.

예를들어 feature -> develop 형태라면 develop이 나온다

![](.index_images/8aa86e4c.png)

`$.pull_request.labels..name`는 해당 PR의 라벨들의 이름을 의미한다.

## Generic Webhook Trigger 설정

> ![](.index_images/a11771d7.png)
파이프라인 설정 구성에 들어와서 작업을 이어나갑니다.

![](.index_images/f13f0a48.png)

기존에 설정해둔 `GitHub hook trigger for GITScm polling` 설정을 해제한다.

![](.index_images/56956f06.png)

그리고 Generic WebHook Trigger 를 활성화한다.

### Post content parameters 설정

`Post content parameters`의 추가 버튼을 눌러 파라미터를 추가한다.

![](.index_images/355d4b02.png)

`$.pull_request.merged`를 등록하여 Merge 되었는지 여부를 가져온다.

![](.index_images/cbb47ad1.png)

`$.pull_request.base.ref`를 등록하여 base 브랜치를 가져온다.

원하는 값은 develop 브랜치가 될 것이다.

![](.index_images/3abd8938.png)

`$.pull_request.labels..name`를 등록하여 라벨들의 이름을 가져온다.

원하는 값은 등록된 라벨들 중에 `백엔드`를 포함하는 값이 존재하는 상황이 될 것이다.

### Optional filter 설정

아래로 내리다보면 Optional filter 탭을 볼 수 있다.

![](.index_images/67d90f0b.png)

Expression에 다음과 같은 정규표현식을 작성했다.

`(?=.*true)(?=.*develop)(?=.*백엔드).*`

또한 Text 부분에는 위에서 파라미터로 선언한 값들을 사용했다.

위 표현식으로 다음과 같은 효과를 기대할 수 있다.

`merge가 true 이고`, `브랜치가 develop 이고`, `라벨에 백엔드가 존재`하는 작업에 대해 트리거를 수행하는것을 기대할 수 있다.

### Token 설정

![](.index_images/fc1d455f.png)

Token 탭에 토큰 이름을 작성한다.

![](.index_images/0a7fa9b2.png)

이 때 GitHub WebHook 설정에 들어가서 Payload URL에 token 파라미터를 포함하도록 수정한다.

![](.index_images/9ed80db7.png)

바로 아래 탭에서 TokenCredential에 토큰에 대한 Secret Key를 추가해줘야한다.

외부에서 빌드를 유발하기 위해서는 token을 확인하는 방식이 존재하는데 이 때 Jenkins의 Credential에 있는 정보를 기반으로 수행한다.

> token을 사용하지 않는 경우 계정정보를 입력할 수도 있지만 해당 과정에서는 token을 등록하는 방식을 사용합니다.

![](.index_images/f4b07e6c.png)

위와 같은 정보로 Secret text를 작성하고 등록하고 적용한다.

## 응답 확인하기

![](.index_images/ba5532ba.png)

GitHub의 WebHook 탭의 Recent Deliveries 탭으로 가서 확인해보면 Response Body값에 다음과 같은 응답이 담겨있는 것을 확인할 수 있다.

```json
{
  "jobs": {
    "pium-dev": {
      "regexpFilterExpression": "(?=.*true)(?=.*develop)(?=.*백엔드).*",
      "triggered": true,
      "resolvedVariables": {
        "BRANCH": "develop",
        "LABELS": "[\"?️ \\b리팩터링\",\"? 백엔드\"]",
        "LABELS_0": "?️ \b리팩터링",
        "LABELS_1": "? 백엔드",
        "MERGED": "true"
      },
      "regexpFilterText": "true develop [\"?️ \\b리팩터링\",\"? 백엔드\"]",
      "id": 248,
      "url": "queue/item/248/"
    }
  },
  "message": "Triggered jobs."
}
```

`regexpFilterText`에 원하는 값이 잘 들어간 것을 확인할 수 있다.

## 결론

젠킨스의 `Generic Webhook Trigger` 플러그인을 이용하여 Merge 여부, target branch, label 값을 비교하고 빌드를 유발하는 설정을 했다.

경우에 따라 세부적인 검증을 추가할 수 있을 것 같다.

### 트러블 슈팅

Generic Webhook Trigger 설정 - Token 설정에서 많이 헤맸다.

![](.index_images/87d14459.png)

`{"jobs":null,"message":"Did not find any jobs with GenericTrigger configured! If you are using a token, you need to pass it like ...trigger/invoke?token=TOKENHERE. If you are not using a token, you need to authenticate like http://user:passsword@example.org/generic-webhook... "}`

404 에러와 함께 위 메시지가 반환되었는데 Jenkins의 Token에 대해 이해를 하지 못해 생긴 문제였다.

> 왜 다들 Token이 뭔지는 설명을 안해주는거지..?? ㅠㅠ
> 
> `외부에서 빌드를 유발하기 위해서는 token을 확인하는데 이 때 Jenkins의 Credential에 있는 정보를 기반으로 수행한다.`
> 
> 이 한마디를 이해하기 위해 하루가 걸렸다... 😇

## Reference

### 설정 과정에서 참고한 글

https://bepoz-study-diary.tistory.com/385

https://velog.io/@zayson/Jenkins-CICD-4.-WebHook%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EC%9E%90%EB%8F%99-%EB%B0%B0%ED%8F%AC


### 트러블 슈팅 과정에서 참고한 글

https://georgik.rocks/jenkins-generic-webhook-plugin-failed-with-did-not-find-any-jobs-with-generictrigger-configured/

https://pikachu987.tistory.com/61