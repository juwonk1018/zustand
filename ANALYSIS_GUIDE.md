# 🐻 Zustand 오픈소스 분석 가이드

> 처음 오픈소스 프로젝트를 분석하는 사람을 위한 zustand 코드베이스 분석 로드맵.
> 소스 전체가 약 1,400줄, 핵심 엔진은 단 100줄([src/vanilla.ts](src/vanilla.ts))이라 "전체를 다 읽고 이해하는 것"이 현실적으로 가능한, 입문용으로 최고의 프로젝트입니다.

---

## 0단계 — 분석의 일반 원칙 (모든 OSS에 적용)

코드부터 파고들기 전에 **"바깥 → 안" + "사용법 → 구현"** 순서를 기억하세요.

| 순서 | 무엇을 | 어디서 (zustand) |
|------|--------|------------------|
| ① 정체성 | 이게 뭘 하는 라이브러리인가 | [README.md](README.md), `package.json` description |
| ② 사용법 | 사용자는 이걸 어떻게 쓰나 | [README.md](README.md), [docs/getting-started/](docs/getting-started/) |
| ③ 진입점 | 코드의 시작은 어디인가 | [src/index.ts](src/index.ts), `package.json` exports |
| ④ 핵심 | 가장 중요한 한 파일은 | [src/vanilla.ts](src/vanilla.ts) |
| ⑤ 확장 | 핵심 위에 무엇이 얹히나 | [src/react.ts](src/react.ts), [src/middleware/](src/middleware/) |
| ⑥ 검증 | "무엇을 보장하는가" | [tests/](tests/) |

> 💡 **초보자가 가장 많이 하는 실수:** 첫 파일부터 TypeScript 제네릭(`Mutate<S, Ms>` 같은 것)을 이해하려 듭니다.
> **타입은 두 번째 읽기로 미루세요.** 첫 읽기는 "값이 어떻게 흐르는가(런타임 동작)"에만 집중합니다.

---

## 아키텍처 한눈에 보기

zustand는 **3개의 동심원** 구조입니다. 안에서 밖으로 읽으면 자연스럽게 이해됩니다.

```
┌─────────────────────────────────────────────┐
│  ③ 미들웨어 (middleware/)                       │
│   persist, devtools, immer, redux...          │
│   "set/get/api를 가로채서 기능 추가"            │
│  ┌───────────────────────────────────────┐   │
│  │  ② React 바인딩 (react.ts)              │   │
│  │   useStore + useSyncExternalStore       │   │
│  │   "vanilla를 React 렌더링에 연결"        │   │
│  │  ┌─────────────────────────────────┐   │   │
│  │  │  ① 코어 (vanilla.ts) ★심장부★     │   │   │
│  │  │   createStore                    │   │   │
│  │  │   getState/setState/subscribe    │   │   │
│  │  │   "순수 JS pub-sub, 100줄"        │   │   │
│  │  └─────────────────────────────────┘   │   │
│  └───────────────────────────────────────┘   │
│         + shallow (얕은 비교 유틸)              │
└─────────────────────────────────────────────┘
```

**핵심 통찰:** zustand의 본질은 React 라이브러리가 아니라 **"프레임워크 독립적인 pub-sub 상태 저장소"**이고,
React는 그 위에 얇게 얹힌 어댑터입니다. 이걸 깨닫는 순간 코드 전체가 이해됩니다.

---

## 단계별 읽기 로드맵 (난이도순)

### 1단계 — 코어 엔진 `vanilla.ts` ★가장 중요★ (난이도: 입문)

📄 [src/vanilla.ts](src/vanilla.ts) (100줄, 전체 정독)

여기가 zustand의 전부입니다. 이 100줄만 이해하면 절반은 끝납니다.

```typescript
const createStoreImpl = (createState) => {
  let state                                    // 현재 상태
  const listeners = new Set()                  // 구독자 목록 (Set!)

  const setState = (partial, replace) => {
    const nextState = typeof partial === 'function' ? partial(state) : partial
    if (!Object.is(nextState, state)) {        // 변경됐을 때만
      const previousState = state
      state = (replace ?? typeof nextState !== 'object' || nextState === null)
        ? nextState                            // 전체 치환
        : Object.assign({}, state, nextState)  // 얕은 병합 ★
      listeners.forEach((l) => l(state, previousState))  // 전파 ★
    }
  }
  const getState = () => state
  const subscribe = (listener) => {
    listeners.add(listener)
    return () => listeners.delete(listener)    // 구독 해제 함수 반환
  }
  // ...
}
```

