---
title: "AWS S3로 정적 이미지 배포하기"
description: "AWS S3로 정적 이미지 배포하기"
date: 2023-08-30
update: 2023-08-30
tags:
  - AWS
  - S3
  - CloudFront
---

> 이 글은 우테코 피움팀 크루 '[주노](https://github.com/Choi-JJunho)'가 작성했습니다.

## 서론

![img.png](.index_images/img.png)

피움 팀에서 비로그인 시 로그인을 수행하라는 페이지와 함께 이미지를 띄우려고 한다.
이 때 코드를 배포하지 않고도 이미지를 변경할 수 있는 방법이 없을까 생각했다.

현재 이미지를 프론트엔드의 assets에 넣어서 배포를 하게 된다면 이미지가 변경하나에도 배포가 이뤄져야한다는 번거로움이 존재한다.

CDN을 이용하여 이미지를 배포하고 해당 파일을 관리할 수 있다면 재배포하는 번거로움은 줄어들 것이다.

다시말해 `https://이미지경로.이미지이름.png`라는 경로는 동일하나 해당 서버에 있는 자원을 변경하면 이미지가 변경될 수 있다.

이러한 효과를 내기 위해 AWS의 S3를 이용해볼 수 있다.

## AWS S3란?

아마존 웹 서비스에서 제공하고있는 객체 스토리지 서비스(Simple Storage Service)다.
쉽게 말해 클라우드 환경에 저장소를 제공하는 서비스를 의미한다.

![img_1.png](.index_images/img_1.png)

## 시작하기

