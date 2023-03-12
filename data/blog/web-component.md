---
title: Web Component 개념 살펴보기
date: '2023-03-12'
tags: ['web']
draft: false
summary: Web Component의 개념과 이를 구성하는 주요 기술을 살펴보자.
---

`컴포넌트`는 React, Vue 등의 SPA 라이브러리를 사용해본 개발자에겐 익숙한 개념일텐데요.
Web API에도 HTML 태그를 컴포넌트로 다룰 수 있는 기술이 있습니다.

웹 컴포넌트(Web Component)는 복잡한 HTML 코드를 캡슐화하여 재사용 가능한 커스텀 엘리먼트를 생성하고, Web/App에서 활용할 수 있도록 하는 여러 기술들의 모음입니다.

웹 컴포넌트를 만들기 위해서 알아야 할 몇 가지 개념을 살펴봅시다.

# 웹 컴포넌트(Web Component)를 구성하는 주요 기술

웹 컴포넌트 기술들은 DOM을 OOP 대상으로 바라볼 수 있게 해줍니다.

- `Custom Elements`: HTML element를 어떤 이름을 가진 태그로 정의하는 방법
- `Shadow DOM`: DOM에서 외부의 영향을 받지 않고, 독립적으로 작동하는 DOM
- `HTML Template`: HTML 마크업 템플릿의 임시 저장소

## Custom Element

```js
class MyComponent extends HTMLElement {...}
customElements.define("my-component", MyComponent)
```

`Custom Element`는 class로 정의를 해야 하는데, 이때 위와 같이 `HTMLElement`을 extends합니다. 위 예시 속 `customElements.define`의 `customElements`는 무엇일까요?

`window.customElements`는 window 내장 객체이며, 호출 시 `CustomElementRegistry` 객체의 참조(reference)를 리턴합니다. 즉, `customElements.define`는 `customElements`의 자체 메서드가 아닌 `customElements`가 리턴한 `CustomElementRegistry` 객체가 가진 메서드인 것이죠.

### 생명 주기 콜백 (lifecycle callbacks)

앞서 Custom Element는 class로 정의해야 한다고 하였는데요. class로 정의된 Custom Element의 내부에는 Custom Element의 생명주기를 다루는 콜백(callback)을 정의할 수 있습니다.

```js
class MyComponent extends HTMLElement {
  // element 생성시
  constructor() {
    super();
    ...
  }

  // connectedCallback: component가 DOM에 추가 되면 가장 먼저 불리워짐
  connectedCallback() {
    .
  }

  // disconnectedCallback: MyComponent에 element가 제거될 때 마다 호출
  disconnectedCallback() {
    ...
  }

  // observedAttributes: 변경을 감지할 attribute들을 모니터링하기 위한 메서드
  static get observedAttributes() {
    return [...];
  }

  // attributeChangedCallback: observedAttributes에서 모니터링중인 attribute가 변경되었을 때, 호출
  attributeChangedCallback(name, oldValue, newValue) {
    ...
  }

  // adoptedCallback: element가 새로운 document로 이동될 때마다 호출
  adoptedCallback() {
    ...
  }
}
```

### 사용법

**1. Custom Element class를 선언합니다.**

component가 DOM에 추가 되면 가장 먼저 불리워지는 메서드인 `connectedCallback` 내부에 생성하고자 하는 element를 작성해줍니다.

```js
class MyComponent extends HTMLElement {
  connectedCallback() {
    const BoxOne = document.createElement('div')
    BoxOne.innerText = '안녕'
    this.appendChild(BoxOne)

    const BoxTwo = document.createElement('div')
    BoxTwo.innerText = '하세요'
    this.appendChild(BoxTwo)
  }
}
```

**2. Custom Element 정의하기**

```js
customElements.define('my-component', MyComponent)
```

**3. 정의한 Custom Element 사용하기**

```js
<my-component></my-component>
```

