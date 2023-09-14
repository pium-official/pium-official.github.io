---
title: "Git submodule을 이용한 민감정보 관리"
description: "submodule 적용방식을 알아봅시다."
date: 2023-09-13
update: 2023-09-13
tags:
  - git
  - submodule
---

> 이 글은 우테코 피움팀 크루 '[주노](https://github.com/Choi-JJunho)'가 작성했습니다.

## 서론

현재 피움 프로젝트는 properties, env 파일과 같은 민감정보 내용을 서버 내부에 저장, 관리하고있다.

이렇게 파일을 관리했을 때 다음과 같은 문제점이 생각나 팀원들과 discussion에서 의견을 나눠봤다.

- 캠퍼스 외부에서 properties / env 파일 관리 불가
> 현재 서버 접속 권한이 캠퍼스 내부 IP로 한정되어있기 때문에 발생한 문제점이다.
- properties 설정을 파악하기가 어려움 (vim, cat 명령어로 확인)
- 분업화 진행 간 properties / env 파일 변경에 대한 공유가 어려움


> 자세한 내용은 [discussion - submodule 도입에 대해](https://github.com/woowacourse-teams/2023-pium/discussions/324) 에서 확인

![](.index_images/59c3159b.png)

팀원들 모두 동의해줬고 이에 프로젝트에 submodule을 도입하는 과정을 정리해보려고한다.

## submodule이란?

서브모듈은 하나의 저장소(레포지토리)에서 다른 저장소를 포함하고 관리할 수 있게 해주는 기능이다.
주로 의존성 관리 혹은 외부 프로젝트 통합에 사용되곤 한다.

피움에서는 private 레포지토리를 이용해 민감정보를 관리하기 위해 submodule을 적용해보려고한다.

## 시작하기

properties, env 파일 등 다양한 민감정보를 가지는 설정파일들을 private 레포지토리에 저장하고 사용해보자.

### Private Repository 생성

민감정보를 관리할 Private Repository를 생성한다.

![](.index_images/0658737b.png)

### submodule 시작하기

우선 백엔드 프로젝트를 기준으로 submodule을 적용시켜보자.

`git submodule add {서브모듈 Repo URL}`

![](.index_images/effc0cc4.png)

git submodule add 명령어를 수행하면 루트 경로에 `.gitmodules` 파일이 생성된 것을 확인할 수 있다.

![](.index_images/20ec3a3b.png)

![](.index_images/a8d5f20f.png)

파일 내용을 살펴보면 submodule이 적용된 path와 submodule을 가져온 url이 표기되어있는 것을 확인할 수 있다.

![](.index_images/0185252a.png)


> 추가로 branch 정보를 넣으면 해당 branch를 기준으로 submodule 정보를 가져온다.
![](.index_images/0ebe1978.png)

.gitsubmodule 파일과 config 폴더가 생성되고 git stage에 올라가있는 상태가 되어있을것이다.

commit 후 push를 해보자.

![](.index_images/2e9366b0.png)

다음과 같이 서브모듈 폴더가 추가된 것을 볼 수 있다.

![](.index_images/8a331998.png)

### submodule 적용하기

내 컴퓨터에서의 submodule 적용은 완료했다.

다른 개발자들이 submodule을 어떻게 적용해야할지 가이드를 제공하자.

우선 git clone부터 시작해보자.

```shell
git clone {Project}
```

![](.index_images/632999b1.png)

submodule을 초기화하고 update한다.
**해당 작업은 처음 1번만 수행하면 된다.**

```shell
git submodule init

git submodule update
```

![](.index_images/3a3b7bf7.png)

### submodule 업데이트 반영하기

프로젝트를 진행하는 과정에서 submodule 프로젝트에 수정사항이 생길 수 있다. 이를 반영하기 위해서는 다음과 같은 작업을 수행하면 된다.

> 아래와 같이 새로운 커밋(변경사항)이 발생했다고 가정해보자.
![](.index_images/8fe95451.png)

```shell
git submodule update --remote --merge

## .gitmodules 파일에 정의되어 있는 브랜치의 최신 버전으로 업데이트 (혹은 default 브랜치)
git submodule update --remote

## 로컬에서 작업 중인 부분과 원격에 작업된 부분이 다른 경우 머지까지 진행
git submodule update --remote --merge
```

![](.index_images/34d677ff.png)

submodule이 update된 정보에 대해 기존 프로젝트에도 반영을 해줘야한다.

![](.index_images/dc253c7e.png)

```shell
git add {file}

git commit -m "build: 서브모듈 최신사항 반영"

git push
```

![](.index_images/0a7b7407.png)

## 정리

민감정보를 관리하기위해 submodule 도입방법을 정리해봤다.
생각보다 간단했다 👍

## Reference

- https://tecoble.techcourse.co.kr/post/2021-07-31-git-submodule/

### Reference

- https://medium.com/dream-youngs/lets-encrypt-dns-%EB%A1%9C-%EB%93%B1%EB%A1%9D%ED%95%98%EA%B8%B0-fdc3efda36af
- https://techblog.woowahan.com/6217/