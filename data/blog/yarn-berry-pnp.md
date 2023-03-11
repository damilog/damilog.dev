---
title: Yarn Berry의 PnP(Plug'n'Play) 살펴보기
date: '2023-02-26'
tags: ['yarn']
draft: false
summary: PnP(Plug'n'Play)란 무엇일까?
---

## node_moudules에서 module을 로딩하는 법

`node_modules`는 Node.js 프로젝트에서 필요한 모듈을 포함하는 디렉토리이며, npm이나 yarn과 같은 패키지 매니저를 사용하여 프로젝트의 의존성(dependencies)을 관리할 때 자동으로 생성된다.

프로젝트에서 패키지 매니저를 통해 패키지 install을 실행할 때 노드의 파일 시스템(file system)은 resolution algorithm에 따라 현재 디렉토리에서 require()할 모듈을 찾고, 찾을 수 없다면 파일 시스템의 root에 도달할 때까지 상위 디렉토리로 이동하며 모듈을 찾는다.

예를 들어 '/home/ry/projects/foo.js'에 있는 파일이 require('bar.js')을 호출한다면 Node.js는 아래 순서로 모듈을 찾는다.

```
/home/ry/projects/node_modules/bar.js
/home/ry/node_modules/bar.js
/home/node_modules/bar.js
/node_modules/bar.js
```

이러한 node_modules 구조는 몇 가지 한계를 가진다.

## node_modules의 한계

### 비효율적인 탐색 구조

node_modules는 모듈의 종류와 개수에 따라 용량을 많이 차지하게 되고 이로 인해 패키지 install 시간 중 70% 이상의 시간이 node_modules를 생성하는 데 쓰이기도 한다.

node_modules의 용량이 커짐에 따라 의존성 탐색도 느려질 수 밖에 없다.노드의 파일 시스템은 필요한 모든 단일 파일을 로드할 위치를 파악하기 위해 많은 stat 및 readdir 호출을 수행하게 된다. 이에 따라 많은 모듈이 node_modules 디렉토리에 포함되어 있을 때는 그만큼 파일 시스템에서 모듈을 빠르게 검색하거나 읽기 어려워진다.

### 유령 의존성(phantom dependency)

node_modules 종속성 구조를 효율적으로 다루기 위하여, npm v3에서는 호이스팅(hoisting) 개념이 도입되었다. 호이스팅은 node_modules의 모든 종속성을 하나의 폴더 수준으로 끌어 올려 중첩으로 인한 중복성을 줄이는 방법이다.

![](https://velog.velcdn.com/images/dami/post/8390df89-835c-40e3-b045-73b8dd3dfdf3/image.png)

위 그림 속 npm v3의 구조와 같이 호이스팅으로 인해 node_modules가 플랫한 구조가 되면 프로젝트에서 package.json에 정의 하지 않은 패키지를 사용할 수 있는 '유령 의존성(phantom dependency)'이 생길 수 있다.

### 버전 충돌 문제

프로젝트 모듈은 서로 의존성을 가지기에 모든 모듈이 동일한 버전의 의존성을 사용하는 것이 중요하다. 의존성 버전관리가 제대로 되지 않으면 예기치 않은 동작이 발생할 수 있으며, 버그발생 및 버전 충돌 가능성이 높아지기 때문이다.

예를 들어, 프로젝트 A와 B가 각각 lodash 모듈의 버전 3.x.x를 사용하고 있을 때, 프로젝트 C에서 이 두 모듈을 모두 의존성으로 가지고 있다면, 프로젝트 C는 어떤 버전의 lodash를 사용해야 하는지 결정할 수 없는 경우가 생긴다.

## yarn berry의 PnP(Plug'n'Play)

Yarn이 모듈이 있는 위치를 알고 있고, 의존성도 관리할 수 있으면 어떨까?

Yarn berry(Yarn v2)는 PnP(Plug'n'Play) 방식으로 다양한 패키지의 복사본을 포함하는 node_modules 디렉토리를 사용하지 않고, .pnp.cjs을 통해 의존성을 로드할 수 있도록 했다.

`.pnp.cjs` 파일에는 패키지 이름과 버전을 디스크의 해당 위치에 연결하고 패키지 이름과 버전을 종속성 목록에 연결하는 정보를 가지고 있다.

Yarn은 `.pnp.cjs`에서 패키지의 위치를 node에 즉시 알려줄 수 있게 되고 node는 해당 정보를 가지고 패키지를 즉시 찾아 설치할 수 있게 된다.

### node_modules 폴더 내 react

![](https://velog.velcdn.com/images/dami/post/7d37b218-88cb-48e8-9b4e-196645eaa988/image.png)

먼저 node_modules을 살펴보자. node_modules의 경우 react 폴더 내에 react가 의존하는 패키지가 중첩으로 들어있는 구조를 가진다.

### .pnp.cjs 내 react 정보

```js
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

반면 .pnp.cjs 내 react 정보는 react 버전, 패키지 위치(packageLocation), 패키지 의존성(packageDependencies), linkType이 기재되어 있다.

node는 이 정보를 기반으로 react와 react 설치에 필요한 의존성을 즉시 찾을 수 있다.

- linkType

```typescript
export enum LinkType {
  HARD = `HARD`,
  SOFT = `SOFT`,
}
```

- HARD: The package manager owns the location (typically things within the cache) and can transform it at will (for instance the PnP linker may decide to unplug those packages).
- SOFT: The package manager doesn't own the location (symlinks, workspaces, etc), so the linkers aren't allowed to do anything with them except use them as they are.

### .yarn/cache 폴더 내 react

![](https://velog.velcdn.com/images/dami/post/57e18b82-a09e-4eaa-baa8-f7c9b6ae6714/image.png)

- 실제 .yarn/cache 폴더를 확인해보면 .pnp.cjs 내 기입된 해시값과 동일한 해시값을 가진 zip 파일이 존재하는 것을 알 수 있다.

## zero-install

Yarn berry를 사용하는 경우 패키지 설치를 하지 않아도 되는 zero-install 기능을 사용할 수 있다. zero-install을 사용하기 위해 .yarn/cache 폴더와 pnp.cjs 파일을 git reository에 업로드 하면 된다.

zero-install을 적용한다면, 오프라인 상태에서 혹은 프로젝트를 clone 하거나 브랜치 전환을 할 때도 install 없이 프로젝트를 실행할 수 있다. 또한 .yarn/cache 폴더를 git repository에 업로드하여 버전 관리에 포함시킴으로써 의존성도 git으로 관리할 수 있다.

물론 단점도 존재한다. 내가 느낀 단점은 버전 관리에 포함해야 하는 파일 수가 많다 보니까 commit, push에 시간이 오래 걸린다는 점이었다. 추후 단점을 더 발견한다면 추가해보겠다.

### 참고) .gitignore

**zero-install 사용시 .gitignore**

```
.yarn/*
!.yarn/cache
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
```

**zero-install 미사용시 .gitignore**

```
.pnp.*
.yarn/*
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions
```

# 참고

- [yarnpkg 공식문서](https://yarnpkg.com/)
- [loading-from-node_modules-folders](https://nodejs.org/api/modules.html#loading-from-node_modules-folders)
- [node_modules로부터 우리를 구원해 줄 Yarn Berry](https://toss.tech/article/node-modules-and-yarn-berry)
- [Phantom dependencies](https://rushjs.io/pages/advanced/phantom_deps/)