**4. 결과**
![](https://velog.velcdn.com/images/dami/post/84142bab-ef44-406b-bcb1-0d78eda4263e/image.png)

## HTML Template & Slot

### Template

`<template>` 태그는 HTML 마크업을 정의해 놓고 실제 사용하고자 하는 마크업에 가져다 쓸 수 있도록 하는 HTML 임시 보관소입니다.

```js
//template 코드 예시
    <template id="myGreeting">
      <div>반갑습니다</div>
      <div id="greeting">안녕하세요</div>
      <style>
        #greeting {
          color: blue;
        }
      </style>
    </template>


//사용 예시
 this.shadowRoot.append(myGreeting.content.cloneNode(true));
```

### Slot

slot은 컴포넌트 내 HTML 태그를 위치 시킬 슬롯을 나타내는 용도로 쓰입니다.

```js
<script>
customElements.define('user-card', class extends HTMLElement {
  connectedCallback() {
    this.attachShadow({mode: 'open'});
    this.shadowRoot.innerHTML = `
      <div>Name:
        <slot name="username"></slot>
      </div>
      <div>Birthday:
        <slot name="birthday"></slot>
      </div>
    `;
  }
});
</script>

<user-card>
  <span slot="username">John Smith</span>
  <span slot="birthday">01.01.2001</span>
</user-card>
```

![](https://velog.velcdn.com/images/dami/post/1ad5f2a5-2126-47f4-9561-317ad34dd18c/image.png)

![](https://velog.velcdn.com/images/dami/post/b224cebe-eb38-4f6f-adc5-808966d016bb/image.png)

## Shadow DOM

Shadow DOM을 사용하면 DOM에 비밀 공간을 만들 수 있습니다. Shadow DOM 트리는 shadow root로부터 시작되어 원하는 모든 요소의 안에 부착될 수 있는데요. Shadow DOM은 일반적인 DOM과 조작 방법(자식 요소 추가나 스타일을 넣는 등)에서 차이가 없지만, **shadow DOM 내부의 어떤 코드도 외부에 영향을 줄 수 없는 캡슐화**를 허용한다는 차이점이 있습니다.

![](https://velog.velcdn.com/images/dami/post/fa75e616-9099-43c5-8b50-11545c9418cd/image.png)

> Flattened Tree (for rendering): (렌더링을 위해) 평평해진 트리

- **Shadow host**: shadow DOM이 부착되는 통상적인 DOM 노드.
- **Shadow tree**: shadow DOM 내부의 DOM 트리.
- **Shadow boundary**: shadow DOM이 끝나고, 통상적인 DOM이 시작되는 장소.
- **Shadow root**: shadow 트리의 root 노드

### 사용법

앞서 생성한 Custom Element가 DOM에 추가 되면 가장 먼저 불리워질 connectedCallback 메서드 내부에`this.attachShadow({ mode: "open" })`를 사용하여 shadow DOM을 생성해줍니다. 생성한 shadow DOM에 myGreeting template을 복제해서 부착해줍니다.

```html
<body>
  <my-component name="daisy"></my-component>
  <template id="myGreeting">
    <div>안녕하세요</div>
    <div id="greeting">반갑습니다.</div>
    <style>
      #greeting {
        color: blue;
      }
    </style>
  </template>

  <script>
    class MyComponent extends HTMLElement {
      connectedCallback() {
        this.attachShadow({ mode: 'open' })
        this.shadowRoot.append(myGreeting.content.cloneNode(true))
      }
    }

    customElements.define('my-component', MyComponent)
  </script>
</body>
```

![](https://velog.velcdn.com/images/dami/post/42493d0a-0fba-468f-b80a-c74595d3a09f/image.png)

개발자도구로 살펴보면 shadow DOM이 부착된 my-component에는 #shadow-root가 open 된 것을 확인할 수 있습니다.

이렇게 shadow DOM을 사용하면 shadow DOM 내부의 코드가 외부 DOM 요소에 영향을 미치지 않기에 캡슐화가 가능합니다. 예를 들어 shadow DOM 내부에서 정의한 스타일은 shadow DOM 내부에만 영향을 미칠 뿐 shadow DOM 바깥에 있는 element에는 영향을 미치지 않습니다.

## custom element + shadow DOM 조합으로 컴포넌트 만들기

앞서 살펴본 Web Component를 구성하는 요소를 조합한다면, Web API 만드로도 캡슐화, 재사용성을 가진 컴포넌트를 구현할 수 있습니다.

1. HTMLElement를 상속받은 클래스에 element를 정의합니다.
2. shawdow root를 만들어 붙입니다.
3. shadow DOM 구조를 만듭니다.
4. shadowDOM에 스타일 코드 작성합니다.
5. shadow root에 shadow DOM을 붙입니다.
6. custom element 사용합니다.

# 마치며

웹 컴포넌트를 구성하는 여러 요소를 살펴보았습니다. React 이전에 이미 웹 컴포넌트 만으로도 컴포넌트 Life cycle을 섬세하게 다룰 수 있는 기술이 존재했다는 점이 인상 깊었습니다. React의 소중함도 알게 되었네요.

# 참고

- [Using custom elements](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements)
- [slots-composition](https://javascript.info/slots-composition)
- [Custom Elements v1 - Reusable Web Components](https://web.dev/custom-elements-v1/)
