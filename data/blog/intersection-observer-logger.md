---
title: Intersection Observer로 뷰포트 기반 로그 전송 설계하기
date: '2025-04-20'
tags: ['web']
draft: false
summary: 뷰포트 기반 로그 전송 설계한 경험을 공유합니다.
---

# 들어가며

현재 개발 중인 서비스에는 유저 액션을 트래킹하기 위한 사내 로그 라이브러리가 적용되어 있는데요.
기존에는 모든 상품 정보를 하나의 API 응답으로 받아 렌더링했지만,
효율성을 높이기 위해 일부 영역을 별도의 API로 분리하는 작업을 진행하였고, 이에 따라 기존 로그를 전송한 뒤 지연 로딩된 영역의 로그를 추가로 전송할 수 있는 기능이 필요해졌습니다.

![상품 정보 중 지연 로딩 영역 확인](/static/images/intersection-observer-logger/img2.png)

사내 로그 라이브러리의 동작 방식은 다음과 같습니다.

1.  사용자의 뷰포트에 보이는 상품의 로그를 전송한다.
2.  스크롤이 멈춘 후 일정 시간이 지나면, 뷰포트에 있는 상품의 로그를 전송한다.
3.  페이지 로딩 직후, 스크롤 이벤트 발생 전에도 로그를 전송한다.
4.  한 번의 pageshow당 상품 로그는 한 번만 전송한다.

단순한 동작처럼 보이지만, 실제로는 로그 업데이트, 초기화, 전송 타이밍 등 다양한 상황을 고려해야 했고 디테일한 테스트가 요구되는 기능이었습니다.
이 글에서는 각 기능 구현 과정과 더불어 추후 유지보수를 위한 테스트 코드 구성 예시에 대해 공유해보려합니다.

# 사용자의 뷰포트에 보이는 상품 구분하기

우선 뷰포트에 노출된 상품을 구분하기 위해 Intersection Observer를 활용했습니다.
Intersection Observer는 관찰할 대상을 지정하여 root 요소의 콘텐츠 영역 내에서 관찰 대상 요소가 얼마나 보이는지를 판별할 수 있는 API입니다. 예를 들어, 아래처럼 특정 상품 요소가 뷰포트에 10% 이상 보일 때 콜백을 실행하도록 설정할 수 있습니다.

```js
useEffect(() => {
  const observer = new IntersectionObserver(
    ([entry]) => {
      if (entry.isIntersecting) {
        console.log('상품이 뷰포트에 들어왔습니다')
      }
    },
    {
      root: null, // 브라우저 뷰포트
      threshold: 0.1, // 10% 이상 보일 때
    }
  )

  const target = document.getElementById('product')
  if (target) observer.observe(target)

  return () => observer.disconnect()
}, [])
```

위 코드에서처럼 root는 직접 설정할 수 있으며, null로 지정하면 브라우저의 뷰포트가 기준이 됩니다.

![상품 리스팅 영역의 뷰포트](/static/images/intersection-observer-logger/img1.png)

그렇다면, 단순히 상품을 뷰포트 관찰 대상에 등록만 하면 끝일까요? 아쉽게도 그렇지 않았습다.

한 번이라도 뷰포트에 노출된 상품의 로그를 전송하는 것이 아니라,
`(1)첫 스크롤 이벤트 발생 이전`, 그리고 `(2)스크롤이 멈춘 이후 시점`에 뷰포트에 노출된 상품의 로그만 전송해야 했기 때문입니다.

이를 구현하기 위해, 뷰포트에 들어온 상품은 Map에 추가하고, 뷰포트에서 사라지면 Map에서 제거하는 방식으로 관리하여 로그 전송 시점에 **실제로 화면에 노출 중인 상품**만 필터링하도록 구현했습니다.

```ts
//... 생략
const useViewportLogger =
  const { addViewportItem, removeViewportItem, isViewportItem } = 전역 상태 관리 스토어
  const { ref, inView } = useInView({ triggerOnce: false, threshold: 0.1 }); //Intersection Observer hook

  useEffect(() => {
    if (inView && !isViewportItem(id, type)) {
      addViewportItem(id, type, logData); //뷰포트에 있으면 Map에 추가
    }

    if (!inView && isViewportItem(id, type)) {
      removeViewportItem(id, type); //뷰포트를 벗어나면 Map에서 제거
    }
  }, [...]);

  return {
    ref,
  };
};
```