**읽기 순서:**
1. [vanilla.ts:9-14](src/vanilla.ts#L9-L14) — `StoreApi` 인터페이스. 스토어가 제공하는 4개 메서드(`setState`, `getState`, `getInitialState`, `subscribe`)가 전부입니다. **이것이 "계약"**.
2. [vanilla.ts:66-81](src/vanilla.ts#L66-L81) — `setState`. **모든 상태 관리의 심장.** 함수형 업데이트 처리 → `Object.is`로 변경 감지 → 얕은 병합 vs 치환 → 리스너에게 알림.
3. [vanilla.ts:88-92](src/vanilla.ts#L88-L92) — `subscribe`. `Set`에 추가하고 삭제 함수를 반환하는 전형적인 pub-sub.
4. [vanilla.ts:99-100](src/vanilla.ts#L99-L100) — `createStore`의 커링 패턴 (`createState ? ... : ...`). 처음엔 "왜 이렇게?" 싶지만, TypeScript 타입 추론의 한계를 우회하는 트릭입니다. 지금은 넘어가도 됨.

**꼭 짚고 넘어갈 포인트:**
- **얕은 병합(shallow merge)의 한계** — `Object.assign({}, state, nextState)`는 1단계만 병합합니다.
  `{a: {b: 1}}` 상태에서 `setState({a: {c: 2}})`하면 `{a: {b:1, c:2}}`가 아니라 `{a: {c: 2}}`가 됩니다.
  → 이 한계가 나중에 **immer 미들웨어가 존재하는 이유**가 됩니다.
- **`Object.is` 변경 감지** — 같은 값을 다시 set하면 리스너가 호출되지 않습니다. 불필요한 리렌더를 막는 첫 번째 방어선.

---

### 2단계 — React 바인딩 `react.ts` (난이도: 심화 — 천천히)

📄 [src/react.ts](src/react.ts) (64줄)

vanilla 스토어를 React 렌더링에 연결하는 어댑터입니다. 핵심은 **단 한 줄**:

```typescript
const slice = React.useSyncExternalStore(
  api.subscribe,                                  // ① 구독: 상태 바뀌면 React에 알림
  React.useCallback(() => selector(api.getState()), [api, selector]),  // ② 스냅샷
  React.useCallback(() => selector(api.getInitialState()), ...),       // ③ SSR용
)
```

**읽기 순서:**
1. [react.ts:26-37](src/react.ts#L26-L37) — `useStore`. React 18의 공식 API `useSyncExternalStore`를 씁니다. vanilla의 `subscribe`/`getState`를 그대로 넘길 뿐입니다.
2. **selector 패턴 이해** — `useStore(store, s => s.count)`처럼 selector를 주면, `count`가 바뀔 때만 리렌더됩니다. zustand 성능 최적화의 핵심.
3. [react.ts:53-61](src/react.ts#L53-L61) — `create`. vanilla 스토어를 만들고, 그것을 **훅으로도 쓰고 API로도 쓸 수 있는 하이브리드 객체**로 포장합니다 ([react.ts:58](src/react.ts#L58)의 `Object.assign(useBoundStore, api)`가 핵심). 그래서 `useStore()`도 되고 `useStore.getState()`도 됩니다.

> ⚠️ **왜 `useSyncExternalStore`인가?** React 18 동시성 모드에서 "tearing"(같은 상태인데 컴포넌트마다 다른 값을 보는 현상)을 막기 위해서입니다. 첫 분석에서는 **"외부 상태 동기화 표준 API를 쓴다"** 정도로만 이해하고 넘어가세요.

비교 대상: [src/traditional.ts](src/traditional.ts) — `equalityFn`(커스텀 비교 함수)을 지원하는 버전. `react.ts`와 나란히 놓고 **"무엇이 더 추가됐나"**만 비교하면 됩니다.

---

### 3단계 — 얕은 비교 `shallow` (난이도: 중급)

📄 [src/vanilla/shallow.ts](src/vanilla/shallow.ts) (74줄)

selector가 매번 새 객체(`{ a, b }`)를 반환할 때 발생하는 "리렌더 폭발"을 막는 유틸입니다.

- [shallow.ts:48-74](src/vanilla/shallow.ts#L48-L74) — `Object.is` 빠른 경로 → 프로토타입 체크 → Map/Set/배열 분기 → 일반 객체는 `Object.entries`로 **1단계만** 비교.
- 📄 [src/react/shallow.ts](src/react/shallow.ts) — `useShallow`. `useRef`로 이전 결과를 캐싱하고, 내용이 같으면 **이전 참조를 재사용**해서 리렌더를 막습니다.

> ⚠️ **함정:** `shallow`는 깊은 비교가 **아닙니다.** `{user: {name: 'A'}}`에서 `name`만 바뀌면 감지 못 합니다.
> 또 `Date`/`RegExp` 같은 내장 객체는 제대로 비교 안 됩니다 ([tests/vanilla/shallow.test.tsx](tests/vanilla/shallow.test.tsx)에 엣지 케이스가 명세돼 있음).

---

### 4단계 — 미들웨어 시스템 (난이도: 중급 → 심화)

📁 [src/middleware/](src/middleware/)

모든 미들웨어는 **동일한 패턴** `(StateCreator) => StateCreator`입니다. `set`/`get`/`api`를 가로채(wrap) 기능을 추가합니다. **반드시 쉬운 것부터** 읽으세요:

| 순서 | 파일 | 줄 수 | 배우는 것 |
|------|------|-------|-----------|
| 1 | [combine.ts](src/middleware/combine.ts) | 15 | 가장 단순. 미들웨어 패턴의 뼈대 (초기 상태 병합만) |
| 2 | [redux.ts](src/middleware/redux.ts) | 50 | `api`에 `dispatch` 추가하기 |
| 3 | [subscribeWithSelector.ts](src/middleware/subscribeWithSelector.ts) | 73 | `api.subscribe`를 교체해 selector 구독 추가 |
| 4 | [immer.ts](src/middleware/immer.ts) | 87 | `set`을 `produce()`로 감싸기 (← 1단계의 얕은 병합 한계를 해결!) |
| 5 | [devtools.ts](src/middleware/devtools.ts) | 431 | `set`을 가로채 Redux DevTools로 액션 전송 (양방향) |
| 6 | [persist.ts](src/middleware/persist.ts) | 390 | 스토리지 저장/복원(rehydrate), 가장 복잡 |

**가장 좋은 학습법:** [combine.ts](src/middleware/combine.ts) 15줄을 먼저 보고 "아, 미들웨어는 그냥 `StateCreator`를 받아서 감싼 `StateCreator`를 돌려주는 함수구나"를 체득한 뒤, [redux.ts](src/middleware/redux.ts)에서 "set을 가로채 새 기능을 붙이는" 핵심 동작을 봅니다. devtools/persist는 같은 패턴의 복잡한 응용일 뿐입니다.

> ⚠️ **미들웨어 순서 주의:** `immer(devtools(...))`와 `devtools(immer(...))`는 타입과 동작이 다릅니다. 안쪽 미들웨어부터 적용됩니다.

---

### 5단계 — 테스트로 "명세" 확인하기 (난이도: 중급)

📁 [tests/](tests/)

**테스트는 곧 기능 명세서입니다.** "이 라이브러리가 무엇을 보장하는가"를 가장 정확히 알려줍니다.

- [tests/vanilla/basic.test.ts](tests/vanilla/basic.test.ts) — 순수 스토어 동작. **여기부터 읽으세요.**
- [tests/basic.test.tsx](tests/basic.test.tsx) — React 바인딩, selector, 리렌더 최적화 검증. 특히 "한 컴포넌트는 `count` 구독, 다른 컴포넌트는 `name` 구독 → `count`만 바꾸면 후자는 리렌더 안 됨"을 증명하는 부분이 압권입니다.
- [tests/types.test.tsx](tests/types.test.tsx) — `@ts-expect-error`로 "이런 코드는 타입 에러가 나야 한다"를 검증. **타입 안전성도 테스트 대상**이라는 걸 배웁니다. (숙련 후 읽기)

---

## ✋ 직접 해보는 실습 (이게 진짜 분석입니다)

읽기만 하면 금방 잊습니다. **코드를 직접 돌리고, 수정하고, 깨뜨려 보세요.**

**실습 1 — 코어를 React 없이 돌려보기** (vanilla의 본질 체감)
```js
// node REPL 또는 작은 스크립트
import { createStore } from './src/vanilla.ts'   // 또는 빌드 후 zustand/vanilla
const store = createStore((set) => ({
  count: 0,
  inc: () => set((s) => ({ count: s.count + 1 })),
}))
const unsub = store.subscribe((s, prev) => console.log(prev.count, '→', s.count))
store.getState().inc()       // 0 → 1
store.setState({ count: 1 }) // 같은 값! → 콜백 호출 안 됨 (Object.is 방어선 확인)
unsub()
```

**실습 2 — 일부러 깨뜨려 보기** (코드의 의도를 역으로 이해)

[vanilla.ts:73](src/vanilla.ts#L73)의 `if (!Object.is(nextState, state))` 조건을 지우고 항상 리스너를 호출하게 바꾼 뒤 테스트를 돌려보세요:
```powershell
pnpm run test:spec
```
→ 리렌더/구독 관련 테스트가 **깨집니다.** "이 한 줄이 무엇을 보장하는가"를 몸으로 알게 됩니다. (확인 후 `git checkout`으로 복원)

**실습 3 — selector 유무 비교** — 컴포넌트 2개를 만들어 하나는 `useStore()`, 하나는 `useStore(s => s.count)`로 구독하고, `name`만 바꿔보며 리렌더 횟수를 비교.

---

## 분석 결과를 "검증"하는 법

개발 환경은 이미 다 갖춰져 있습니다 ([package.json](package.json) scripts 참고):

```powershell
pnpm install          # 의존성 설치 (pnpm 사용 — package.json에 고정됨)
pnpm run test:spec    # 단위 테스트 (vitest, jsdom 환경)
pnpm run test:types   # 타입 검증 (tsc --noEmit)
pnpm run test:lint    # ESLint
pnpm run build        # 멀티 엔트리 빌드 → dist/ 생성
```

**검증 체크리스트** — 아래를 한 문장으로 설명할 수 있으면 분석 성공입니다:
- [ ] `setState`가 왜 **얕은** 병합을 하는가? 깊은 병합이 필요하면?
- [ ] `set(state, true)`의 `replace` 플래그는 언제 쓰는가?
- [ ] selector를 주면 왜 리렌더가 줄어드는가?
- [ ] `create`(react)와 `createStore`(vanilla)의 차이는?
- [ ] 미들웨어의 공통 시그니처는? immer는 무엇을 가로채는가?

---

## 추천 학습 동선 요약

```
README + docs/getting-started  (사용자 관점, 30분)
   ↓
src/vanilla.ts 정독 + 실습 1   (코어, 45분) ★여기가 핵심★
   ↓
src/react.ts (selector, useSyncExternalStore, 30분)
   ↓
src/vanilla/shallow.ts (얕은 비교, 20분)
   ↓
middleware: combine → redux → immer (패턴 체득, 1시간)
   ↓
tests/ 읽기 + 실습 2 (깨뜨려보기, 검증, 45분)
   ↓
package.json / rollup.config.mjs (빌드 구조, 30분)
```

---

## 빌드 / 프로젝트 구조 참고

- **멀티 엔트리 빌드:** 하나의 소스에서 `zustand`, `zustand/vanilla`, `zustand/react`, `zustand/middleware`, `zustand/shallow` 등 여러 진입점을 빌드합니다. `package.json`의 `build:*` 스크립트와 [rollup.config.mjs](rollup.config.mjs)의 `config-*` 플래그가 대응됩니다.
- **peerDependencies:** `react`, `immer`, `use-sync-external-store`가 **optional**입니다. 즉 `zustand/vanilla`는 React 없이도 동작하고, immer 미들웨어를 쓸 때만 immer가 필요합니다. 이것이 "프레임워크 독립적" 설계의 증거입니다.
- **개발 경로 매핑:** [tsconfig.json](tsconfig.json)이 `zustand/*` import를 `./src/*.ts`로 매핑해 IDE 지원을 제공합니다.

---

## 기여(Contribute)까지 생각한다면

[CONTRIBUTING.md](CONTRIBUTING.md) 참고:
- **conventional commits** 요구: `feat:`, `fix(react):`, `docs:`, `chore:` 등
- 코어 수정 시 **실패하는 테스트를 먼저 작성**하도록 권장
- 첫 기여로는 분석 과정에서 발견한 작은 문서 수정/테스트 추가 PR이 좋습니다.

---

_이 가이드는 zustand 5.0.8 기준으로 작성되었습니다._
