---
title: "bundle-analyze를 통한 bundle 크기 분석"
description: "bundle-anaalyze를 라이브러리를 통해 번들 크기 분석하기"
date: 2023-07-28
update: 2023-07-28
tags:
- bundle
- 최적화
---

> 이 글은 우테코 피움팀 크루 '[클린](https://github.com/hozzijeong)'가 작성했습니다.

## 사건의 발단

이번 Level3 2차 데모데이때 권장사항을 맞추기 위해서 프로덕션 배포까지 진행했습니다. 어찌 저찌 배포는 잘 끝이 났는데 문제는 첫 페이지가 로드 되는데 약 4초 가량의 시간이 걸린다는 것이었습니다. 네트워크 문제 문제인가? 하고 봤는데 아니었고, 결국 `bundle.js` 파일의 크기를 보니 무려 16m나 되었던 것입니다.

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/72ec258f-77e0-41aa-8bcd-c4c3d3b0410e)

왜 이렇게 크기가 크지? 라는 생각을 하며 크게 2가지의 경우를 생찾아봤습니다. 하지만 다음과 같은 이유로 원인이 아님을 알았습니다.

1. minify가 되지 않았다 → webpack 4 이상을 사용하면서 자동으로 `production` 일때는 자동으로 설정
2. css 파일 크기가 너무 큼 → `styled-component`를 사용하기 때문에 .css 가 없음.

따라서 `bundle.js`의 성능을 확인해 볼 필요가 있었고 검색하던 도중 [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer)라는 라이브러리를 알게되었습니다.

## Webpack Bundle Analyzer

webpack을 통해 build한 bundle의 성능을 측정하는 라이브러리로 모든 번들에 대한 트리를 시각화 해서 보여주는 이점을 갖고 있습니다. 사용법은 매우 간단합니다. 우선 설치를 해줍니다.

```jsx
# NPM
npm install --save-dev webpack-bundle-analyzer
# Yarn
yarn add -D webpack-bundle-analyzer
```

설치를 한 뒤 `webpack.config.js` 파일 내부에서 플러그인을 사용해 줍니다.

```jsx
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}
```

플러그인 안에 옵션을 설정할 수 있는데, [여기](https://github.com/webpack-contrib/webpack-bundle-analyzer#options-for-plugin)에서 옵션에 대한 정보를 확인할 수 있습니다.

위에 플러그인 옵션에서 `generateStatsFile` 을 `true` 로 설정한 다음에 `package.json`에 `scripts` 에다음 명령어를 통해 실행할 수 있습니다.

```json
"scripts":{
  ...
  "preanalyze": "npm run build-dev",
  "analyze": "webpack-bundle-analyzer bundle/output/path/stats.json"
}
```

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/6bc524c4-4866-4ebe-9daa-95035e0c0dd5)

실행한 뒤에 bundle을 분석해 봤더니 react-icons에서 15.3m나 차지하고 있었습니다. 드디어 문제의 원흉을 찾아냈습니다. 그렇다면 react-icons를 대체할 수 있는 방법이 있을까요?

### Icons

[여기](https://icones.js.org/collection/all?s=material-symbols:cancel)에 가시면 다양한 아이콘들을 찾아볼 수 있습니다. 원하는 아이콘을 선택한 다음에 SVG, Component 등으로 코드를 반환해 줍니다. 이를 통해서 react-icons를 제거하고 icons를 사용해서 아이콘을 대체할 수 있었습니다.
