---
title: "내가 이해한 OAuth2.0"
description: "OAuth2.0의 개념과 동작 원리에 대해 알아보자"
date: 2023-08-10
update: 2023-08-10
tags:
  - OAuth2.0
---

> 이 글은 우테코 피움팀 크루 '[클린](https://github.com/hozzijeong)'가 작성했습니다.


## OAuth란?

현재 개발하고 있는 서비스에서 구글 캘린더에 접근을 한다고 가정을 해봅시다. 현재 접근하는 사용자의 구글 캘린더 정보를 얻고자 한다면 구글 계정으로 로그인을 해야합니다. 진짜 간단한 방법이 있습니다. 사용자로부터 구글 id와 password를 받고 저희가 대신 로그인 해서 해당 사용자의 계정 정보를 받아오는 것입니다.

하지만 이 방법은 매우 위험합니다. 계정 정보가 탈취당할 수도 있고, 애초에 **대리 로그인**이라니 신개념 보이스피싱으로 오해받을 수도 있습니다.

제일 괜찮은 방법은 구글 자체에서 로그인 서비스를 대체할 수 있는 방식을 만들어서 제공을 하는 것입니다. 실제로  과거에 AuthSub라는 자체 개발한 방법을 통해 구현을 했지만 이 정형화 되지 않은 방법은 다양한 문제를 야기할 수 있었습니다. 흔히 카카오 로그인, 구글 로그인, 깃헙 로그인 등등 여러가지의 방식이 존재하는데 형식이 통일되어 있지 않다면 이 모든 것들을 각각 구현해서 사용을 해야하는 문제점이 있습니다. 

이러한 문제점을 해결하기 위해 나타난 개념이 OAuth입니다. 초기 버전인 1.0은 범위문제, 불확실한 역할, 보안 문제등으로 인해서 많이 사용 되지 않고 OAuth 2.0이 통상적으로 많이 사용 되고 있습니다. 

> 💡 **인증과 인가**: 인증은 요청자의 신원을 확인하는 것이고, 인가는 신원이 확인된 어느 사람이 권한이 있는지를 확인하는 것입니다. 개인적으로 생각했을때 인증은 사람의 직업 검증하는 것이고 인가는 그 사람의 직책을 통해 권한을 접근한다고 생각합니다. (직책에 따른 정보 접근 권한이 다르기 때문)


## OAuth 2.0

OAuth2.0을 이해하기 위해서는 크게 4가지 용어를 알고 있으면 됩니다. 바로 Resource Owner, Client, Resource Server, Authorization Server입니다. 각각의 역할과 설명은 다음과 같습니다.

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/c65c8e80-b04a-4b51-b66a-dbdd6d43b543)

- `Resource Owner`**:** 리소스(정보)의 주인. 즉,서비스를 이용하고자 하는 사용자를 의미합니다.
- `Client`: 리소스(정보)를 가공 혹은 Resource Owner에게 제공하는 주체. OAuth 2.0을 개발하는 어플리케이션(서비스)을 의미한다.
- `Resource Server`: 리소스(정보)를 갖고 있는 주체. 흔히 카카오, 구글, Github등 사용자 정보를 제공하는 주체를 의미한다.
- `Authorization Server`: Resource Owner를 인증하고, Client에 인가를 할당하는 역할이다.

OAuth를 가장 많이 도입한 기능 중 하나는 소셜 로그인입니다. 하나의 예시를 들어보겠습니다. 사용자가 Todo앱에 접근하려고 합니다. 사용자의 Todo인지를 확인하기 위해서는 로그인 기능이 필요합니다. 그런데 사용자는 회원가입을 하기 귀찮습니다. 이 과정에서 사용자의 기본 정보 (이메일, 닉네임, 프로필 사진 등)를 갖고 있는 소셜 네트워크가 있다고 생각해 봅시다. Client는 이 소셜 네트워크를 통해서 사용자에게 간단한 회원가입/로그인 기능을 제공할 수 있습니다. 이제 본인이 Todo앱에 회원가입을 하고, 나의 Todo를 카카오 스토리에 공유한다고 상상하고 OAuth의 동작과정에 대해 한번 알아보겠습니다.

> 💡 OAuth에는 크게 4가지 grant(승인 방식)가 있는데, 이번에 다루는 승인 방식은 기본적으로 사용되고 있는 **Authorization Code Grant(권한 부여 승인 코드 방식)** 을 통해 기준으로 설명하겠습니다.

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/e4f90a4b-6783-4d32-974a-2fdc2c5d3ae3)

