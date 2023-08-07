---
title: "stylelint와 husky를 통해 컨벤션 통일하기"
description: "stylelint를 통해 css 컨벤션을 통일하고, husky를 통해 git 컨벤션을 통일할 수 있습니다."
date: 2023-08-07
update: 2023-08-07
tags:
  - stylelint
  - husky
  - 컨벤션
---

> 이 글은 우테코 피움팀 크루 '[클린](https://github.com/hozzijeong)'가 작성했습니다.

# style-lint, husky를 통한 컨벤션 자동화 하기

프로젝트는 혼자 할 수 도 있지만 팀원들과 함께 진행하는 경우도 있습니다. 피움의 경우에 팀원들과 함께 진행하고 있습니다. 개인적으로 생각했을 때 코드를 작성한다는 것은 그 자체로 하나의 문서를 만드는 것이라고 생각합니다. 그리고 그 문서의 양식을 통일하는 것이 가독성이나 유지 보수를 수월하게 한다고 생각합니다. 그렇기에 각각의 팀바다 코드 컨벤션이 존재하고, 그 컨벤션을 강제하도록 하는 플러그인 (`eslint`, `prettier`)이 존재합니다. 

이번에 소개할 기능은 css컨벤션을 맞출 수 있게 도와주는 style-lint와 git 메세지 양식을 지정해주는 husky에 대해 소개해 보려 합니다. 

### Stylelint

[링크](https://stylelint.io/)

> A mighty CSS linter that helps you avoid errors and enforce conventions.
> 

stylelint의 소개글 입니다. 간단하게 말해서 css 에러를 피하고 컨벤션을 강요하도록 하는 기능입니다. 설치는 다음과 같습니다. 저희는 styled-component를 사용하기 때문에 해당 요건에 맞는 라이브러리를 설치하겠습니다.

```bash
npm install --save-dev stylelint stylelint-config-styled-components
```

라이브러리 설치가 끝나면 `.stylelintic`이라는 파일에서 원하는 css규칙을 결정할 수 있습니다. 

```json
{
  "extends": ["stylelint-config-clean-order", "stylelint-config-styled-components"],
  "plugins": ["stylelint-order"],
  "customSyntax": "postcss-styled-syntax"
}
```

피움에서는 `stlyelint-order`라는 플러그인을 설치한 뒤에 해당 규칙에 따라 css 컨벤션을 정했습니다. 실행시키는 방법은 다음과 같습니다.

```json
"scripts": {
	"stylelint": "stylelint --ignore-path .gitignore '**/*.(ts|tsx)'",
}
```

위와같이 설정하고 명령어를 입력하게 된다면 css파일을 돌면서 규칙에 맞지 않는 파일을 찾아줍니다. 

```bash
npm run stylelint
```

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/ddb7ccf7-48ac-498f-a01b-199966021c93)


하지만 우린 좀 더 편한걸 원합니다. 매번 실행시키고 경고창들을 변경하기 보다 실행시킬때마다 자동으로 수정시켜주는걸 원합니다. 만약 그것을 원한다면 명령어에 `—fix`만을 추가해주면 됩니다.

```json
"scripts": {
	"stylelint": "stylelint --ignore-path .gitignore '**/*.(ts|tsx)' --fix",
}
```


https://github.com/pium-official/pium-official.github.io/assets/50974359/0f9aa05c-a861-4601-a959-58754752294d


여기에 더 가서 파일을 저장할 때마다 실행시킬수도 있습니다.

vsc의 `setting.json`에 들어가서 stylelint에 대한 설정을 해주면 됩니다. (vsCode extension을 먼저 설치해야 합니다)

```json
//...
	"editor.codeActionsOnSave": {
		"source.fixAll.stylelint": true,
		"source.fixAll.eslint": true,
		"source.fixAll.prettier": true
	},
	"stylelint.validate": [
		"typescript",
		"typescriptreact",
		"css",
		"less",
		"postcss",
		"scss",
		"html",
		"javascript",
		"vue-html",
		"vue-postcss",
		"sass",
		"markdown",
		"vue"
	],
//...
```

codeActionOnSave는 저장할때마다 stylelint에 있는 모든 것을 fix한다는 것이고 아래에 있는 stylelint.validate는 해당 옵션을 적용시킬 확장자를 의미합니다. 위와 같이 설정하고 저장한다면 저장을 할 때마다 자동으로 컨벤션 적용이 되는 것을 볼 수 있습니다.

### Husky

