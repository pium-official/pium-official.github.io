---
title: "브라우저에서 알림 구현하기"
description: "web-push를 통해 브라우저에서도 푸시 알림을 구현해보자!"
date: 2023-11-01
update: 2023-11-01
tags:
  - PWA
  - web-push
  - Notifications API
  - Push API
  - FCM
---

> 이 글은 우테코 피움팀 크루 '[클린](https://github.com/hozzijeong)'가 작성했습니다.

PWA를 통해서 설치도 했고 Service Worker를 세팅함으로써 오프라인과 백그라운드 상황에서 이벤트에 대한 처리 역시 마칠 수 있었습니다. 이번에는 푸시 알림에 대해 알아보고 구현까지 해보려고 합니다.

푸시 알림은 말 그대로 가만히 있는데 알림을 밀어 넣어 준다는 것을 의미합니다. 푸시 기능과 알림은 서로 다른 맥락을 갖고 있지만, 둘을 합치면 우리가 상상하는 그 기능이 됩니다. 브라우저가 닫혀 있는 상황에서 서버에서 알림을 보냈을 때 푸시 기능을 통해 알림을 표시할 수 있습니다. 어떤 기술을 통해 알림이 허용되는지 보기에 앞서 어떤 원리로 웹 브라우저에서 푸시 알림 기능이 가능한지 먼저 살펴보겠습니다.

### 웹 푸시