- 첫 번째는 로그인 요청입니다. 흔히 우리가 알고있는 “카카오로 로그인하기”를 클릭 했을 때 나오는 설정되는 방식입니다. 이때 Authorization Server에 4가지 파라미터를 동봉해서 보냅니다.
    - client_id(required): Server에서 발급받은 인증 id입니다.
    - redirect_uri(optional): Server에 등록할 때 함께 입력한 redirect_uri입니다.
    - response_type(required): 반드시`code`로 설정해야 합니다.
    - scope(optional): 제공하는 정보 범위를 설정합니다.
- Authorization Server에서 Resource Owner에게 아이디와 비밀번호를 요구합니다. 사용자 인증이 된다면, 미리 입력받은 redierct_uri로 이동합니다.
- 해당 uri로 리다이렉트 되면서 함께 동봉되어 온 Authorization Code를 받고 이 코드를 Access Token으로 교환합니다.
- 교환한 Access Token을 통해서 Resource Server에서 원하는 데이터를 얻을 수 있습니다. (Todo일정을 카카오 스토리에 공유할 수 있습니다!!)

위와 같은 과정을 통해서 OAuth를 통해서 제 3의 서비스에서 사용자의 정보에 인가를 받아서 가공하거나 직접적으로 제공할 수 있습니다.

사용자로부터 로그인 정보를 받아서 로그인을 하고 사용자의 권한을 부여받은 Access Token을 통해서 Reource Server로부터 데이터를 얻는 것입니다.

---

### OAuth공부를 하면서 들었던 의문점

Access Token을 통해서 Resource에 있는 데이터를 얻는 것은 알겠는데, 그래서 이게 로그인이랑 어떤 관련이 있는가? 그리고, Client DB에 있는 데이터나 API랑도 관련이 있나?

Authorization Server를 통해서 넘겨받은 Access Token을 활용하면 소셜 로그인을 구현할 수 있었고, 해당 토큰을 통해서 모든 정보에 접근이 가능하겠구나 생각을 했었습니다. 하지만, 이 Access Token은 Resource Server에 있는 데이터를 얻는데 필요한 Access Token이고 **Client 내부에서 접근하는 DB와 API랑은 관련이 없다는 것입니다.**

Client에서는 넘겨받은 Access Token을 통해서 개별적으로 구축한 member DB에서 해당 유저를 식별하고, 그에 맞는 로그인 기능을 구현해야 합니다.

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/e6c1a44b-6e6e-4034-bca6-cfe9e2c5d8c0)

로그인 기능을 개별적으로 구현해야 한다는 얘기를 듣고, Client의 역할을 어떻게 분리가 되는지에 대해 의문이 들었습니다. 여기서 Client는 사용자가 이용하는 앱 서비스인데, 사용자와 직접적인 인터렉션을 하는 Client(FE)와 데이터를 제공하는 Sever(BE)로 분리가 됩니다. 

그렇다면 OAuth를 이용한 서비스를 구현할 때 Client안에서는 어떤 일이 일어날까요?

1. 사용자 로그인 페이지 제공 (FE)
2. redirect_uri로 이동시키고, 넘겨받은 Autorization Code를 BE로 넘겨줌 (FE)
3. FE로부터 넘겨받은 Autorization Code를 Authorization Server에 전달하고, Access Token과 Refresh Token을 받음(BE)
4. 넘겨 받은 Access Token을 통해 Resource Owner 정보 조회(BE)
5. DB Table에 해당 유저 확인 후 Client Session이나 Client Token 전달(BE)
6. 전달 받은 Session이나 Token을 통해서 유저 로그인 유지 (FE)
7. 사용자가 Client의 서비스를 이용하고자 할 때 Client Session을 통해 인증을 함 (FE)
8. Client에서 제공하고자 하는 데이터가 Resource Server에서 얻어야 하는 데이터라면, 저장해 놓았던 Access Token을 통해 Resource Server로 부터 데이터를 받아옴. 그리고 Client에서 제공하는 API와 잘 조합해서 데이터 제공(BE)
9. 사용자가 확인 가능하게 UI제공 (FE)

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/a668a660-8b62-4fbd-8caf-e3998c570a1c)

Client에서의 역할을 나눠보면 다음과 같습니다.

### FE

- 로그인 페이지 제공
- redirect_uri를 통해 Authorization Code 전달
- Resource Server에 맞는 UI 제공

### BE

- 전달받은 Authorization Code를 통해 Access Token 교환
- Access Token을 통해 Resource Server에 있는 데이터 요청 및 가공
- Access Token만료 시 Refresh Token으로 Access Token 갱신

---

### 참조

[OAuth의 인증 종류](https://blog.naver.com/mds_datasecurity/222182943542)

[OAuth의 개념과 동작 원리](https://hudi.blog/oauth-2.0/)