[링크](https://typicode.github.io/husky/)

> Husky improves your commits and more 🐶 *woof!*
> 
> 
> You can use it to **lint your commit messages**, **run tests**, **lint code**, etc... when you commit or push. Husky supports **[all Git hooks](https://git-scm.com/docs/githooks)**.
> 

이번에는 `git`에 대한 컨벤션을 맞추기 위해 `husky`라는 라이브러리를 통해 `git hook`을 실행하보려 합니다. `git`에는 `commit`, `push` 등의 명령어가 있지만, 각각의 명령어가 실행되기 전에 전반적으로 실행시키고 싶은 명령어가 있거나, 컨벤션을 맞추거나, `commit` 메세지 형식을 강제할 수 있습니다. 이러한 일련의 과정이 필요한 이유는 팀 컨벤션에 맞춰진 **신뢰성 있는 코드**를 원격 저장소에 저장하기 위해서 입니다. 예시로 앞서 얘기한 `stylelint`가 적용이 되지 않은 코드가 원격 저장소에 올라가게 된다면, 컨벤션이 맞춰지지 않은것이고 공통된 문서로써의 역할을 잃을 수 있습니다. 이에 각각의 명령어기 실행되기 전에 단계를 `pre`라는 접두사를 붙여서 `pre-commit` 또는 `pre-push`단계라고 하는데, 이 각각의 단계에서 `test`를 실행하거나 build`를` 실행할 수도 있습니다.

다음 명령어를 통해 설치할 수 있습니다.

```bash
npm install husky --save-dev
npx husky install // 이 명령어를 통해서 git hook을 사용할 수 있도록 함.
```

`hooks`를 만들기 위해서는 다음과 같이 입력하면 됩니다. `husky add <file> [cmd]` → 이와 같은 형식으로 `husky` 폴더를 생성할 수 있습니다. (단, 해당 명령어를 입력하기 전에 `husky install`을 꼭 해야합니다!)

```bash
npx husky add .husky/pre-commit "npm test"
```

위와같이 설정하고 나면 .husky폴더에 다음과 같은 파일이 생성됩니다.

![image](https://github.com/pium-official/pium-official.github.io/assets/50974359/82a9d271-4748-4871-be3a-220ca36c2de0)

이제 모든 커밋을 하기 전에 `npm test`명령어가 실행되게 됩니다. 이제, `test`를 통과하지 못한다면 `commit`은 일어나지 않습니다. 이를 통해 원격 저장소에 올라가는 코드들의 신뢰성은 급상승 하게 됩니다! 

하지만 하나의 단점이 있습니다. 피움 프로젝트를 모노레포 형식으로 진행하고 있는데, `husky`가 `githook`이기 때문에 `npm`에 의존성이 없는 `backend` 폴더에서도 커밋이 발생하기 전에 해당 `npm test`가 실행된다는 점입니다. 이는 상당히 좋지 못합니다… `pre-commit` 단계에서 명령어를 `cd frontend && npm test`와 같이 변경해서 실행해봐도 백엔드에서 커밋을 할때마다 해당 명령어가 실행된다는게 상당히 비효율적인 방식이라고 생각했습니다. 따라서 피움 프론트엔드 CI는 husky로 진행하지 않기로 했습니다. 단, `commit` 메세지 형태는 강제할 수 있었습니다. `git` 컨벤션인 부분이라 FE/BE 모두 공통으로 해당되는 내용이었기에 해당 부분만 추가하는 방식으로 `husky`를 사용하기로 했습니다.

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# 커밋 컨벤션
# 0. 검사 예외 조건 (자동 생성, 최초 커밋)
# - Merge branch*, Merge pull request*, initial*
# 1. 접두사의 글자는 소문자
# 2. 맨 마지막 글자 '.' 마침표 금지
# 3. 커밋 접두사 (규칙: '접두사' + '콜론' + ' ')
# - feat: 새로운 기능 추가
# - fix: 버그 수정
# - docs: 문서의 수정
# - style: (코드의 수정 없이) 스타일(style)만 변경(들여쓰기 같은 포맷이나 세미콜론을 빼먹은 경우)
# - refactor: 코드를 리펙토링
# - test: Test 관련한 코드의 추가, 수정
# - chore: (코드의 수정 없이) 설정을 변경
# - design: css 코드 수정
# - build: build 및 관련 설정 변경

COMMIT_MSG_FILE=$1
FIRST_LINE=`head -n1 ${COMMIT_MSG_FILE}`
RES="needCheck" # needCheck, auto, initial, lintError*, clear

if [[ $FIRST_LINE =~ ^(Merge branch) ]] ||
   [[ $FIRST_LINE =~ ^(Merge pull request) ]]; then
  RES="auto"
fi

if [[ $FIRST_LINE =~ ^(initial) ]]; then
  RES="initial"
fi

if [ $RES == "needCheck" ]; then
  if [[ $FIRST_LINE =~ (\.)$ ]]; then
    RES="lintError1"
  fi

  if [[ ! $FIRST_LINE =~ ^(feat: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(fix: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(docs: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(style: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(refactor: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(test: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(design: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(build: ) ]] &&
     [[ ! $FIRST_LINE =~ ^(chore: ) ]]; then
    RES="lintError2"
  fi

  if [[ ! $RES =~ ^(lintError) ]]; then
    RES="clear"
  fi
fi

if [[ $RES =~ ^(lintError) ]]; then
  if [[ $RES == "lintError1" ]]; then
    echo "CommitLint#1: 문장 마지막의 ('.') 마침표를 제거해주세요."
  fi
  if [[ $RES == "lintError2" ]]; then
    echo "CommitLint#2: 접두사, 콜론, 띄어쓰기 형태를 확인하세요. (feat: , fix: , docs: , style: , refactor: , test: , chore: )"
  fi
  exit 1
elif [[ $RES == "auto" ]]; then
  echo "Automatically generated commit message from git"
elif [[ $RES == "initial" ]]; then
  echo "Initial commit"
elif [[ $RES == "clear" ]]; then
  echo "Pass commit lint!"
fi

exit 0
```

프로젝트 `git` 컨벤션에 맞춰서 메세지 작성을 강제 했고, 이를 통해 `commit` 메세지 통일을 할 수 있었습니다!

---

### 맺는말
프로젝트는 팀원들이 하나의 문서를 만드는 것이라고 생각을 합니다. 그리고 통일된 문서를 만들때 가장 필요한 것은 규칙(컨벤션)이고 그 규칙을 자동으로 강제할 수 있고, 코드의 신뢰성을 높여줄 수 있는 라이브러리에 대해서 한번 알아봤습니다.
스타일 컨벤션을 맞추기 위해서는 `stylelint`를 `git hook`을 통해 `git`컨벤션을 맞추고 싶은 경우에는 `husky`를 통해서 컨벤션을 통일 할 수 있었습니다.
