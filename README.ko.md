<p align="center">
  <img src="./docs/bear.jpg" />
</p>

[![Build Status](https://img.shields.io/github/actions/workflow/status/pmndrs/zustand/test.yml?branch=main&style=flat&colorA=000000&colorB=000000)](https://github.com/pmndrs/zustand/actions?query=workflow%3ATest)
[![Build Size](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fdeno.bundlejs.com%2F%3Fq%3Dzustand&query=%24.size.uncompressedSize&style=flat&label=bundle%20size&colorA=000000&colorB=000000)](https://bundlejs.com/?q=zustand)
[![Version](https://img.shields.io/npm/v/zustand?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/zustand)
[![Downloads](https://img.shields.io/npm/dt/zustand.svg?style=flat&colorA=000000&colorB=000000)](https://www.npmjs.com/package/zustand)
[![Discord Shield](https://img.shields.io/discord/740090768164651008?style=flat&colorA=000000&colorB=000000&label=discord&logo=discord&logoColor=ffffff)](https://discord.gg/poimandres)

<a href="https://dai-shi.github.io/zustand-banner-sponsorship/sponsors/" target="_blank" rel="noopener">
  <p align="center">
    <img src="https://dai-shi.github.io/zustand-banner-sponsorship/api/banner.png" />
  </p>
</a>

단순화된 flux 원칙을 사용하는 작고, 빠르며, 확장 가능한 최소한의(bearbones) 상태 관리 솔루션입니다. 훅(hook) 기반의 편안한 API를 제공하며, 보일러플레이트가 많거나 특정 방식을 강요하지 않습니다.

귀엽다고 무시하지 마세요. 꽤 날카로운 발톱을 가지고 있습니다. 악명 높은 [좀비 자식 문제(zombie child problem)](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children), [React 동시성(concurrency)](https://github.com/bvaughn/rfcs/blob/useMutableSource/text/0000-use-mutable-source.md), 그리고 혼합 렌더러 간의 [컨텍스트 손실(context loss)](https://github.com/facebook/react/issues/13332) 같은 흔한 함정들을 다루는 데 많은 시간을 쏟았습니다. React 생태계에서 이 모든 것을 제대로 처리하는 유일한 상태 관리자일지도 모릅니다.

라이브 [데모](https://zustand-demo.pmnd.rs/)를 사용해 보고 [문서](https://zustand.docs.pmnd.rs/)를 읽어볼 수 있습니다.

```bash
npm install zustand
```

:warning: 이 readme는 JavaScript 사용자를 위해 작성되었습니다. TypeScript 사용자라면 [TypeScript 사용법 섹션](#typescript-사용법)을 꼭 확인하세요.

## 먼저 스토어를 생성하세요

스토어는 곧 훅입니다! 원시값(primitive), 객체, 함수 등 무엇이든 담을 수 있습니다. 상태는 불변(immutable)하게 업데이트해야 하며, `set` 함수가 [상태를 병합](./docs/guides/immutable-state-and-merging.md)해 이를 돕습니다.

```jsx
import { create } from 'zustand'

const useBearStore = create((set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
}))
```

## 그런 다음 컴포넌트에 바인딩하면, 끝입니다!

어디에서든 훅을 사용하세요. 프로바이더(provider)는 필요 없습니다. 원하는 상태를 선택하면 그 상태가 변경될 때 컴포넌트가 리렌더링됩니다.

```jsx
function BearCounter() {
  const bears = useBearStore((state) => state.bears)
  return <h1>{bears} around here ...</h1>
}

function Controls() {
  const increasePopulation = useBearStore((state) => state.increasePopulation)
  return <button onClick={increasePopulation}>one up</button>
}
```

### 왜 redux 대신 zustand인가요?

- 단순하고 특정 방식을 강요하지 않습니다
- 훅을 상태 소비의 기본 수단으로 삼습니다
- 앱을 컨텍스트 프로바이더로 감싸지 않습니다
- [렌더링을 일으키지 않고 컴포넌트에 일시적으로(transiently) 정보를 전달할 수 있습니다](#일시적-업데이트-자주-발생하는-상태-변경-시)

### 왜 context 대신 zustand인가요?

- 보일러플레이트가 적습니다
- 변경이 있을 때만 컴포넌트를 렌더링합니다
- 중앙 집중식의 액션 기반 상태 관리

---

# 레시피

## 전체 상태 가져오기

가능합니다. 다만 이렇게 하면 모든 상태 변경마다 컴포넌트가 업데이트된다는 점을 명심하세요!

```jsx
const state = useBearStore()
```

## 여러 상태 슬라이스 선택하기

기본적으로 엄격한 동등성(strict-equality, old === new)으로 변경을 감지하며, 이는 원자적인(atomic) 상태 선택에 효율적입니다.

```jsx
const nuts = useBearStore((state) => state.nuts)
const honey = useBearStore((state) => state.honey)
```

redux의 mapStateToProps처럼 여러 상태를 선택해 하나의 객체로 구성하고 싶다면, [useShallow](./docs/guides/prevent-rerenders-with-use-shallow.md)를 사용해 셀렉터의 출력이 얕은 비교(shallow equal) 기준으로 변하지 않을 때 불필요한 리렌더링을 방지할 수 있습니다.

```jsx
import { create } from 'zustand'
import { useShallow } from 'zustand/react/shallow'

const useBearStore = create((set) => ({
  nuts: 0,
  honey: 0,
  treats: {},
  // ...
}))

// 객체로 선택, state.nuts 또는 state.honey가 변경되면 컴포넌트를 리렌더링합니다
const { nuts, honey } = useBearStore(
  useShallow((state) => ({ nuts: state.nuts, honey: state.honey })),
)

// 배열로 선택, state.nuts 또는 state.honey가 변경되면 컴포넌트를 리렌더링합니다
const [nuts, honey] = useBearStore(
  useShallow((state) => [state.nuts, state.honey]),
)

// 매핑하여 선택, state.treats의 순서, 개수, 키가 변경되면 컴포넌트를 리렌더링합니다
const treats = useBearStore(useShallow((state) => Object.keys(state.treats)))
```

리렌더링을 더 세밀하게 제어하려면 원하는 커스텀 동등성 함수를 제공할 수 있습니다 (이 예제는 [`createWithEqualityFn`](./docs/migrations/migrating-to-v5.md#using-custom-equality-functions-such-as-shallow) 사용이 필요합니다).

```jsx
const treats = useBearStore(
  (state) => state.treats,
  (oldTreats, newTreats) => compare(oldTreats, newTreats),
)
```

## 상태 덮어쓰기

`set` 함수는 두 번째 인자를 가지며 기본값은 `false`입니다. 이를 사용하면 상태를 병합하는 대신 상태 모델 전체를 교체합니다. 액션처럼 의존하고 있는 부분을 실수로 지우지 않도록 주의하세요.

```jsx
const useFishStore = create((set) => ({
  salmon: 1,
  tuna: 2,
  deleteEverything: () => set({}, true), // 액션을 포함해 스토어 전체를 비웁니다
  deleteTuna: () => set(({ tuna, ...rest }) => rest, true),
}))
```

## 비동기 액션

준비가 되면 `set`을 호출하기만 하면 됩니다. zustand는 액션이 비동기인지 아닌지 신경 쓰지 않습니다.

```jsx
const useFishStore = create((set) => ({
  fishies: {},
  fetch: async (pond) => {
    const response = await fetch(pond)
    set({ fishies: await response.json() })
  },
}))
```

## 액션 내에서 상태 읽기

`set`은 함수형 업데이트 `set(state => result)`를 허용하지만, 그 바깥에서도 `get`을 통해 상태에 접근할 수 있습니다.

```jsx
const useSoundStore = create((set, get) => ({
  sound: 'grunt',
  action: () => {
    const sound = get().sound
    ...
```

## 컴포넌트 외부에서 상태를 읽고 쓰며 변경에 반응하기

때로는 비반응형(non-reactive) 방식으로 상태에 접근하거나 스토어를 조작해야 할 때가 있습니다. 이런 경우를 위해, 생성된 훅의 프로토타입에 유틸리티 함수들이 부착되어 있습니다.

:warning: 이 기법은 [React 서버 컴포넌트](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)(보통 Next.js 13 이상)에서 상태를 추가하는 용도로는 권장되지 않습니다. 예기치 않은 버그와 사용자 개인정보 문제로 이어질 수 있습니다. 자세한 내용은 [#2200](https://github.com/pmndrs/zustand/discussions/2200)을 참고하세요.

```jsx
const useDogStore = create(() => ({ paw: true, snout: true, fur: true }))

// 비반응형으로 최신 상태 가져오기
const paw = useDogStore.getState().paw
// 모든 변경을 구독, 변경이 있을 때마다 동기적으로 실행됩니다
const unsub1 = useDogStore.subscribe(console.log)
// 상태 업데이트, 리스너를 트리거합니다
useDogStore.setState({ paw: false })
// 리스너 구독 해제
unsub1()

// 물론 훅을 평소처럼 사용할 수도 있습니다
function Component() {
  const paw = useDogStore((state) => state.paw)
  ...
```

### 셀렉터와 함께 subscribe 사용하기

셀렉터를 사용해 구독해야 한다면 `subscribeWithSelector` 미들웨어가 도움이 됩니다.

이 미들웨어를 사용하면 `subscribe`가 추가적인 시그니처를 받습니다:

```ts
subscribe(selector, callback, options?: { equalityFn, fireImmediately }): Unsubscribe
```

```js
import { subscribeWithSelector } from 'zustand/middleware'
const useDogStore = create(
  subscribeWithSelector(() => ({ paw: true, snout: true, fur: true })),
)

// 선택한 값의 변경을 구독, 이 경우 "paw"가 변경될 때 실행됩니다
const unsub2 = useDogStore.subscribe((state) => state.paw, console.log)
// subscribe는 이전 값도 함께 제공합니다
const unsub3 = useDogStore.subscribe(
  (state) => state.paw,
  (paw, previousPaw) => console.log(paw, previousPaw),
)
// subscribe는 선택적인 동등성 함수도 지원합니다
const unsub4 = useDogStore.subscribe(
  (state) => [state.paw, state.fur],
  console.log,
  { equalityFn: shallow },
)
// 구독과 동시에 즉시 실행
const unsub5 = useDogStore.subscribe((state) => state.paw, console.log, {
  fireImmediately: true,
})
```

## React 없이 zustand 사용하기

Zustand 코어는 React 의존성 없이 가져와 사용할 수 있습니다. 유일한 차이점은 create 함수가 훅이 아니라 API 유틸리티를 반환한다는 것입니다.

```jsx
import { createStore } from 'zustand/vanilla'

const store = createStore((set) => ...)
const { getState, setState, subscribe, getInitialState } = store

export default store
```

v4부터 제공되는 `useStore` 훅과 함께 바닐라 스토어를 사용할 수 있습니다.

```jsx
import { useStore } from 'zustand'
import { vanillaStore } from './vanillaStore'

const useBoundStore = (selector) => useStore(vanillaStore, selector)
```

:warning: `set` 또는 `get`을 수정하는 미들웨어는 `getState`와 `setState`에는 적용되지 않는다는 점에 유의하세요.

## 일시적 업데이트 (자주 발생하는 상태 변경 시)

subscribe 함수를 사용하면 컴포넌트가 상태의 일부에 바인딩하면서도 변경 시 리렌더링을 강제하지 않을 수 있습니다. 언마운트 시 자동으로 구독을 해제하도록 useEffect와 함께 사용하는 것이 가장 좋습니다. 뷰를 직접 변경(mutate)할 수 있는 경우 이는 [극적인](https://codesandbox.io/s/peaceful-johnson-txtws) 성능 향상을 가져올 수 있습니다.

```jsx
const useScratchStore = create((set) => ({ scratches: 0, ... }))

const Component = () => {
  // 초기 상태 가져오기
  const scratchRef = useRef(useScratchStore.getState().scratches)
  // 마운트 시 스토어에 연결하고, 언마운트 시 연결을 해제하며, 상태 변경을 ref에 담아둡니다
  useEffect(() => useScratchStore.subscribe(
    state => (scratchRef.current = state.scratches)
  ), [])
  ...
```

## 리듀서와 중첩 상태 변경에 지치셨나요? Immer를 사용하세요!

중첩된 구조를 리듀싱하는 일은 번거롭습니다. [immer](https://github.com/mweststrate/immer)를 사용해 보셨나요?

```jsx
import { produce } from 'immer'

const useLushStore = create((set) => ({
  lush: { forest: { contains: { a: 'bear' } } },
  clearForest: () =>
    set(
      produce((state) => {
        state.lush.forest.contains = null
      }),
    ),
}))

const clearForest = useLushStore((state) => state.clearForest)
clearForest()
```

[이 외에도 다른 해결책들이 있습니다.](./docs/learn/guides/updating-state.md#with-immer)

## Persist 미들웨어

어떤 종류의 스토리지를 사용하든 스토어의 데이터를 영속화(persist)할 수 있습니다.

```jsx
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

const useFishStore = create(
  persist(
    (set, get) => ({
      fishes: 0,
      addAFish: () => set({ fishes: get().fishes + 1 }),
    }),
    {
      name: 'food-storage', // 스토리지에 저장될 항목의 이름 (고유해야 합니다)
      storage: createJSONStorage(() => sessionStorage), // (선택) 기본적으로 'localStorage'가 사용됩니다
    },
  ),
)
```

[이 미들웨어에 대한 전체 문서를 확인하세요.](./docs/reference/integrations/persisting-store-data.md)

## Immer 미들웨어

Immer는 미들웨어로도 제공됩니다.

```jsx
import { create } from 'zustand'
import { immer } from 'zustand/middleware/immer'

const useBeeStore = create(
  immer((set) => ({
    bees: 0,
    addBees: (by) =>
      set((state) => {
        state.bees += by
      }),
  })),
)
```

## redux 같은 리듀서와 액션 타입 없이는 못 살겠나요?

```jsx
const types = { increase: 'INCREASE', decrease: 'DECREASE' }

const reducer = (state, { type, by = 1 }) => {
  switch (type) {
    case types.increase:
      return { grumpiness: state.grumpiness + by }
    case types.decrease:
      return { grumpiness: state.grumpiness - by }
  }
}

const useGrumpyStore = create((set) => ({
  grumpiness: 0,
  dispatch: (args) => set((state) => reducer(state, args)),
}))

const dispatch = useGrumpyStore((state) => state.dispatch)
dispatch({ type: types.increase, by: 2 })
```

또는 그냥 redux 미들웨어를 사용하세요. 메인 리듀서를 연결하고, 초기 상태를 설정하며, 상태 자체와 바닐라 API에 dispatch 함수를 추가해 줍니다.

```jsx
import { redux } from 'zustand/middleware'

const useGrumpyStore = create(redux(reducer, initialState))
```

## Redux devtools

devtools 미들웨어를 사용하려면 [Redux DevTools 크롬 확장 프로그램](https://chromewebstore.google.com/detail/redux-devtools/lmhkpmbekcpmknklioeibfkpmmfibljd)을 설치하세요.

```jsx
import { devtools } from 'zustand/middleware'

// 일반 액션 스토어와 함께 사용하면 액션을 "setState"로 로깅합니다
const usePlainStore = create(devtools((set) => ...))
// redux 스토어와 함께 사용하면 전체 액션 타입을 로깅합니다
const useReduxStore = create(devtools(redux(reducer, initialState)))
```

여러 스토어를 하나의 redux devtools 연결로 사용하기

```jsx
import { devtools } from 'zustand/middleware'

// 일반 액션 스토어와 함께 사용하면 액션을 "setState"로 로깅합니다
const usePlainStore1 = create(devtools((set) => ..., { name, store: storeName1 }))
const usePlainStore2 = create(devtools((set) => ..., { name, store: storeName2 }))
// redux 스토어와 함께 사용하면 전체 액션 타입을 로깅합니다
const useReduxStore1 = create(devtools(redux(reducer, initialState)), { name, store: storeName3 })
const useReduxStore2 = create(devtools(redux(reducer, initialState)), { name, store: storeName4 })
```

서로 다른 연결 이름을 지정하면 redux devtools에서 스토어들이 분리됩니다. 이는 또한 서로 다른 스토어들을 별도의 redux devtools 연결로 그룹화하는 데 도움이 됩니다.

devtools는 첫 번째 인자로 스토어 함수를 받으며, 선택적으로 두 번째 인자를 통해 스토어 이름을 지정하거나 [serialize](https://github.com/zalmoxisus/redux-devtools-extension/blob/master/docs/API/Arguments.md#serialize) 옵션을 구성할 수 있습니다.

스토어 이름 지정: `devtools(..., {name: "MyStore"})`는 devtools에 "MyStore"라는 이름의 별도 인스턴스를 생성합니다.

Serialize 옵션: `devtools(..., { serialize: { options: true } })`.

#### 액션 로깅

devtools는 일반적인 _결합된 리듀서(combined reducers)_ redux 스토어와 달리 각각 분리된 스토어의 액션만 로깅합니다. 스토어를 결합하는 방법은 https://github.com/pmndrs/zustand/issues/163 을 참고하세요.

세 번째 매개변수를 전달하여 각 `set` 함수마다 특정 액션 타입을 로깅할 수 있습니다:

```jsx
const useBearStore = create(devtools((set) => ({
  ...
  eatFish: () => set(
    (prev) => ({ fishes: prev.fishes > 1 ? prev.fishes - 1 : 0 }),
    undefined,
    'bear/eatFish'
  ),
  ...
```

액션의 타입을 페이로드와 함께 로깅할 수도 있습니다:

```jsx
  ...
  addFishes: (count) => set(
    (prev) => ({ fishes: prev.fishes + count }),
    undefined,
    { type: 'bear/addFishes', count, }
  ),
  ...
```

액션 타입을 제공하지 않으면 기본값으로 "anonymous"가 사용됩니다. `anonymousActionType` 매개변수를 제공하여 이 기본값을 커스터마이징할 수 있습니다:

```jsx
devtools(..., { anonymousActionType: 'unknown', ... })
```

(예를 들어 프로덕션 환경에서) devtools를 비활성화하고 싶다면, `enabled` 매개변수를 제공하여 이 설정을 변경할 수 있습니다:

```jsx
devtools(..., { enabled: false, ... })
```

## React 컨텍스트

`create`로 생성한 스토어는 컨텍스트 프로바이더가 필요 없습니다. 하지만 의존성 주입(dependency injection)을 위해 컨텍스트를 사용하거나, 컴포넌트의 props로 스토어를 초기화하고 싶은 경우가 있을 수 있습니다. 일반 스토어는 훅이기 때문에, 이를 일반적인 컨텍스트 값으로 전달하면 훅의 규칙(rules of hooks)을 위반할 수 있습니다.

v4부터 권장되는 방법은 바닐라 스토어를 사용하는 것입니다.

```jsx
import { createContext, useContext } from 'react'
import { createStore, useStore } from 'zustand'

const store = createStore(...) // 훅이 없는 바닐라 스토어

const StoreContext = createContext()

const App = () => (
  <StoreContext.Provider value={store}>
    ...
  </StoreContext.Provider>
)

const Component = () => {
  const store = useContext(StoreContext)
  const slice = useStore(store, selector)
  ...
```

## TypeScript 사용법

기본적인 TypeScript 사용법은 `create(...)` 대신 `create<State>()(...)`로 작성하는 것 외에는 특별한 것이 필요하지 않습니다...

```ts
import { create } from 'zustand'
import { devtools, persist } from 'zustand/middleware'
import type {} from '@redux-devtools/extension' // devtools 타이핑에 필요합니다

interface BearState {
  bears: number
  increase: (by: number) => void
}

const useBearStore = create<BearState>()(
  devtools(
    persist(
      (set) => ({
        bears: 0,
        increase: (by) => set((state) => ({ bears: state.bears + by })),
      }),
      {
        name: 'bear-storage',
      },
    ),
  ),
)
```

더 자세한 TypeScript 가이드는 [여기](docs/learn/guides/beginner-typescript.md)와 [저기](docs/learn/guides/advanced-typescript.md)에 있습니다.

## 모범 사례

- 더 나은 유지보수를 위해 코드를 어떻게 구성해야 할지 궁금할 수 있습니다: [스토어를 여러 슬라이스로 분리하기](./docs/learn/guides/slices-pattern.md).
- 이 비강제적인(unopinionated) 라이브러리의 권장 사용법: [Flux에서 영감을 받은 실천법](./docs/learn/guides/flux-inspired-practice.md).
- [React 18 이전 버전에서 React 이벤트 핸들러 외부에서 액션 호출하기](./docs/learn/guides/event-handler-in-pre-react-18.md).
- [테스트](./docs/learn/guides/testing.md)
- 더 많은 내용은 [docs 폴더](./docs/index.md)를 살펴보세요

## 서드파티 라이브러리

일부 사용자는 Zustand의 기능을 확장하고 싶을 수 있으며, 이는 커뮤니티에서 만든 서드파티 라이브러리를 사용해 가능합니다. Zustand와 함께 사용할 수 있는 서드파티 라이브러리에 대한 정보는 [문서](./docs/reference/integrations/third-party-libraries.md)를 참고하세요.

## 다른 라이브러리와의 비교

- [zustand와 다른 React 상태 관리 라이브러리의 차이점](https://zustand.docs.pmnd.rs/learn/getting-started/comparison)