> [사례별로 알아본 안전한 S3 사용 가이드](https://techblog.woowahan.com/6217/)를 참고하여 작성된 글입니다.
개인 AWS 계정으로 실습을 진행하는점 참고 부탁드립니다~

지금부터 AWS S3를 이용해 정적 웹 호스팅을 수행해보자.

### 웹 호스팅 준비하기

![img_2.png](.index_images/img_2.png)

버킷 생성부터 시작해보자.

#### 버킷 생성

![img_3.png](.index_images/img_3.png)

우선 S3 서비스에 들어가서 버킷을 생성한다.

![img_4.png](.index_images/img_4.png)

`ACL 비활성화됨`, `모든 퍼블릭 엑세스 차단` 옵션을 선택해준다.

![img_5.png](.index_images/img_5.png)

기본 암호화는 `Amazon S3 관리형 키(SSE-S3)를 사용한 서버 측 암호화`를 선택하고 버킷을 생성한다.

![img_6.png](.index_images/img_6.png)

위와같이 버킷이 생성된 모습을 확인할 수 있다.

![img_7.png](.index_images/img_7.png)

버킷 탭에 들어가서 이미지를 하나 업로드하자.

![img_8.png](.index_images/img_8.png)

favicon.png 파일을 업로드했다.

![img_9.png](.index_images/img_9.png)

이 상태로 객체 URL에 접근해보면

![img_10.png](.index_images/img_10.png)

다음과 같이 Access Denied가 뜬다.
버킷을 생성할 때 Public으로 생성하지 않았기 때문이다.

### 비공개인 버킷에 접근하기

버킷을 Private으로 만들었기 때문에 사용자가 자원에 직접적으로 접근하지 못한다.

![img_32.png](.index_images/img_32.png)

> 출처: https://techblog.woowahan.com/6217/

사용자가 CloudFront를 우회하여 접근할 수 있도록 구성해보자.

#### CloudFront 설정

![img_1.png](.index_images/img_34.png)

CloudFront 배포 생성에서 원본 도메인을 위에서 생성한 S3로 선택한다.

![img_2.png](.index_images/img_35.png)

이후 원본액세스 제어 설정을 선택하고 `제어 설정 생성` 탭을 선택한다.

![img_3.png](.index_images/img_36.png)

위와같이 제어 설정을 구성한다.

![img_4.png](.index_images/img_37.png)

기본 캐시 동작은 `Redirect HTTP to HTTPS` 를 선택하고 새 배포를 생성한다.
(WAF 설정은 적용하지 않았다)

#### 버킷 정책 업데이트

![img_5.png](.index_images/img_38.png)

CloudFront를 생성하면 위처럼 S3 버킷 정책을 업데이트하도록 나온다.
`정책 복사` 를 투르고 `정책을 업데이트하려면 S3 버킷 권한으로 이동합니다.` 를 누른다.

![img_6.png](.index_images/img_39.png)

버킷 - 퀀합 탭에서 버킷 정책 편집으로 들어간다.

![img_7.png](.index_images/img_40.png)

복사한 내용을 버킷 정책에 붙여넣고 변경사항을 저장한다.

#### 접속하기

![img_8.png](.index_images/img_41.png)

이제 CloudFront 탭에 다시 들어가서 배포 도메인 이름을 확인한다.

`배포도메인/{S3경로파일명}`로 호출했을 때 파일이 정상적으로 호출되는지 확인해본다.

![img_9.png](.index_images/img_42.png)

#### 대체 도메인 설정하기

> 해당 과정을 수행하기 위해서는 도메인을 구입이 선행되어야합니다.
도메인 구입은 [피움의 배포과정](https://pium-official.github.io/pium-deploy-step/)을 참고해주세요

매번 `배포도메인/{S3경로파일명}`으로 접속하기에는 도메인의 길이도 너무 길고 복잡하다.

`image.pium.life/{S3경로파일명}`로 접속하도록 구성해보자.

#### 인증서 발급

우선 인증서를 발급받아야한다.

특정 웹서버에 인증서를 붙이는것이 아니므로 image.pium.life에 대해 SSL 인증서만 발급받아보자.

- **certbot으로 SSL 인증서 발급**

> certbot에 대한 내용은 [피움 팀블로그 - 내 서버에 HTTPS 설정하기](https://pium-official.github.io/apply-https/)를 참고

맥북에서 image.pium.life에 대한 SSL 인증서를 발급받아보자.
certbot을 받아준다.

```shell
# certbot 설치
brew install certbot

# dns challenge 방식의 인증을 수행한다
sudo certbot certonly --manual --preferred-challenges dns -d image.pium.life
```

![img_10.png](.index_images/img_43.png)

서비스에 대한 동의를 진행한다.

![img_11.png](.index_images/img_11.png)

여기서 해당 도메인의 소유자인지 검증하기위해 DNS 서비스에 위 값을 등록해야한다.
**(아직 엔터를 누르면 안된다!!!)**

![img_12.png](.index_images/img_12.png)

위처럼 DNS 관리 툴에서 TXT 레코드 정보를 추가해준다.
이후 해당 도메인이 정상적으로 배포되었는지 확인하기 위해 `Admin Toolbox: https://...`에 쓰여있는 링크로 접속한다.

> https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.image.pium.life.

![img_13.png](.index_images/img_13.png)

도메인이 아직 배포되지 않았다면 `Record not found!`가 뜬다.

![img_14.png](.index_images/img_14.png)

잠시 기다리면 이렇게 도메인 조회가 된다.

![img_15.png](.index_images/img_15.png)

배포 확인 후 다시 이 화면으로 넘어와서 엔터를 누르면 된다.

![img_16.png](.index_images/img_16.png)

![img_18.png](.index_images/img_18.png)

`/etc/letsencrypt/live/image.pium.life` 경로에 SSL 인증서가 발급되었다.

> 인증서 발급이 완료되었으므로 DNS 서비스에 등록했던 TXT 레코드는 삭제해도된다.

#### AWS에 인증서 넣기

이제 이 인증서를 AWS로 가져가보자.

![img_19.png](.index_images/img_19.png)

AWS의 Certificate Manager 서비스를 이용한다.

![img_20.png](.index_images/img_20.png)

> CloudFront의 SSL 인증서 적용 과정에서 미국 동부(버지니아 북부) 리전의 인증서를 요구하기 때문에 리전을 버지니아-북부로 설정해두고 작업을 진행해야한다!
>
> ![img_21.png](.index_images/img_21.png)
>
> ![img_22.png](.index_images/img_22.png)


인증서 가져오기를 수행한다.

![img_23.png](.index_images/img_23.png)

인증서 세부 정보를 입력하는 칸이 나오는데 각각 해당하는 정보를 적어주면 된다.

![img_24.png](.index_images/img_24.png)

```
sudo cat cert.pem
sudo cat privkey.pem
sudo cat fullchain.pem
```

> ![img_25.png](.index_images/img_25.png)
>
> `-----BEGIN CERTIFICATE-----`부터
> `-----END CERTIFICATE-----`까지
> 다 복사해서 넣어줘야한다.

![img_26.png](.index_images/img_26.png)

등록을 누르면 다음과 같이 인증서를 가져올 수 있다.

![img_27.png](.index_images/img_27.png)

#### CloudFront 별칭 설정

![img_28.png](.index_images/img_28.png)

DNS 호스팅 사이트에 가서 CNAME 레코드를 추가해준다.
호스트는 image이고 값은 CloudFront의 배포 주소를 넣어준다.

![img_29.png](.index_images/img_29.png)

CloudFront 탭으로 와서 편집탭으로 들어간다.

![img_30.png](.index_images/img_30.png)

대체 도메인을 작성하고 추가한 SSL 인증서를 선택한 뒤 저장한다.

![img_31.png](.index_images/img_31.png)

이제 image.pium.life로 cloudfront로 접속할 수 있게 되었다.

## 후기

단순한 내용도 많았지만 중간에 인증서 발급 등과 같은 세세한 부분에서 헤맬 수 있다고 생각하여 최대한 자세하게 기록했다.

## Reference

- https://medium.com/dream-youngs/lets-encrypt-dns-%EB%A1%9C-%EB%93%B1%EB%A1%9D%ED%95%98%EA%B8%B0-fdc3efda36af
- https://techblog.woowahan.com/6217/