---
title: Eslint 사용하여 import 순서 자동 정렬 하기
date: '2023-06-18'
tags: ['Eslint']
draft: false
summary: Eslint의 eslint-plugin-import를 사용하여 import 순서 규칙을 생성하는 방법을 알아봅니다.
---

# 들어가며

코드를 작성하다보면 외부 라이브러리나 내부 모듈을 불어오기 위해 `import` 구문을 작성하게 됩니다. 이때 `import` 구문을 사용한 순서대로 `import` 문을 배치한다면, `import` 문이 길어질 때 해당 파일에 어떤 모듈이 어디서 `import` 되고 있는지 파악하기 어려울 때가 있습니다.

물론 `import` 문의 순서를 신경쓰지 않을 수도 있지만 개인적으로 규칙없이 작성되어 있는 `import` 문을 보면 규칙에 맞게 정리를 하고 싶은 생각이 들어 일일히 `import` 위치를 정리하곤 했는데요. Eslint rules를 통해 `import` 구문에 적용할 규칙을 생성하는 방법에 대해 알아보겠습니다.

# Eslint

ESLint는 자바스크립트 코드에서 발견되는 문제시되는 패턴들을 식별하기 위한 정적 코드 분석 도구입니다. 코드를 분석해서 문법에 맞지 않거나 안티 패턴으로 코드를 작성한 경우, 수정이 필요한 내용을 에러 메세지로 제공하고, fix 기능을 통해 Eslint 규칙에 맞게 코드를 수정할 수도 있습니다. Eslint에서 제공하는 다양한 플러그인의 조합으로 팀 컨벤션에 적합한 규칙을 직접 만들 수도 있고, 에어비엔비 등에서 권장하는 규칙을 가져다 사용할 수도 있습니다. Eslint를 사용하면 이렇게 정해진 규칙에 맞게 코드를 작성하도록 강제하기에 협업을 하는 팀원들이 일관되게 코드를 작성할 수 있습니다.

## `import` 순서

`import` 순서 규칙을 생성하기 전, 팀원분들과 어떤 순서로 `import` 구문을 생성할지에 대해 논의하였습니다. 그 결과 저희가 생각하는 중요도 순으로 `import` 구문을 작성하고, 각 그룹 사이에는 공백을 삽입하기로 결정했습니다. `import` 구문 순서 규칙은 아래와 같습니다.

- library
- store
- page component
- component
- hook
- util
- constant
- type

위 순서에 맞게 `import` 구문 작성을 강제하기 위해서는 규칙에 맞지 않는 `import` 구문을 작성했을 때 에러가 발생해야하고, 규칙에 맞게 자동 정렬해주는 도구가 필요합니다. 이때 필요한 도구가 Eslint입니다. Eslint의 eslint-plugin-import 규칙을 사용하면 `import` 구문 규칙을 커스터 마이징해서 적용할 수 있습니다.

# eslint-plugin-import

eslint-plugin-import는 ES2015+(ES6+) `import`/`export` 구문의 린트를 지원하고 파일 경로 및 `import` 이름의 철자가 틀린 문제를 방지하는 용도로 사용하는 규칙입니다.

## 설치하기

우선 eslint-plugin-import를 개발 의존성으로 설치합니다.

```js
npm install eslint-plugin-import --save-dev
// or
yarn add -D eslint-plugin-import
```

## `import` 규칙 적용하기

eslint-plugin-import을 eslint rules로 사용하겠다는 설정을 하기위해 Eslint의 configuration 파일인 `.eslintrc` 파일을 수정하는 방법에 대해 알아보겠습니다.

## `import` 규칙 커스터마이징

eslint-plugin-import 규칙을 사용하기 위해 먼저 `.eslintrc` 내 `rules` 속성에 ``import`/order` 규칙을 추가해줍니다.

```js
module.`export`s = {
...
  rules : {
    '`import`/order': [
      ...
    ],
  }

}
```

이제 ``import`/order` 배열에 옵션을 커스터 마이징할 수 있습니다.

### `groups: [array]:`

