---
title: React.FC 살펴보기
date: '2022-09-18'
tags: ['TypeScript']
draft: false
summary: React.FC의 타입 특성을 살펴봅시다.
---

리액트의 컴포넌트를 정의하는 방법은 `JSX.Element`, `React.Node` 등 여러 가지 타입이 있습니다. 각 타입에 대해 살펴보다 보니 유독 `React.FC` 사용에 대한 문제점을 이야기하는 글이 많아 이번 글에서는 `React.FC`의 특징을 살펴보도록 하겠습니다.

## React.FC
`React.FC`의 FC는 FunctionComponent 타입의 줄임말로 리액트에서 제공하는 함수형 컴포넌트 선언을 위한 타입입니다.

### 사용법
아래와 같이 Props타입을 정의하여 `React.FC` 제네릭으로 넣어주면 됩니다.

```tsx
import { FC } from 'react'

type GreetingProps = {
  name: string
}

const Greeting: FC<GreetingProps> = ({ name }) => {
  return <h1>Hello {name}</h1>
}

```

## 1. 암시적 children props

> 이 내용은 리액트 18 업데이트 후 FC의 암시적인 children이 삭제되었습니다. 업데이트 사항은 이 [PR](https://github.com/DefinitelyTyped/DefinitelyTyped/pull/56210)에서 확인할 수 있습니다.


FC는 다음과 같은 구조로 되어 있습니다.

```tsx
type FC<P = {}> = FunctionComponent<P>

interface FunctionComponent<P = {}> {
  (props: PropsWithChildren<P>, context?: any): ReactElement<any, any> | null
  propTypes?: WeakValidationMap<P> | undefined
  contextTypes?: ValidationMap<any> | undefined
  defaultProps?: Partial<P> | undefined
  displayName?: string | undefined
}
```
위 코드 내 props의 `PropsWithChildren` 이라는 타입을 확인 할 수 있는데, 이 `PropsWithChildren` 타입은 아래와 같은 형태로 정의되어 있습니다. 

```tsx
type PropsWithChildren<P> = P & { children?: ReactNode | undefined };
```
 `React.FC`는 children이 옵셔널로 가지고 있습니다. `FC`는 children 타입을 기본적으로 가지고 있기 때문에 컴포넌트의 props의 타입이 명백하지 않게 됩니다.  `React.FC`가 자체적으로  children을 옵셔널로 가지고 있기 때문에 Props 타입에  children을 별도로 정의해주지 않아도 타입 에러가 발생하지 않습니다.

 반면 `JSX.Element`와 같은 타입은 암시적으로  children을 가지고 있지 않습니다. `React.FC`와 다르게  children props를 명시적으로 선언해줘야 하기 때문에  children 존재 여부를 확실하게 정의해줄 수 있습니다. `React.FC`이  children을 가지고 있어 별도 정의를 하지 않아도 되니 편할 수도 있으나, `React.FC`가  children을 옵셔널로 가지고 있다는 사실을 모르는 개발자가 `React.FC`로 정의된 컴포넌트를 봤을 때  children이 정의되지 않았음에도  children이 들어와도 타입 에러가 발생하지 않는 상황을 이해하기 어려울 수 있습니다. 



### React.FC로 정의하지 않은 경우
![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FdDQkl4%2FbtrMt4VmO1l%2FwAzrZ8W3bv8E6ADkK9ypd1%2Fimg.png)

일반적인 함수형 컴포넌트 방식으로 작성하면 위와 같이 children을 명시적으로 선언할 것을 제안하는 에러가 발생합니다.

```
Type ErrorProperty 'children' does not exist on type 'IntrinsicAttributes & GreetingProps'.ts(2322)
Type '{ children: Element; name: string; }' is not assignable to type 'IntrinsicAttributes & GreetingProps'.
```
### React.FC로 정의한 경우
![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FdgvXXy%2FbtrMkMIYyKw%2FqVtXDTnHlTKLRtP2AulQBk%2Fimg.png)


 `React.FC`로 정의한 Greeting는 children을 렌더링 하거나 처리하는 코드가 없음에도 위 코드는 정상적으로 처리됩니다. GreetingProps에 children이 정의되어 있지 않으니 오류를 발생시킬 것 같은데 말이죠. `React.FC`가 children props를 암시적으로 가지고 있기 때문입니다. 


## 2. Default Props
React.FC는 React의 Default Props를 사용하는 데에도 어려움이 있습니다.
![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FtdV94%2FbtrMlVZIDAm%2F9OrNufh9AWjEXEJsIZLX5k%2Fimg.png)

 위 예제에서 mark를 defaultProps로 선언했음에도 불구하고, mark값이 존재하지 않다는 이유로 타입 에러가 발생합니다. 

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FyQWFU%2FbtrMlWdfYB8%2FR2RWJgnkjn7fvsqOh3WEA1%2Fimg.png)


반면, React.FC를 제거하면 어떨까요? 위 예제에서 처럼 mark가 필수 props임에도 deafultProps로 이미 정의해줬으니 에러가 발생하지 않습니다.



## 3. 인수를 타이핑하지 않는다.
React.FC는 함수를 타이핑하지만, 인수를 타이핑하지는 않습니다. 그래서 함수 선언식으로 작성된 컴포넌트의 return Type을 React.FC 타입으로 정해줄 수 없습니다.

- 익명 함수를 변수에 할당하는 것 -가능 
```tsx
const Greeting: FC<GreetingProps> = function ({ name }) {
return <h1>Hello {name}</h1>
}
```
- 화살표 함수 - 가능
```tsx
const Greeting: FC<{ name: string }> = ({ name }) => {
  return <h1>Hello {name}</h1>
}
```
- 함수 선언식 - 불가