또한 이러한 로직은 커스텀 훅(useViewportLogger)으로 분리하고, ref를 리턴하여 관찰 대상인 상품 컴포넌트를 쉽게 설정할 수 있도록 구성했습니다.

```ts
const ItemLink = ({ id, type, logData }) => {
  const { ref: loggerRef } = useViewportLogger({ id, type, logData })

  return <a ref={loggerRef}>...</a>
}
```

# 뷰포트에 있는 상품 로그 보내기

앞서 뷰포트에 노출된 상품들을 필터링했으니, 이제 로그를 어느 시점에 전송해야 하는지 다시 정리해보겠습니다.

- 스크롤이 멈추고 일정 시간이 지난 후
- 페이지 로딩 직후, 스크롤이 발생하기 전
- 한 pageshow 당 상품 로그는 1회만 전송

## 스크롤 이벤트, 어떻게 처리할까?

스크롤 이벤트는 사용자 입력이 발생할 때마다 매우 자주 호출됩니다.
만약 매 스크롤마다 로그를 전송하게 되면,
스크롤 한 번에 수십 번씩 콜백이 실행되어 불필요한 연산이 반복될 수 있습니다.

보통 이 경우, 디바운스(debounce)와 쓰로틀(throttle)을 통해 스크롤 이벤트를 최적화 할텐데요. 저는 사용자가 스크롤을
**마지막 스크롤 이벤트 이후 일정 시간 동안 추가 스크롤이 없을 때**만 로그를 전송해야 했기 때문에 디바운스 방식을 적용했습니다.

```ts
//useViewportLog.ts
const useViewportLog = () => {
  const { handleLog } = 로그 전송 관련 스토어
  const { getViewportItems } = 뷰포트 관련 스토어

  const SCROLL_STOP_DEBOUNCE_MS = 500

useEffect(() => {
  const handleScrollStop = debounce(sendLog, SCROLL_STOP_DEBOUNCE_MS)
  window.addEventListener('scroll', handleScrollStop)

  return () => {
    window.removeEventListener('scroll', handleScrollStop)
    handleScrollStop.cancel()
  }
}, [sendLog])
```

즉, 사용자가 스크롤을 멈춘 뒤 일정 시간(예: 500ms)이 지나야 로그가 전송하여 과도한 로그 전송을 방지하고, 사용자 행위에 기반한 로그를 더 정확하게 수집할 수 있습니다.

또한, useEffect 를 통해 스크롤 이벤트 발생 전 로그는 컴포넌트 마운트 후 보낼 수 있도록 구성했습니다.

```ts
const useViewportLog = () => {
  const { handleLog } = 로그 전송 관련 스토어
  const { getViewportItems } = 뷰포트 관련 스토어

  const WAIT_FOR_RENDER = 1000;

  const sendLog = useCallback(() => {
    const visibleItems = getViewportItems();
    handleLog(visibleItems);
  }, [getViewportItems, handleLog]);

  useEffect(() => {
    const timeoutId = setTimeout(sendLog, WAIT_FOR_RENDER);
    return () => clearTimeout(timeoutId);
  }, [sendLog]);


```

아쉬운 점은, 지연 로드된 영역의 렌더링 완료 시점에 로그를 전송하는 것이 아니라,
단순히 1초의 타이머를 기준으로 로그를 전송하도록 구현했다는 점인데요.

이는 응답 시간이 1초일 경우 timeout 처리되어 화면에 노출되지 않는다는 전제 하에 프론트와 백엔드 간 협의를 통해 설정된 방식이었지만,
지연 로딩이 완료된 정확한 시점을 기준으로 로그를 전송하는 구조로 개선할 필요가 있어 보입니다.

## 한 pageShow 당 상품 로그는 1회만 전송

상품 검색, 새로고침, 외부 페이지에서 페이지 진입, 탭 이동 등 pageshow 이벤트 발생 후 동일 로그를 1회만 전송해야하는 요구사항도 있었는데요. 이 내용은 앞서 뷰포트 상품을 저장했던 것과 동일하게 로그가 전송된 상품 key(상품 id, type 조합)를 Set에 넣고 로그 전송 시점에 상품의 로그 전송 여부를 확인하여 중복 전송을 방지하도록 구현했습니다.