`import` group의 순서를 정할 수 있는 옵션이며 기본값은 `["builtin", "external", "parent", "sibling", "index"]` 입니다.
`groups` 옵션은 `string` 또는 `string` 배열로 작성해야 하며, 이때 허용되는 `string`에는 `"builtin", "external", "internal", "unknown", "parent", "sibling", "index", "object", "type"`이 있습니다. 내가 원하는 group 명으로 새로 정의할 수 있으면 좋겠지만, `groups` 배열에는 반드시 허용되는 group 명 만을 사용해야 합니다. 각 group 타입이 의미하는 범위는 다음과 같습니다.

- builtin
- external
- internal
- unknown
- parent
- sibling
- index
- object
- type

위 타입을 사용하여 아래와 같은 `groups` 규칙을 작성한 경우, `import` 문은
builtin그룹, sibling과 parent 그룹, index 그룹, object 그룹 순으로 작성되어야 하며, 각 그룹 사이에는 공백이 삽입되어야 합니다.

```js
;[
  'builtin', // Built-in types are first
  ['sibling', 'parent'], // Then sibling and parent types. They can be mingled together
  'index', // Then the index file
  'object',
  // Then the rest: internal and external type
]
```

### `pathGroups: [array of objects]:`

그룹으로 정의할 경로의 패턴을 직접 정의하여 그룹화하여 앞서 작성한 `groups` 배열을 기준으로 배치할 수 있습니다. 예를 들어 `import` 문에 react가 들어가는 경우 이 경로를 상단에 배치하기 위한 규칙은 아래와 같이 작성할 수 있습니다. ``import` React from 'react'`와 같이 react가 들어가는 경로는 `builtin` 그룹전에 배치하는 규칙입니다.

```js
     pathGroups: [
          {
            pattern: 'react',
            group: 'builtin',
            position: 'before',
          },
       ...
```

`pattern`을 활용하면 특정 경로의 순서도 `pathGroups`으로 정의할 수 있습니다. 앞서 잠깐 언급했던 `import` 구문 순서 규칙에 따라 `pathGroups`를 정의해보겠습니다.

```js
      pathGroups: [
          {
            pattern: 'react',
            group: 'builtin',
            position: 'before',
          },
          { pattern: 'react-dom', group: 'builtin', position: 'before' },
          { pattern: 'mobx-react-lite', group: 'external', position: 'after' },
          {
            pattern: '@/stores/**/*',
            group: 'external',
            position: 'after',
          },
          {
            pattern: '@/components/**/*',
            group: 'internal',
          },
          {
            pattern: '@/constants/**/*',
            group: 'unknown',
            position: 'before',
          },
          {
            pattern: '@/utils/**/*',
            group: 'unknown',
          },
        ],
```

### `alphabetize: {order: asc|desc|ignore, order`import`Kind: asc|desc|ignore, caseInsensitive: true|false}:`

같은 group내 `import` 문을 정렬할 땐 어떤 순서로 정렬해보면 좋을까요? 기본적으로 알파벳순으로 순서를 정렬해볼 수 있겠습니다. alphabetize 속성은 그룹 내 `import` 문을 경로에 따라 알파벳순으로 정렬하는 규칙입니다.

- `order`: `asc`를 사용하여 오름차순으로 정렬하고 `desc`를 사용하여 내림차순으로 정렬합니다. (기본값: ignore).
- `order`import`Kind`: `asc`를 사용하여 다양한 `import` 문을 오름차순으로 정렬합니다. 동일한 `import` 문 사용하여 type 또는 typeof 접두사가 붙은 `import` 문을 내림차순으로 정렬하려면 desc를 사용합니다. (기본값: ignore).
- `caseInsensitive`: 대소문자를 무시하려면 true를 사용하고 대소문자를 고려하려면 false를 사용합니다(기본값: false).

위 설정값을 토대로 대소문자를 무시하고, 알파벳 오름차순으로 정렬하는 설정을 추가해보겠습니다.

```js
    alphabetize: {
          order: 'asc',
          caseInsensitive: true,
        },
```

# 마치며

Eslint rules를 통해 `import` 구문에 적용할 규칙을 생성하는 방법에 대해 알아봤습니다. `import` 구문 순서를 어떻게 할 건지에 대한 규칙은 있으나, 직접 순서를 조정하고 계시다면 eslint-plugin-import 규칙 도입이 비효율적인 작업을 줄여줄 수 있는 대안책이 될 수 있으니 참고해보시면 좋을 것 같습니다.

# 참고

- [eslint-plugin-import](https://github.com/`import`-js/eslint-plugin-import/blob/main/docs/rules/order.md)
