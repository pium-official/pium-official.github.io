---
title: "홈화면에 추가 안내 prompt 만들기"
description: "웹 브라우저를 어플처럼 사용할 수 있는 홈 화면에 추가 기능을 간단하게 만들어보자"
date: 2023-11-01
update: 2023-11-01
tags:
  - PWA
  - beforeInstallPromptEvent
---

> 이 글은 우테코 피움팀 크루 '[클린](https://github.com/hozzijeong)'가 작성했습니다.

# 바로가기 prompt 만들기

`manifest` 설정을 했다면 홈 화면에 바로가기를 추가할 수 있습니다. 보통은 “홈 화면에 바로가기 추가”를 통해 바로가기를 등록할 수 있지만 모든 사용자가 이 방법을 알고 있는 것도 아니고 접근성이 떨어지게 됩니다. 추가적으로 화면에 바로 들어왔을 때 아래와 같이 바로 알림이 나타난다면 그닥 이쁘지는 않고 이게 뭐지? 라는 생각이 들 수 있습니다.

<img width="324" alt="스크린샷 2023-10-31 오후 9 46 51" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/b6b1f2c7-bb0e-41bb-a8d0-a96f1d91e282">


그래서 사용자의 접근성을 높이기 위해 바로가기 prompt를 만들려고 합니다. 원하는 결과는 아래와 같습니다. 바로가기 추가 안내를 통해 설치를 유도해보겠습니다.

<img width="302" alt="스크린샷 2023-10-31 오후 9 48 25" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/2c475f33-03b8-4829-9192-d07c8359b98e">


설치 prompt를 띄우기 위해서는 `beforeinstallprompt`라는 window이벤트를 통해 띄울 수 있습니다. 그렇다면 해당 이벤트가 모든 브라우저에서 지원하는지 먼저 확인해 보겠습니다.

<img width="1289" alt="스크린샷 2023-10-31 오후 9 59 11" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/a7839044-1bc8-41ba-b0a6-0778a7095534">


전 세계적으로 72%가 사용이 가능하다고 하지만 충격적이게도 IOS와 FireFox에서는 지원하지 않는 기능입니다. 일단은 지원한다는 가정하에 코드를 작성해 보겠습니다.

### Vanilla Javascript

```jsx
<button id="install" hidden>Install</button>
```

```jsx
let installPrompt = null;
const installButton = document.querySelector("#install");

window.addEventListener("beforeinstallprompt", (event) => {
  event.preventDefault();
  installPrompt = event;
  installButton.removeAttribute("hidden");
});
```

제일 처음 접속했을 때 `beforeinstallpromptEvent`객체를 전달 받아서 `installPrompt`에 event를 할당할 수 있습니다. 그리고 이벤트 할당이 완료된다면 `install`버튼의 `hidden`속성을 제거해서 설치 버튼을 나타나게 할 수 있습니다. 

```jsx
installButton.addEventListener("click", async () => {
  if (!installPrompt) {
    return;
  }
  const result = await installPrompt.prompt();
  console.log(`Install prompt was: ${result.outcome}`);
  installPrompt = null;
  installButton.setAttribute("hidden", "");
});
```

그리고 해당 버튼에 click 이벤트를 걸어서 바로가기 추가를 만든 다음에 다시 버튼을 가리는 방식으로 사용할 수 있습니다. 

### React + TypeScript

대략적인 흐름은 위와 같고 피움에서 `React` + `TypeScript`를 통해 어떻게 구현했는지 한번 보겠습니다. 요약하자면 다음과 같습니다.

- `deferredPrompt`라는 상태 필요
- useEffect를 통해 window에 `beforeInstallPrompt` 이벤트 부착
- 이벤트 핸들러 에서 `deferredPrompt`상태에 값을 set 함.
- `defferdPrompt` 상태에 따라 installPrompt 조건부 렌더링
- 설치 버튼에서 사용자 응답에 따라 설치 진행

```jsx
declare global {
  interface WindowEventMap {
    beforeinstallprompt: BeforeInstallPromptEvent;
  }
}

interface BeforeInstallPromptEvent extends Event {
  readonly platforms: Array<string>;
  readonly userChoice: Promise<{
    outcome: 'accepted' | 'dismissed';
    platform: string;
  }>;
  prompt(): Promise<void>;
}
```

우선 window에 있는 `beforeinstallpromptEvent`라는 타입이 존재하지 않기 때문에 `interface`로 선언한 뒤 전역 타입으로 추가를 해야 했습니다.

```jsx
const [deferredPrompt, setDeferredPrompt] = useState<BeforeInstallPromptEvent | null>(null);
```

렌더링이 완료된 다음에 window객체에 이벤트를 부착하겠습니다.

```jsx
const beforeInstallPromptHandler = (event: BeforeInstallPromptEvent) => {
    event.preventDefault();
    const showPrompt = JSON.parse(getCookie('PromptVisible') ?? 'true');

    if (!showPrompt) return;

    setDeferredPrompt(event);
  };

 useEffect(() => {
    window.addEventListener('beforeinstallprompt', beforeInstallPromptHandler);

    return () => {
      window.removeEventListener('beforeinstallprompt', beforeInstallPromptHandler);
    };
  }, []);
```

처음에 handler에서 `defferdPrompt`를 set한 다음에 다음 값을 렌더링 할 수 있습니다. 그 다음은 딱히 다를 것이 없습니다. `installApp`메서드를 통해 설치 완료시에 `prompt`를 띄워주고, 설치를 원하지 않는다면 `defferdPrompt`의 값을 null로 set 해주는 방식으로 작성 했습니다.

여기서 사용자가 처음에 설치를 거절했다면 그 상태를 cookie에 저장하도록 했습니다. localStorage에 값을 저장한다면 사용자가 비우지 않는한 혹은 브라우저를 바꾸지 않는 한 영원히 뜨지 않기 때문에… 서비스 입장에서도 좋지 않고 사용자도 혹시 원할지도 모른다는 생각이 들었습니다. 따라서  일정 시간이 지나면 다시 `prompt`를 띄우기 위해서 `cookie`를 활용했습니다.

### If not Support Browser

만약에 지원하지 않는 브라우저라면 (특히 ios라면) 약간은 다르게 접근을 했습니다. 애초에 홈 화면 바로가기가 지원되지 않는 상황에서 다른 방식으로 안내를 할 수 있습니다. 

```jsx
const isIos = /iPhone|iPod|iPad/i.test(navigator.userAgent);

return isIos ? <IOSGuide/> : <buttonComponent/>
```

아래와 같이 `beforeInstallPrompt`를 지원하는 여부에 따라 다르게 안내를 할 수있습니다.

<img width="371" alt="스크린샷 2023-10-31 오후 11 19 20" src="https://github.com/pium-official/pium-official.github.io/assets/50974359/b8731296-383b-424e-ab24-b94f1871f3e3">


### Challenage

`beforeinstallprompt`이벤트가 url이 `/`인 경우에만 부착이 되는 문제가 있었습니다. 저희 서비스는 로그인을 한 사용자를 상대로 바로가기 추가를 안내하고 있었고, 로그인이 완료 된다면 리마인데 페이지로 바로 이동하기 때문에 `/reminder` 경로에 `installPrompt` 컴포넌트를 렌더링 하도록 했는데 이게 잘 작동하지 않는 문제가 있었습니다. 

심지어 홈으로 돌아간 다음에 새로고침을 해야지 `prompt`가 나타나는 상황이 있는데, 이 문제의 원인을 잘 모르겠어서 골머리를 썩히고 있습니다. 혹시 짐작가시는게 있다면 댓글로 공유 부탁드립니다!
