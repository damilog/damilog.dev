---
title: Yarn Berry의 PnP(Plug'n'Play) 살펴보기
date: '2023-02-26'
tags: ['yarn']
draft: false
summary: PnP(Plug'n'Play)란 무엇일까?
---

# 들어가며

최근 사내 일부 서비스를 Yarn berry 기반의 모노레포 형식으로 마이그레이션하며 Yarn berry에 대한 내용을 찾아봤습니다. 여러 Yarn berry 도입기와 공식 문서를 살펴보면, 공통적으로 Yarn berry의 PnP(Plug'n'Play) 방식을 가장 장점으로 내세웁니다.

사실 이 개념을 처음 접했을 때는 이해가 잘 되지 않는 부분이 많아 몇번이고 다시 찾아봤던 기억이 있습니다. 또, 버전 관리를 다루는 내용이다 보니 마이그레이션 과정에서도 알 수 없는 오류들이 여럿 발생했기도 했습니다. Yarn berry 이전 버전에서 어떤 식으로 패키지 관리가 되어 왔고, Yarn berry의 PnP(Plug'n'Play) 방식이 이 한계를 어떻게 해결했는지 정리했습니다.

# NPM v1, v2에서의 node_modules의 모듈 탐색 방식의 한계

`node_modules`는 Node.js 프로젝트에서 필요한 모듈을 포함하는 디렉토리입니다. 우리가 프로젝트에서 npm이나 Yarn과 같은 패키지 매니저를 사용하여 패키지를 설치하면 해당 패키지가 프로젝트의 `node_modules`에 저장됩니다.

`require()`나 import 문을 사용하여 module을 로드할 때, Node.js는 resolution 알고리즘에 따라 현재 파일 디렉토리를 기준으로 `require()`할 모듈을 찾습니다. 만약 현재 디렉토리의 node_modules 디렉토리에서 모듈을 찾지 못하면, Node.js는 부모 디렉토리의 node_modules 디렉토리에서 모듈을 찾습니다. 이 과정은 모듈이 찾아질 때까지 반복되며, 모듈이나 파일 시스템의 루트 디렉토리에 도달할 때까지 계속됩니다.

## 비효율적인 파일 탐색 구조

node_modules에서 모듈을 찾을 때 루트 디렉토리로 타고 올라가면서 찾는 방식은 파일 시스템의 디렉토리를 탐색하고 파일들을 읽어오는 I/O 작업을 수행합니다. I/O 작업은 컴퓨터 시스템에서 상대적으로 느린 작업에 속하기에, node_modules 탐색에 비효율적인 I/O 작업을 수행하는 것은 성능상 좋지 않습니다.

## 중복 모듈 탐색

node_modules의 내 모듈 탐색 방식은 탐색 범위가 넓고, 동일한 모듈이 여러 디렉토리에 중복으로 설치됐을 때도 중복 모듈까지 모두 탐색하기 때문에 비효율적입니다.
또한, node_modules는 프로젝트의 모든 종속성 패키지와 그에 따른 하위 종속성들을 포함하고 있기 때문에 프로젝트의 규모와 사용되는 패키지 수가 증가할수록 모듈을 탐색하는 비용이 더욱 커지며, 프로젝트의 빌드 및 실행 시간 역시 증가하게 됩니다.

# yarn v1에서의 node_modules의 한계

NPM v1, v2에서의 node_modules의 모듈 탐색 방식의 한계를 개선하기위해 yarn v1가 적용한 방식은 다음와 같은 한계를 가집니다.

## 유령 의존성(phantom dependency)

