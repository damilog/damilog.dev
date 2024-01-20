---
title: Cypress 코드 작성할 때 유용한 4가지 개념
date: '2024-01-19'
tags: ['Cypress']
draft: false
summary: Cypress 코드를 작성할 때 알아 두면 좋을 4가지 개념을 살펴봅니다.
---

## 들어가며

Cypress로 E2E 테스트 코드를 작성하다 보면 반복되는 코드나 가독성이 떨어지는 코드가 자주 눈에 띄곤 합니다. 특히 Cypress Studio를 이용해 자동으로 E2E 테스트 코드를 생성하는 경우 그런 코드를 자주 접하게 됩니다.

Cypress는 유지 보수가 쉽고 가독성 좋은 Cypress 코드를 작성하는 데 활용할 수 있는 기능을 제공하는데요. 이번 글에서는 Cypress 코드를 작성할 때 알아 두면 좋을 4가지 개념과 도구를 살펴보겠습니다.

## Retry-ability

Cypress는 기본적으로 특정 명령이 실패할 경우 자동으로 명령을 재시도하는 Retry-ability 기능을 갖추고 있습니다.

예를 들어 다음 코드에서는 버튼이 활성화(`'be.enabled'`)될 때까지 대기한 후 클릭을 수행합니다.

```js
cy.get('button').should('be.enabled').click()
```

만약 버튼이 처음에 활성화되어 있지 않다면, Cypress는 자동으로 재시도하여 버튼이 활성화될 때까지 대기하게 됩니다.

### `retry()` 메소드

Cypress에는 재시도 횟수를 정할 수 있는 `retry()` 메서드가 존재합니다. 예를 들어 다음 코드에서는 `.should('not.exist')` 명령이 실패할 경우 최대 3번까지 재시도합니다.

```js
cy.get('.loading-indicator').should('not.exist').retry(3)
```

## 커스텀 커맨드(Custom Command)

Cypress는 사용자가 직접 함수, 유저 액션을 커맨드로 정의하여 재사용할 수 있는 커스텀 커맨드(Custom Command) 기능을 제공합니다.

로그인 기능을 테스트하는 코드를 작성해 보겠습니다. 로그인 페이지에서 아이디와 패스워드를 제출했을 때 대시 보드가 노출되는 기능과 로그인 후 프로필 페이지로 이동하는 기능을 테스트하는 코드를 작성한다면 다음과 같은 구조가 됩니다.

```js
describe('로그인 테스트', () => {
  it('아이디와 비밀번호를 입력하면 대시보드가 노출된다.', () => {
    // 로그인 페이지 방문
    cy.visit('/login')

    // 아이디와 비밀번호 입력
    cy.get('#id').type('testID')
    cy.get('#password').type('testPassword')

    // 로그인 버튼 클릭
    cy.get('button[type="submit"]').click()

    // 로그인 후 페이지에서의 대시보드 확인
    cy.get('.dashboard').should('be.visible')
  })
  it('로그인 후 프로필 링크를 클릭하면 프로필 페이지로 이동한다.', () => {
    // 로그인 페이지 방문
    cy.visit('/login')

    // 아이디와 비밀번호 입력
    cy.get('#id').type('testID')
    cy.get('#password').type('testPassword')

    // 로그인 버튼 클릭
    cy.get('button[type="submit"]').click()

    // 로그인 후 프로필 페이지 링크를 클릭
    cy.get('.profile-link').click()

    // 프로필 페이지에서의 특정 요소 확인
    cy.get('.profile-header').should('be.visible')
  })
})
```

간단한 기능을 테스트하는 코드이기에 이해가 어려운 부분은 없어 보이네요. 하지만 로그인과 관련된 테스트 코드가 수십 개로 늘어난다면 어떨까요? 혹은 로그인 로직이 바뀌거나, 로그인에 필요한 아이디와 패스워드를 입력하는 element의 id 값이 바뀌면 어떨까요?

### 커맨드 정의: `Cypress.Commands.add()`

Cypress에는 중복 코드를 커맨드로 등록하여 사용할 수 있는 커스텀 커맨드 기능이 존재합니다. command.js 파일을 생성하고, `Cypress.Commands.add` 를 사용하여 원하는 테스트 로직 작성하면 마치 Cypress의 기본 커맨드인 것처럼 호출하여 사용할 수 있습니다. 로그인 페이지로 이동하여 아이디와 패스워드를 입력하여 제출하는 과정을 커스텀 커맨드로 정의해 보겠습니다.

```js
Cypress.Commands.add('login', (id, password) => {
  cy.visit('/login')
  cy.get('#id').type(id)
  cy.get('#password').type(password)
  cy.get('button[type="submit"]').click()
})
```

이제 생성한 `login` 커스텀 커맨드를 로그인 관련 테스트 코드에 적용해 보겠습니다.

