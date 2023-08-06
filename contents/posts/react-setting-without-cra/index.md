---
title: "webpack으로만 React 설정하기 (without CRA)"
description: "피움 팀의 프론트엔드 초기 세팅 과정입니다."
date: 2023-08-06
update: 2023-08-06
tags:
  - React
  - Webpack
---

# CRA없에 webpack으로 react 세팅하기

피움 서비스 환경을 만들기로 하면서, FE의 개발 스펙은 React + TypeScript로 설정하였습니다. React는 당연 현존 최강 라이브러리이고, 여기에 JavaScript의 동적 타입을 컴파일 단계에서 잡아주는 TypeScript까지 더해서 안정적인 서비스를 개발하려고 합니다. 

평소같았으면 `CRA`를 통해 손쉽게 리액트 설치를 하고 개발을 시작했을 텐데, CRA의 단점이 이미 정교하게 세팅된 babel과 webpack 설정을 따라 개발을 해야 한다는 것이고 그에 따른 트레이드 오프가 상당히 많이 발생할 수 있다는 점입니다. (하나의 예시로 CRA를 통해서는 절대경로 지정 하기가 어렵습니다.) 또한, 이왕에 내 서비스 개발하는거 남이 차려준 밥상 보다 내가 직접 한번 해보겠다는 생각으로 webpack으로만 react세팅을 시도해 보려고합니다 (가장 큰 원인은 요구사항이 CRA없이 webpack으로 react 설치하는 거였습니다 ㅎ)

### package.json 설치

우선 가장 처음 해야할 단계입니다. react, typescript, webpack 등 우리가 설치할 의존성 관리와 기타 프로젝트 정보를 저장해 줄 `package.json`을 설치 해줍니다.

```bash
npm init // 또는
yarn init -y
```

### 환경 구축에 필요한 패키지 설치

```bash
npm install react react-dom // react 설치
npm install -D typescript @types/react @types/react-dom // typescript 설치
npm install tsc-init // tsconfig 설치

npm install -D webpack webpack-cli // 웹팩 라이브러리와 명령어를 통해 웹팩을 이용할 수 있는 라이브러리 설치
npm install -D html-webpack-plugin webpack-dev-server ts-loader  // 각각 번들 후 html파일을 만들어주는 플러그인, 개발할 때 사용할 웹 서버, typescript파일을 javascript로 변환하는 webpack loader 설치
```

위에 파일들을 설치하고 나면 이제부터 본격적인 세팅을 합니다.

### webpack.config.js 파일 설정

웹팩 파일 설정입니다. 

```jsx
// webpack.config.js
const { resolve } = require('path'); // path 설정에 도움을 줍니다.
const HtmlWebpackPlugin = require('html-webpack-plugin');
const ReactRefreshWebpackPlugin = require('@pmmmwh/react-refresh-webpack-plugin');
const ReactRefreshTypeScript = require('react-refresh-typescript');

const isDevelopment = process.env.NODE_ENV === 'development';

module.exports = {
  entry: resolve(__dirname, 'src', 'index.tsx'), 
  output: {
    path: resolve(__dirname, 'dist'), // 번들 파일이 저장될 디렉토리
    filename: 'bundle.js', // 파일 이름
    assetModuleFilename: 'assets/[name][ext]', // 에셋의 파일 이름
    clean: true, // 빌드 이전 결과물 정리 옵션
    publicPath: '/', // 빌드된 파일 앞에 붙일 경로
  },
  resolve: {
    extensions: ['.ts', '.tsx', '...'], // 파일 확장자. TS및 JSX확자자ㅡㄹ 설정했습니다.
    alias: { // 절대경로 설정
      types: resolve(__dirname, 'src', 'types'),
      pages: resolve(__dirname, 'src', 'pages'),
      components: resolve(__dirname, 'src', 'components'),
      hooks: resolve(__dirname, 'src', 'hooks'),
      apis: resolve(__dirname, 'src', 'apis'),
      utils: resolve(__dirname, 'src', 'utils'),
      assets: resolve(__dirname, 'src', 'assets'),
      constants: resolve(__dirname, 'src', 'constants'),
      contexts: resolve(__dirname, 'src', 'contexts'),
    },
  },
  module: { // 로더 및 규칙 설정
    rules: [
      {
        test: /\.tsx?$/i, // ts-loader를 사용했습니다. babel-loader를 사용하지 않은 이유는 IE까지 폴리필 지원을 하지 않아도 되기 때문입니ㅏㄷ.
        exclude: /node_modules/,
        loader: 'ts-loader',
        options: {
          getCustomTransformers: () => ({
            before: isDevelopment ? [ReactRefreshTypeScript()] : [],
          }),
          transpileOnly: isDevelopment,
        },
      },
      {
        test: /\.(png|jpg|jpeg|svg|woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource',
      },
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({ //빌드된 다음에 템플릿을 지정해줍니다.
      template: resolve(__dirname, 'public', 'index.html'),
    }),
    ...(isDevelopment ? [new ReactRefreshWebpackPlugin()] : []),
  ],
  devServer: {
    open: true,
    port: 8282, // 기본은 3000번이지만 커스텀 기념으로 바꿨씁니다.
    static: {
      directory: resolve(__dirname, 'public'),
    },
    historyApiFallback: true,
  },
};
```