npm v3 이전 버전에서는 각 패키지마다 종속성을 별도로 설치해서 동일한 버전의 종속성이 여러 번 중복으로 설치했습니다. 이를 개선하기 위해, 2017년 9월 릴리스 된 Yarn v1은 npm v3에서 사용되는 호이스팅 개념을 적용했습니다. 호이스팅은 여러 패키지에서 동일한 버전의 종속성이 필요한 경우 해당 버전을 프로젝트의 루트 레벨에 한 번만 설치하고, 각 패키지에서는 해당 버전을 참조하는 방식으로 종속성을 관리하는 개념입니다.
![](https://velog.velcdn.com/images/dami/post/e7febbcf-7622-44c8-b3c2-33c28f63fc24/image.png)

예를 들어 위 종속성 트리에서 처럼 패키지 A가 패키지 B를 의존성으로 가지고 있고, 패키지 C가 패키지 A를 의존성으로 가지고 있을 때,

```
패키지 A:
- 의존성: 패키지 B@1.0.0

패키지 C:
- 의존성: 패키지 A@1.0.0
           ㄴ패키지 B@1.0.0
```

호이스팅이 적용된 경우 패키지 A, B가 프로젝트의 루트에 한 번만 설치되며 다른 패키지에서 A, B를 공유할 수 있는 구조로 바뀌게 됩니다. 호이스팅에 따라 직접 의존하고 있지 않는 패키지를 참조할 수 있는 현상을 유령 의존성이라 부릅니다. Yarn v1는 이러한 호이스팅 개념이 적용되며, 동일한 패키지를 중복해서 설치하는 문제를 해결할 수 있었으나, 유령 의존성으로 인해 의존성 관리에 혼동을 가져올 수 있다는 문제를 가지고 있었습니다.

## 버전 충돌 문제

프로젝트 모듈은 서로 의존성을 가지기에 모든 모듈이 동일한 버전의 의존성을 사용하는 것이 중요합니. 의존성 버전 관리가 제대로 되지 않으면 예기치 않은 동작이 발생할 수 있으며, 버그 발생 및 버전 충돌 가능성이 높아지기 때문입니다.

예를 들어, 프로젝트 A와 B가 각각 lodash 모듈의 버전 3.x.x를 사용하고 있을 때, 프로젝트 C에서 이 두 모듈을 모두 의존성으로 가지고 있다면, 프로젝트 C는 어떤 버전의 lodash를 사용해야 하는지 결정할 수 없는 경우가 생깁니다.

만일 package.json파일 내에 latest, ^x.x.x, ~x.x.x등 명확한 기준 없이 패키지 버전 관리가 이루어 졌을 때, 각 개발자들이 패키지를 install 했을 때 node_modules 의 의존성이 다르게 설치될 위험성이 있습니다.

# yarn berry의 PnP(Plug'n'Play)

Yarn이 모듈이 있는 위치를 알고 있고, 의존성도 관리할 수 있으면 어떨까요? Yarn은 Yarn의 version 2인 Yarn berry에 PnP(Plug'n'Play) 방식 도입했습니다. PnP는 앞서 설명한 node_modules 디렉토리에 패키지를 설치하지 않고, `.pnp.cjs`을 통해 의존성을 로드하는 방식입니다.

`.pnp.cjs` 파일에는 패키지 이름과 버전을 디스크의 해당 위치에 연결하고 패키지 이름과 버전을 종속성 목록에 연결하는 정보를 가지고 있습니다.

Yarn은 `.pnp.cjs`에서 패키지의 위치를 node에 알려줄 수 있고, node는 해당 정보를 가지고 패키지를 즉시 찾아 설치할 수 있게 됩니다.

## node_modules 폴더 내 react

![](https://velog.velcdn.com/images/dami/post/7d37b218-88cb-48e8-9b4e-196645eaa988/image.png)

먼저 node_modules을 살펴보겠습니다. node_modules의 경우 react 폴더 내에 react가 의존하는 패키지가 중첩으로 들어있는 구조를 가집니다.

## .pnp.cjs 내 react 정보

```
       ["react", [
        ["npm:17.0.2", {
          "packageLocation": "./.yarn/cache/react-npm-17.0.2-99ba37d931-b254cc17ce.zip/node_modules/react/",
          "packageDependencies": [
            ["react", "npm:17.0.2"],
            ["loose-envify", "npm:1.4.0"],
            ["object-assign", "npm:4.1.1"]
          ],
          "linkType": "HARD"
        }]
      ]],

```

반면 `.pnp.cjs` 내 react 정보는 react 버전, 패키지 위치(packageLocation), 패키지 의존성(packageDependencies), linkType이 기재되어 있습니다. node는 이 정보를 기반으로 react와 react 설치에 필요한 의존성을 즉시 찾을 수 있습니다.

## .yarn/cache 폴더 내 react

![](https://velog.velcdn.com/images/dami/post/57e18b82-a09e-4eaa-baa8-f7c9b6ae6714/image.png)
실제 `.yarn/cache` 폴더를 확인해보면 `.pnp.cjs` 내 기입된 해시값과 동일한 해시값을 가진 `.zip` 파일이 존재하는 것을 알 수 있습니다.

# zero-install

Yarn berry를 사용하는 경우 패키지 설치를 하지 않아도 되는 zero-install 기능을 사용할 수 있습니다. zero-install을 사용하기 위해 .yarn/cache 폴더와 `pnp.cjs` 파일을 git reository에 업로드 하면 됩니다.

zero-install을 적용한다면, 오프라인 상태에서 혹은 프로젝트를 clone 하거나 브랜치 전환을 할 때도 install 없이 프로젝트를 실행할 수 있습니다. 또한 .yarn/cache 폴더를 git repository에 업로드하여 버전 관리에 포함시킴으로써 의존성도 git으로 관리할 수 있습니다.

참고) .gitignore
zero-install 사용, 미사용 시의 .gitignore 형태는 다음과 같습니다.

## zero-install 사용시 .gitignore

```
.yarn/\*
!.yarn/cache
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
```

## zero-install 미사용시 .gitignore

```
.pnp._
.yarn/_
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
```

# 마치며

Yarn v1, NPM에서의 node_modules가 가지는 한계와 이를 개선하기 위한 아이디어로 Yarn v2가 도입한 PnP 개념에 대해 살펴보았습니다.
Yarn v1에서 berry로 마이그레이션하며, 모든 패키지가 zip으로 관리되고 캐싱되어 패키지 설치가 매우 빨라진 것을 체감하였고, 패키지를 node_modules에 설치하지 않기에, 프로젝트 용량 역시 크게 감소한 것을 확인할 수 있었습니다.

물론 단점도 존재했습니다. 필자가 느낀 단점은 버전 관리에 포함해야 하는 파일 수가 많다 보니 commit, push에 시간이 오래 걸리고 Pull Request 내 diff가 지나치게 길어진다는 점이었습니다. 사실 그 외에 아직 크게 겪은 문제는 없으나, 특정 패키지는 Yarn Berry에 호환이 안 되는 문제가 있다고도 합니다. 이 이슈는 발생하면 추가해보도록 하겠습니다.

# 참고

- [yarnpkg 공식문서](https://yarnpkg.com/)
- [loading-from-node_modules-folders](https://nodejs.org/api/modules.html#loading-from-node_modules-folders)
- [node_modules로부터 우리를 구원해 줄 Yarn Berry](https://toss.tech/article/node-modules-and-yarn-berry)
- [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/)