```js
handleLog = (visibleItems) => {
  visibleItems.forEach(({ id, type, logData }) => {
    if (!this.isSentRakeLogImpression(id, type)) {
      //로그가 전송된 상품인지 Set을 조회하여
      sendLog(logData) //로그를 전송하고
      this.addSentRakeLogImpression(id, listType) //로그가 전송된 상품은 Set에 추가
    }
  })
}
```

# 테스트는 어떻게 할 수 있을까?

스크롤 이벤트나 Intersection Observer는 DOM 환경에 강하게 의존하기 때문에 테스트가 쉽지 않았는데요. 테스트 케이스를 유닛 테스트, E2E 테스트로 구분해보면 다음과 같이 구분할 수 있었습니다.

**단위 테스트 흐름**

- 상품이 뷰포트에 보이면 로그를 전송한다.
- 상품이 뷰포트에서 사라지면 로그를 제거한다.
- 동일 상품에 대해 로그는 한 번만 전송된다.
- 스크롤이 멈춘 후 일정 시간 후 로그가 호출된다 (`debounce` 검증)

**E2E 테스트 흐름**

- 페이지 진입 후, 화면에 상품이 보이면 로그가 호출된다.
- 스크롤 후 멈췄을 때 로그가 정확히 전송된다.
- 같은 상품이 반복 노출되어도 중복 로그가 발생하지 않는다.

이 중 useViewportLog에 대한 단위 테스트를 적용해본다면, 스크롤을 멈췄을 때 로그가 전송되는지, debounce를 통해 스크롤 종료 후 로그가 1번 전송 되고 있는지 여부 등을 테스트해볼 수 있습니다.

```ts
it('스크롤 멈춘 후 500ms 뒤에 로그가 호출된다', () => {
  renderHook(() => useViewportLog(), { wrapper })

  act(() => {
    jest.advanceTimersByTime(1000) // 초기 렌더링 타이머 1초
  })

  mockHandleLog.mockClear()

  act(() => {
    window.dispatchEvent(new Event('scroll'))
    jest.advanceTimersByTime(500)
  })

  expect(mockHandleLog).toHaveBeenCalledWith('1', [])
})

it('동일한 스크롤 이벤트에 대해 디바운스가 적용된다', () => {
  renderHook(() => useViewportLog(), { wrapper })

  act(() => {
    jest.advanceTimersByTime(1000) // 초기 렌더링 타이머 1초
  })

  mockHandleLog.mockClear()

  act(() => {
    window.dispatchEvent(new Event('scroll'))
    window.dispatchEvent(new Event('scroll'))
    window.dispatchEvent(new Event('scroll'))
    jest.advanceTimersByTime(500)
  })

  expect(mockHandleLog).toHaveBeenCalledTimes(1) //스크롤 이벤트가 여러 번 발생해도 로그는 1번 전송한다
})
```

# 마치며

이번 로그 개선 작업은 단순한 기능 구현을 넘어서,
요구사항을 어떻게 해석하고 기술적으로 풀어내는지가 얼마나 중요한지를 절실히 느낀 경험이었습니다.

처음에는 요구사항 분석이 부족한 상태로 기능 구현에 급급하다보니,

- 로그가 중복 전송되거나
- 실제 뷰포트에 보이지 않는 상품까지 로그가 전송되는 문제

등의 문제를 겪으며 많은 시행착오를 겪어 예상보다 시간이 오래 걸리기도 했습니다.

이 과정을 통해, Web 이벤트 흐름을 정확히 이해하고,
Intersection Observer나 scroll 이벤트 같은 기술을 목적에 맞게 조합해
최적의 동작 시점을 찾아내는 능력이 매우 중요하다는 걸 체감했습니다.

요구사항 분석 → 최적화된 관찰 방식 설계 → 테스팅에 이르는 흐름을 직접 경험하면서,
기능의 완성도뿐 아니라 유지보수성과 안정성까지 고려하는 사고의 확장이 개발자로서 꼭 필요한 역량임을 다시 한 번 깨달았습니다.