`entry`: 웹팩의 진입점을 의미. react에서는 src의 index.js에서 모든게 시작됩니다.

`output`: 번들된 파일의 출력 경로를 정의합니다.

`resolve`: 모듈 해석 옵션을 설정합니다.

`module`: 로더의 규칙을 설정합니다.

- 보통은 .css,.scss파일 확장을 위해 `style-loader`를 사용하기도 하지만, 피움에서는 CSS-in-JS를 선택했기 때문에 해당 확장자들이 존재하지 않아서 따로 설정하지 않았습니다.

`plugins`:  웹팩 플러그인을 설정합니다. 플러그인은 빌드 과정을 확장하거나 수정하는데 사용됩니다.

`devServer`: 개발 서버 옵션(localhost)을 설정합니다. 

> `ReactRefreshWebpackPlugin`은 리액트에서 저장을 파일 저장을 할 때마다 상태 초기화를 막아주는 역할을 하는 플러그인 입니다. CRA에서는 기본적으로 제공되어 있는 기능이지만, custom기능에는 존재하지 않기에 추가해 줬습니다.

### tsconfig파일 설정

tsconfig파일은 typescript의 버전과 모듈 등 컴파일 과정에서 어떤 버전으로 실행할 것인지 등을 설정하는 파일입니다.

```json
{
  "compilerOptions": { // 컴파일러 옵션 설정
    "target": "ES2016", // 컴파일 하는 JS 버전을 ES2016으로 설정
    "module": "CommonJS", // 컴파일된 모듈 시스템 설정
    "jsx": "react-jsx", // JSX문법을 어떻게 컴파일 할지 설정

    "strict": true, // 엄격한 타입 체크
    "exactOptionalPropertyTypes": true, // 선택적 프로퍼티 타입을 체크함
    "skipLibCheck": true, // 라이브러리 파일의 타입 체크를 건너뜀
    "esModuleInterop": true, // CJS 및 ES 모듈을 함께 사용할 때 옵션을 설정 ES6 모듈 사양을 준수하여 CJS사용 가능하게 함. (즉 import 가능)

    "baseUrl": "src", // 루트가 되는 url
    "paths": { // 절대 경로
      "types/*": ["types/*"],
      "pages/*": ["pages/*"],
      "components/*": ["components/*"],
      "contexts/*": ["contexts/*"],
      "hooks/*": ["hooks/*"],
      "utils/*": ["utils/*"],
      "assets/*": ["assets/*"],
      "constants/*": ["constants/*"]
    },

    "types": ["cypress"] // 타입 정의 파일 지정
  },
  "include": ["src/**/*", "cypress/**/*"] // 컴파일할 소스 코드 파일 정의
}
```

피움에서는 IE를 제외한 모든 브라우저를 타겟으로 설정하고 있기 때문에 최대한 낮은 버전의 `JavaScript`를 타겟으로 설정했습니다. ES2016을 설정하고 처음 코드를 작성하다가 `import`가 제대로 이루어지지 않는 것을 보고 문제를 찾다가 `esModuleInterop`옵션을 true로 설정해 줘야지 CJS에서도 import가 가능하다는 것을 알게 되었습니다. 

또한, 상대 경로가 아닌 절대경로 사용을 선택했는데, 이는 복잡한 파일 구조를 보기 쉽게 알아보기 위해 설정했습니다.

> `esModuleInterop`설정은 target으로 하는 옵션이 node16이거나 nodenext인 경우에는 true로 되어있지만, 그게 아닌 경우에는 false이므로, import 구문을 사용하고 싶다면 옵션을 true로 하는것이 좋습니다.

### package.json 설정

이제 설정한 웹팩과 tsconfig를 확인해 보기 위해서는 실행 스크립트를 알아야 합니다. 보통 자주 사용하는 명령어로는 `start`와 `build`가 있습니다. 그리고 나머지 명령어는 스토리북이나 cypress에 대한 명령어들입니다.
```json
{ //package.json
//...
	"name": "pium-frontend",
  "version": "1.0.0",
  "description": "피움 프론트엔드",
  "main": "src/index.tsx",
  "scripts": {
    "test": "echo test is not prepared",
    "start": "webpack-dev-server --mode=development",
    "build": "NODE_ENV=production webpack",
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build -o ./dist/storybook",
    "cypress": "cypress open"
  },
//...
}
```