> [user agent](https://developer.mozilla.org/en-US/docs/Glossary/User_agent)는 브라우저를, web page는 클라이언트 서비스를 의미한다.

<img width="694" alt="스크린샷 2023-11-01 오후 9 14 14" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/de44e011-8320-4ff9-a46d-f406e06f2475">


웹 푸시에 대한 대략적인 그림을 보기 전에 용어 정리 먼저 하겠습니다.

[Notifications API](https://developer.mozilla.org/ko/docs/Web/API/Notifications_API): Notifications API 는 웹 페이지가 일반 사용자에게 시스템 알림 표시를 제어할 수 있게 해줍니다.

[Push API](https://developer.mozilla.org/ko/docs/Web/API/Push_API): Push API는 웹 애플리케이션이 사용자 에이전트에서 웹 앱이 활성 상태에 있는지 또는 현재 로드 중인지 여부와 관계없이 서버로부터 푸시 메시지를 수신할 수 있는 기능을 제공합니다.

`push service`: 푸시 서비스는 서버에서 전송한 푸시 알림을 클라이언트로 전송하는 중간자 역할을 합니다. 대표적으로 구글의 [FCM](https://firebase.google.com/docs/cloud-messaging?hl=ko)과 애플의 [APNs](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/sending_notification_requests_to_apns)가 있습니다. 

클라이언트에서는 Notification API를 통해 알림 허용을 받고, Push API를 통해서 푸시 알림을 구독한 뒤 서버로 부터 전송받은 푸시 알림을 푸시 서비스를 통해서 전달 받는 동작을 하고 있습니다.

### VAPID 키

> 모든 브라우저에서 푸시를 사용하려면 [VAPID](https://tools.ietf.org/html/draft-thomson-webpush-vapid)(애플리케이션 서버 키라고도 함)를 사용해야 합니다. VAPID는 기본적으로 애플리케이션이 사용자에게 메시지를 보낼 수 있음을 증명하는 값으로 헤더를 설정해야 합니다. 푸시 메시지를 사용하여 데이터를 전송하려면 데이터를 [암호화](https://tools.ietf.org/html/draft-ietf-webpush-encryption)하고 특정 헤더를 추가하여 브라우저가 메시지를 올바르게 복호화할 수 있도록 해야 합니다. - [web.dev](https://web.dev/articles/sending-messages-with-web-push-libraries?hl=ko)
> 

위에 있는 다이어그램 처럼 클라이언트 - 푸시 서비스 - 서버 간 통신을 하기 위해서는 VAPID 키가 필요합니다. 이 `public` - `private` 쌍을 가지고 푸시 서비스는 서버를 식별하고 전달 받은 알림 데이터의 유효성 검증을 합니다.

- 서버에서는 알림 정보가 담긴 JWT를 `private`키를 통해 암호화 해서 푸시 서비스로 알림 요청을 보냅니다.
- 푸시 서비스에서는 `public`키를 통해 전달 받은 데이터를 복호화 하고 유효성을 검증합니다.
- `public`키로 구독한 디바이스에 알맞게 알림을 푸시합니다

**VAPID 키 생성**

(프로젝트가 이미 생성되어 있다고 가정하겠습니다) FCM을 사용하게 된다면 손쉽게 VAPID키를 만들 수 있습니다. 

FCM 콘솔로 이동 > 프로젝트 선택 > 프로젝트 설정 > 클라우드 메세징 > 웹 푸시 인증서 > Generate key pair

<img width="973" alt="클라우드 콘솔" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/79e4e8eb-61e9-4d3d-a1b1-1532cff9bb5d">


다음 버튼을 클릭한다면 public키와 private키가 생성이 됩니다. 여기서 생성된 값을 각각 서버와 클라이언트에 저장해서 사용하면 됩니다.

public키는 클라이언트에서 푸시서비스에 구독을 할 때 사용이되고, private키는 서버에서 푸시 서비스로 알림을 보낼때 함께 동봉되어 날라가게 됩니다.

### Service Worker Registration

서비스 워커 등록 과정은 [이전 포스트](https://blog.pium.life/serviceworker/)에서 다뤘기 때문에 간략적으로 보고 가겠습니다. service worker를 `register` 하고  `install activate` 등의 과정을 거쳐서 서비스 워커를 등록할 수 있습니다.

### Create Push Subscription

<img width="790" alt="스크린샷 2023-11-01 오후 9 05 37" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/1900fde7-b613-4a5a-935e-678bb689e8ba">


Push Suscription이란 ***푸시 서비스와 구독 정보를 주고 받는것*** 입니다. Push Subscription을 주고 받는 과정은 아래와 같습니다. 

- Notification API를 통해 권한 요청
- user agent에서 push service로 subscribe 요청
- subscribe로 얻은 구독 정보(Push Subscription)를 얻고, 해당 정보를 서버에 전송 (Distribute Subsription)

**Notification을 통한 권한 요청**

```jsx
async function notifyMe() {
	// Notification API를 지원하는지 확인한다.
  if (!("Notification" in window)) {
    alert("This browser does not support desktop notification");
		return;
  } 

	if (Notification.permission === 'granted') {
		  // 기존에 브라우저가 알림 허용했는지 확인    
      const notification = new Notification('hello?')
      return;
  }
	// 기본 설정 값이 알림을 거부하지 않았다면
  if (Notification.permission !== 'denied') {
			// 권한 승인 요청
			const permission = await Notification.requestPermission();
			if (permission === 'granted') {
        const notification = new Notification('subscribe On')
      }

}
```

이제 권한 요청을 수락했다면 서비스 워커에서 알림을 받을 수 있습니다.

```jsx
navigator.serviceWorker.register('sw.js')

function showNotification() {
    Notification.requestPermission(result => {
				// 요청을 수락했다면
        if (result === 'granted') {
            navigator.serviceWorker.ready.then(registration => {
                registration.showNotification('Vibration Sample', {
                    body: 'Buzz! Buzz!',
                    icon: '../images/touch/chrome-touch-icon-192x192.png',
                    tag: 'vibration-sample',
                })
            })
        }
    })
```

**push service로 subscribe 요청**

push service로 구독 요청하기 위해서는 `[pushManager` 인터페이스](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/pushManager)에서 `subscribe` 메서드를 통해서 구독을 할 수 있습니다. 구독하는 과정에 앞서 FCM에서 생성한 public VAPID키가 필요합니다. 해당 public키를 `subscribe()`의 옵션으로 설정한다면 그 결과로 `pushSubscription`을 받을 수 있습니다. 

```jsx
navigator.serviceWorker.register("serviceworker.js");

// Use serviceWorker.ready to ensure that you can subscribe for push
navigator.serviceWorker.ready.then((serviceWorkerRegistration) => {
  const options = {
    userVisibleOnly: true,
    applicationServerKey, // public키 입니다.
  };
  serviceWorkerRegistration.pushManager.subscribe(options).then(
    (pushSubscription) => {
      console.log(pushSubscription.endpoint);
			// subscribe에 성공했다면 pushSubscription이라는 객체를 반환해 줍니다. 
			// 이제 이 정보를 서버에 저장하면 됩니다. 
    },
    (error) => {
      console.error(error);
    },
  );
});
```

### Distribute Subscription

푸시 서비스에 구독을 설정함으로써 전달 받은 pushSubsription을 서버에 보냄으로써 서버에서는 어떤 `endpoint`와 정보를 푸시 서비스에 보내는지 확인할 수 있습니다.

```jsx
navigator.serviceWorker.ready.then((serviceWorkerRegistration) => {
  const options = {
    userVisibleOnly: true,
    applicationServerKey, // public키 입니다.
  };
  serviceWorkerRegistration.pushManager.subscribe(options).then(
    (pushSubscription) => {
			// 여기서 얻은 pushSubscription을 서버에 전송하면 됩니다.
			subscribe(pushSubscription);
    },
    (error) => {
      console.error(error);
    },
  );
});
```

### Send Push Message

<img width="804" alt="스크린샷 2023-11-01 오후 9 05 45" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/afa9d56f-9080-461f-8c4d-d4865b6648ab">

서버에서 전달받은 `pushSubscription`와 `private`키를 통해 푸시 서비스로 알림을 보낼수 있습니다. 서비스 워커에서는 푸시 이벤트를 감지하고 해당 이벤트에 알맞는 행동을 하게 됩니다.

```jsx
// serviceWorker.js
self.addEventListener("push", (event) => {
  if (!(self.Notification && self.Notification.permission === "granted")) {
    return;
  }

  const data = event.data?.json() ?? {};
  const title = data.title || "Something Has Happened";
  const message =
    data.message || "Here's something you might want to check out.";
  const icon = "images/new-notification.png";

  const notification = new self.Notification(title, {
    body: message,
    tag: "simple-push-demo-notification",
    icon,
  });

  notification.addEventListener("click", () => {
    clients.openWindow(
      "https://example.blog.com/2015/03/04/something-new.html",
    );
  });
});
```

push 이벤트에는 데이터가 같이 담겨져서 옵니다 [pushEvent 객체에 대한 인터페이스](https://developer.mozilla.org/en-US/docs/Web/API/PushEvent)를 보면 좀 더 자세히 알 수 있습니다.

### Remove Push Subscription

이제 구독중인 `pushSubscription`을 제거해 볼 차례입니다. 구독중인 푸시 서비스를 구독 해제한다고 보시면 됩니다. 

```jsx
navigator.serviceWorker.ready.then((reg) => {
  reg.pushManager.getSubscription().then((subscription) => {
    subscription
      .unsubscribe()
      .then((successful) => {
        // 푸시 서비스 구독 취소 한다면 서버에서도 구독 취소를 같이 해줘야 함
				unsubscribe(); 
      })
      .catch((e) => {
        // Unsubscribing failed
      });
  });
});
```

`pushManager`에 있는 `subscription` 정보를 얻어오고, 해당 정보에서 `unsubscribe`를 하면 간단하게 할 수 있습니다. 푸시 서비스 구독 취소에 성공하게 된다면 서버에 저장되어 있는 구독 취소 API를 호출함으로써 이제 클라이언트 - 푸시 서비스 - 서버의 관계를 끊어낼 수 있습니다.

---

### 참조

https://www.w3.org/TR/push-api/#subscription-refreshes

https://web.dev/articles/sending-messages-with-web-push-libraries?hl=ko

https://developer.mozilla.org/ko/docs/Web/API/Push_API

https://developer.mozilla.org/ko/docs/Web/API/Notifications_API