```js
describe('로그인 테스트', () => {
  it('아이디와 비밀번호를 입력하면 대시보드가 노출된다.', () => {
    //로그인
    cy.login('myusername', 'mypassword')

    // 로그인 후 페이지에서의 대시보드 확인
    cy.get('.dashboard').should('be.visible')
  })
  it('로그인 후 프로필 링크를 클릭하면 프로필 페이지로 이동한다.', () => {
    //로그인
    cy.login('myusername', 'mypassword')

    // 로그인 후 프로필 페이지 링크를 클릭
    cy.get('.profile-link').click()

    // 프로필 페이지에서의 특정 요소 확인
    cy.get('.profile-header').should('be.visible')
  })
})
```

`cy.login`라는 커스텀 커맨드를 사용하여 중복 코드를 제거함으로써 유지 보수가 쉬운 구조로 개선되었습니다.

### 커맨드 재정의: `Cypress.Command.overwrite()`

Cypress에서 이미 정의되어 있는 visit, get 등의 커맨드를 재정의 하고 싶다면 `Cypress.Command.overwrite()`를 통해 가능합니다. 하지만 기존에 있던 내장 커맨드를 재정의하면 해당 커맨드를 사용하는 모든 테스트 코드에 영향을 미치고, 커맨드 사용에 혼동을 줘 협업에 어려움을 줄 수 있으니 되도록 add 명령어를 통해 커맨드를 새롭게 정의하는 편이 좋습니다.

## Alias

Alias는 선택한 요소나 결과를 변수에 할당하여 나중에 재사용할 수 있도록 하는 기능입니다. Alias를 활용하면 여러 명령어에서 동일한 결과를 사용하거나, 테스트 코드의 가독성을 높일 수 있습니다.

다음 예시에서 `.as('usernameInput')`은 `.get('.username-input')`로 선택한 요소에 대한 별칭(Alias)을 지정합니다.

```js
cy.get('.username-input').type('myusername').as('usernameInput')

// 다른 명령에서 같은 요소에 접근
cy.get('@usernameInput').clear().type('newusername')
```

이후 `cy.get('@usernameInput')`를 사용하여 같은 요소에 접근할 수 있습니다.

또한, Alias로 사용할 이름이 매번 바뀌어야 한다면 다음과 같이 변수에 할당된 값을 사용하는 동적인 Alias를 활용할 수 있습니다.

```js
const username = 'usernameInput'
cy.get('.username-input').as(username)
cy.get(`@${username}`).clear().type('newusername')
```

이처럼 반복적으로 사용되는 요소나 값을 Alias로 지정하여 코드 중복을 방지하고, 코드의 가독성을 높일 수 있습니다.

## Cypress testing Library

Testing Library는 브라우저 환경이나 Node.js에서 사용할 수 있는 일반적인 테스트 유틸리티를 제공하는 라이브러리이며, 주로 DOM 요소에 대한 테스트를 작성할 때 도움이 됩니다.

Testing Librarys는 Cypress도 지원하는데요. 물론 Cypress 문법으로도 충분하지만 사용해 본 결과, Testing Library를 사용하면 조금 더 직관적으로 코드를 작성할 수 있었습니다.

### 설치 및 셋팅

아래 명령어를 통해 Cypress Testing Library를 개발 의존성으로 설치해 줍니다.

```js
npm install --save-dev cypress @testing-library/cypress
```

설치가 완료되면 테스트 파일에 다음 코드를 import 합니다.

```js
import '@testing-library/cypress/add-commands'
```

### Cypress Testing Library 사용 전

로그인 페이지로 이동하고 로그인 버튼이 화면에 보이고 클릭 가능한지를 테스트하는 코드를 작성하면 다음과 같습니다. 로그인 버튼을 버튼의 id 값을 통해 접근하게 됩니다.

```js
it('로그인 버튼이 화면에 보이고 클릭 가능한지 확인한다.', () => {
  cy.visit('/login')
  cy.get('#loginButton').should('be.visible').click()
})
```

### Cypress Testing Library 사용 후

Testing Library에서는 `getByText`라는 커맨드를 제공합니다. `get('#loginButton')`보다 직관적으로 다가오지 않나요? Testing Library의 문법을 적용한 코드는 다음과 같습니다.

```js
import { getByText } from '@testing-library/cypress'

it('로그인 버튼이 화면에 보이고 클릭 가능한지 확인한다.', () => {
  cy.visit('/login')
  getByText('로그인').should('exist').click()
  // 추가적인 테스트 코드...
})
```

또한 Testing Library는 DOM 요소를 검색할 때 HTML 구조에 의존하지 않기 때문에 버튼의 id와 같이 DOM 구조가 변경되는 상황에도 유연하게 작동합니다.
Testing Library는 `getByText`외에도 `findBy`, `findAllBy`, `queryBy` 등과 같이 DOM에 접근에 유용한 직관적인 이름의 메서드를 제공합니다.

## 마치며

지금까지 Cypress 코드를 작성하는 데 도움이 될 4가지 개념 살펴봤습니다.E2E 테스트 코드를 작성하다 보면 테스트할 케이스 및 이벤트의 선후 관계를 반복적으로 정의하는 경우가 많습니다. 위 개념들과 더불어 Cypress 공식 문서에서 제공하는 여러 커맨드들을 적용해보며 유지 보수에 용이한 테스트 코드로 개선해 보면 좋을 것 같습니다.
