이 글은 react 19를 기반으로 쓰여졌습니다.

**목차**

1. react hook은 어디서 구현되고 있을까?
2. useState는 진짜 useReducer로 이루어져 있을까?
3. 훅 객체 생성하고 linked list로 연결하기
4. circular queue를 이용해 상태 update하기
5. 느낀점

책을 읽던 중, useState는 useReducer로 구현되었다고 하기에 진짜인지 궁금해서 소스코드를 찾아보기 시작했다.

# react hook은 어디서 구현되고 있을까?

[react/packages/react/src/ReactHooks.js at main · facebook/react · GitHub](https://github.com/facebook/react/blob/main/packages/react/src/ReactHooks.js)

## ReactHooks 파일 뜯어보기

react hook이니 ReactHooks.js 파일에 있을 것이라 예상했고, 다음과 같은 코드를 찾을 수 있었다.

```js
import type { Dispatcher } from "react-reconciler/src/ReactInternalTypes";
import ReactSharedInternals from "shared/ReactSharedInternals";

type Dispatch<A> = (A) => void;

function resolveDispatcher() {
  const dispatcher = ReactSharedInternals.H;
  return ((dispatcher: any): Dispatcher);
}

export function useState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

export function useReducer<S, I, A>(
  reducer: (S, A) => S,
  initialArg: I,
  init?: (I) => S
): [S, Dispatch<A>] {
  const dispatcher = resolveDispatcher();
  return dispatcher.useReducer(reducer, initialArg, init);
}

export function useRef<T>(initialValue: T): { current: T } {
  const dispatcher = resolveDispatcher();
  return dispatcher.useRef(initialValue);
}
```

이 파일에서 직접 구현되지는 않고, 내부 dispatcher에게 위임하는 wrapper일 뿐인 것 같다. 내가 기대하는 로직은 ReactSharedInternals에서서 실행되는 것 같아 찾아보았다.

/shared/ReactSharedInternals 파일을 찾아봤다.

```js
import * as React from "react";

const ReactSharedInternals =
  React.__CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE;

export default ReactSharedInternals;
```

아무것도 없다.

CLIENT_INTERNALS~가 뭐지? 궁금해하던 중, react/src/reactClient에서 아래 코드를 발견할 수 있었다.
ReactSharedInternalsClient에서 import한 ReactSharedInternals를 shared/ReactSharedInternals에서 전역으로 가져다쓴다는 것을 알 수 있었다.
[react/packages/react/src/ReactClient.js at 6b7e207cabe4c1bc9390d862dd9228e94e9edf4b · facebook/react · GitHub](https://github.com/facebook/react/blob/6b7e207cabe4c1bc9390d862dd9228e94e9edf4b/packages/react/src/ReactClient.js#L110)

```js
import {
  ...
  useReducer,
  useRef,
  useState,
} from './ReactHooks';
import ReactSharedInternals from './ReactSharedInternalsClient';

export {
  ...
  ReactSharedInternals as __CLIENT_INTERNALS_DO_NOT_USE_OR_WARN_USERS_THEY_CANNOT_UPGRADE,
  ...
};
```

ReactSharedInternalsClient.js파일을 찾아보았다.

[react/packages/react/src/ReactSharedInternalsClient.js at main · facebook/react · GitHub](https://github.com/facebook/react/blob/main/packages/react/src/ReactSharedInternalsClient.js)

```js
export type SharedStateClient = {
  H: null | Dispatcher, // ReactCurrentDispatcher for Hooks
  A: null | AsyncDispatcher, // ReactCurrentCache for Cache
  T: null | Transition, // ReactCurrentBatchConfig for Transitions
  S: null | onStartTransitionFinish,
  G: null | onStartGestureTransitionFinish,
```

ReactSharedInternalsClient.js에서 ReactCurrentDispatcher for Hooks이 존재한다.

외부에서 ReactSharedInternals에 훅 객체를 주입해준다는 것을 알 수 있었다.

## renderWithHooks()

그럼 실제 훅은 어디서 렌더링될까?

아주 직관적인 이름의 함수 renderWithHooks()를 찾을 수 있었다.

[react/packages/react-reconciler/src/ReactFiberHooks.js at main · facebook/react · GitHub](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)

```js
import ReactSharedInternals from 'shared/ReactSharedInternals';

export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
  renderLanes = nextRenderLanes;
  currentlyRenderingFiber = workInProgress;

  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  workInProgress.lanes = NoLanes;

  if (__DEV__) {
    ...//개발모드에서만 실행되는 코드이므로 생략
  } else {
    ReactSharedInternals.H =
      current === null || current.memoizedState === null
        ? HooksDispatcherOnMount
        : HooksDispatcherOnUpdate;
  }

  finishRenderingHooks(current, workInProgress, Component);

  return children;
}
```

이 함수에서 주목할 부분이 있다.

```js
ReactSharedInternals.H =
  current === null || current.memoizedState === null
    ? HooksDispatcherOnMount
    : HooksDispatcherOnUpdate;
```

renderWithHooks()에서 ReactSharedInternals에 HooksDispatcherOnMount 혹은 HooksDispatcherOnUpdate를 주입한다는 것을 알 수 있었다.

## 리액트의 렌더링 방식 - Reconciliation

실제 hook 코드를 뜯어보기 전에, 리액트의 렌더링 방식에 대해 잠깐 살펴보자.

React 는 UI 안에 있는 컴포넌트 구조로 [렌더 트리](https://ko.react.dev/learn/understanding-your-ui-as-a-tree#the-render-tree)를 만들고, Virtual DOM을 이용하여 변경된 부분만 반영한다.

[재조정 (Reconciliation) – React](https://ko.legacy.reactjs.org/docs/reconciliation.html)
지금은 업데이트되지 않는 번역 사이트지만 공식 문서에 reconcilation 관련 내용이 없어서 이거라도 들고와봤다. (어떤 프레임워크나 라이브러리가 어떤 문제를 해결하기 위해 어떻게 구현을 했는지 살펴보는 것을 좋아하는데, 최근 업데이트 된 리액트 공식 문서는 이런 내용 없이 사용 방법 위주로 기재되어 있어 늘 아쉬움이 있다.)

1. 엘리먼트의 type(div, span 등)이 변경될 때
2. key props가 변경될 때
   React 엘리먼트 트리가 변경됨을 감지하고 변경 사항을 적용한다.
   이 때 react reconciler이 감지할 수 있는 내부 객체를 fiber node라고 한다.

fiber에 대해 잠시 알아봤으니 ReactFiberHooks.js를 들여다보도록 하자.

# useState는 진짜 useReducer로 이루어져 있을까?

### ReactFiberHooks.js

ReactFiberHooks.js파일에서 구현된 훅을 확인할 수 있었다.

[react/packages/react-reconciler/src/ReactFiberHooks.js at main · facebook/react · GitHub](https://github.com/facebook/react/blob/main/packages/react-reconciler/src/ReactFiberHooks.js)

HooksDispatcherOnMount, HooksDispathcerOnUpdate 등에서 hook을 관리하고 있다.
mount 될 때 useReducer은 mountReducer, useState는 mountState가 실행되며, update될 때는 updateReducer과 updateState가 실행된다.

```js
const HooksDispatcherOnMount: Dispatcher = {
  ...
  useReducer: mountReducer,
  useRef: mountRef,
  useState: mountState,
  ...
};

const HooksDispatcherOnUpdate: Dispatcher = {
  ...
  useReducer: updateReducer,
  useRef: updateRef,
  useState: updateState,
  ...
};

const HooksDispatcherOnRerender: Dispatcher = {
  ...
  useReducer: rerenderReducer,
  useRef: updateRef,
  useState: rerenderState,
  ...
};
```

실제로 updateState에서 updateReducer을 사용하고 있었다.

```js
function mountState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountStateImpl(initialState);
  const queue = hook.queue;
  const dispatch: Dispatch<BasicStateAction<S>> = (dispatchSetState.bind(
    null,
    currentlyRenderingFiber,
    queue
  ): any);
  queue.dispatch = dispatch;
  return [hook.memoizedState, dispatch];
}

function updateState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  return updateReducer(basicStateReducer, initialState);
}

function rerenderState<S>(
  initialState: (() => S) | S
): [S, Dispatch<BasicStateAction<S>>] {
  return rerenderReducer(basicStateReducer, initialState);
}
```

mountState 내부에서 hook을 선언하기 위해 mountStateImpl()이 사용되므로 함께 살펴보았다다.

```js
function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>] {
  const hook = mountStateImpl(initialState);
  const queue = hook.queue;
  ...

function mountStateImpl<S>(initialState: (() => S) | S): Hook {
  const hook = mountWorkInProgressHook();
  ...
  hook.memoizedState = hook.baseState = initialState;
  const queue: UpdateQueue<S, BasicStateAction<S>> = {
    pending: null,
    lanes: NoLanes,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: (initialState: any),
  };
  hook.queue = queue;
  return hook;
}
```

mountStateImpl에서, const hook = mountWorkInProgressHook(); 부분이 눈에 띈다. hook을 mountWorkInProgressHook()을 이용해 생성하는 것을 알 수 있었다.

mountWorkInProgressHook()을 자세히 들여다보았다.

# 훅 객체 생성하고 linked list로 연결하기

### mountWorkInProgressHook()

```js
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null, // useState 등에서 실제 저장되는 값

    baseState: null, // 이전 상태 값 (업데이트 처리용)
    baseQueue: null, // 업데이트 큐 (pending action 처리용)
    queue: null, // 상태 업데이트 큐 (setState 관련)

    next: null, // 다음 Hook (연결 리스트)
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 이미 Hook이 하나 이상 있다면, 리스트에 새로 추가
    //기존 Hook 리스트의 끝에 새 Hook을 붙이고,
    //현재 위치 포인터도 새 Hook으로 이동
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
```

훅 객체를 생성할 때, baseQueue와 queue, next가 있고 list의 end에 append한다는 것을 보니 훅을 `linked list` 형태로 저장하는 방식으로 구현하고 있음을 알 수 있었다.

Hook이 처음 호출되는 **mount 단계에서**, 현재 렌더링 중인 컴포넌트의 Fiber 노드(currentlyRenderingFiber)에 Hook을 하나씩 **연결 리스트 형태로** 추가한다.

리액트 개발자라면, 누구나 **React의 hook은 항상 함수의 최상단에서 동일한 순서로 호출되어야한다**는 오류를 본 적이 있을 것이다.
`workInProgressHook`는 순서대로 연결된 Linked List를 따라 가면서 `useState`, `useEffect` 등을 "그 위치에 있는 Hook"으로 인식하기 때문이다.

hook은 linked list로 이루어져 있구나! 라고 생각하던 중, hook의 queue를 다루는 함수의 코드에서 주석에 // This is the first update. Create a circular list.라고 적혀있는 것을 발견했다.
circular list는 어떻게 사용되는 것일까?

# circular queue를 이용해 상태 update하기

## enqueueRenderPhaseUpdate()

```js
function enqueueRenderPhaseUpdate<S, A>(
  queue: UpdateQueue<S, A>,
  update: Update<S, A>
): void {
  // This is a render phase update. Stash it in a lazily-created map of
  // queue -> linked list of updates. After this render pass, we'll restart
  // and apply the stashed updates on top of the work-in-progress hook.
  didScheduleRenderPhaseUpdateDuringThisPass =
    didScheduleRenderPhaseUpdate = true;
  const pending = queue.pending;
  if (pending === null) {
    // This is the first update. Create a circular list.
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
}
```

이 함수는 **렌더링 중 발생한 업데이트**를 큐에 추가하는 함수이다.
dispatchReducerAction(), dispatchSetStateInternal() (mountReducer, updateState)에 사용된다.

parameter로 받는 queue와 update는 아래와 같은 역할을 한다.

- `queue`: 상태 업데이트를 보관하는 큐 (`useState`, `useReducer` 등에서 공유)
- `update`: 새로 들어온 업데이트 객체 (`setState(x)`가 만든 값)

pending이 null일 때 update.next를 next로 설정하여 circular list를 만들고 있다.
왜 circular list로 구현했을까? 궁금해서 더 살펴보았다.

### baseQueue란?

우선 `baseQueue`에 대해 알아보자.

`baseQueue`는 실행 예정인 업데이트들을 저장하는 큐이다. `queue.pending`과 병합한 후 사용한다.

다시 돌아와서, 왜 circular list로 구현했을까에 대해 추측해보았다.
아래는 updateReducerImpl(updateReducer, updateState에서 사용)으로, 상태를 업데이트하는 코드다.

```js
function updateReducerImpl<S, A>(
  const baseState = hook.baseState;
  if (baseQueue === null) {
    // If there are no pending updates, then the memoized state should be the
    // same as the base state. Currently these only diverge in the case of
    // useOptimistic, because useOptimistic accepts a new baseState on
    // every render.
    hook.memoizedState = baseState;
    // We don't need to call markWorkInProgressReceivedUpdate because
    // baseState is derived from other reactive values.
  } else {
    // We have a queue to process.
    const first = baseQueue.next;
    let newState = baseState;

    let newBaseState = null;
    let newBaseQueueFirst = null;
    let newBaseQueueLast: Update<S, A> | null = null;
    let update = first;
    let didReadFromEntangledAsyncAction = false;

	//업데이트를 실행하는 부분
    do {
        // Process this update.
        const action = update.action;
        newState = reducer(newState, action);
      }
      update = update.next;
    } while (update !== null && update !== first);

    //상태 업데이트
	 if (newBaseQueueLast === null) {
	  newBaseState = newState;
	} else {
	  newBaseQueueLast.next = newBaseQueueFirst;
	}

	// 상태 변경 확인 및 업데이트 마킹
	if (!is(newState, hook.memoizedState)) {
	  markWorkInProgressReceivedUpdate();
	}

	hook.memoizedState = newState;
	hook.baseState = newBaseState;
	hook.baseQueue = newBaseQueueLast;
}
```

first를 baseQueue.next로 설정-> update를 first로 설정-> update가 모두 소진될 때 까지 update를 실행하며 update=update.next로 설정하는 로직이다.

당장 실행되어야 할 `update`를 `baseQueue.next`로 설정한다. 만약 처음에 `baseQueue`가 하나만 있는 경우 next가 없어 접근할 수 없는 경우가 있을 수 있기 때문에, `circular linked list`로 구현한 것이 아닐까? 추측할 수 있었다.

# 느낀점

리액트의 소스코드를 살펴보며, react hook이 linked list로 구현되어 있으며, react hook의 update는 queue(circular linked list)로 구현되었다는 것을 알게 되었다. queue를 이용하여 순서대로 업데이트를 처리하고, lane을 이용해 우선순위까지 설정한다는 것이 놀라웠다.

내 글에서는 쉬운 이해를 위해 코드에서 핵심적이라고 생각되는 부분만 잘라서 들고왔는데, 실제 코드를 보면 엄청 복잡하게 렌더링 중 업데이트가 되는 경우/아닌 경우/우선순위가 낮은 경우 등이 모두 고려되어 있었다. 이렇게 길고 고려사항이 많은 코드라 그만큼 안정적인가 싶었다.

리액트 19 소스코드를 기준으로 분석하고 싶었는데, 인터넷에 참고할만한 글은 리액트 16이 대부분이라 조금 헤맸다. 리액트 16의 dispatchAction, baseUpdate 등은 리액트 19에서 더이상 사용되지 않는다. 그래도 16에서는 이렇게 구현했던 걸 19에서는 왜 이렇게 바꿨을까? 고민해볼 수 있어서 좋았다.

리액트 16에서는 baseUpdate를 이용해 update가 어디까지 실행되었는지를 저장하고, 이 정보를 기반으로 queue에서 다음 update를 실행하는데, 리액트 19에서는 baseUpdate가 사라지고 baseQueue가 있으며 queue와 baseQueue를 병합한 후 baseQueue를 통해 update를 실행하므로 어디까지 실행되었는지 따로 저장하지 않고 queue로만 움직인다. 그래서 더 직관적인 것 같다.
(baseQueue는 18->19에서 추가된 것이 아니며, 글을 작성한 기준이 19이므로 19라고 칭합니다.)

리액트로 개발을 하면서도 리액트에 대해 잘 모르고 있었음을 알게 되었다. 다음에는 직접 바닐라 js로 리액트를 구현해보고 싶다.

### 참고 자료

react fiber을 이해하는데 도움을 받은 글
[React Fiber 아키텍처 딥다이브](https://velog.io/@alsgud8311/React-Fiber-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98-%EB%94%A5%EB%8B%A4%EC%9D%B4%EB%B8%8C)
[React 파이버 아키텍처 분석](https://d2.naver.com/helloworld/2690975)

리액트 16.8을 기준으로 작성된 글
[React 톺아보기 - 03. Hooks_2 \| Deep Dive Magic Code](https://goidle.github.io/react/in-depth-react-hooks_2/)
[React useState 소스코드 분석하기 \| D5BL5G](https://d5br5.dev/blog/deep_dive/react_useState_source_code)
[진짜 리액트는 어떻게 생겼나? (2) - renderWithHooks와 훅의 본체 — \_0422의 생각](https://0422.tistory.com/322)

react 18 이상을 기준으로 작성된 글
[\[React\] Hook 낯설게하기 - useState & useReducer](https://velog.io/@shinhw371/React-Hook-%EB%82%AF%EC%84%A4%EA%B2%8C%ED%95%98%EA%B8%B0-1)
[\[리액트\] 깊이알아보는 useState, 리렌더링핵심 작동원리 - 방황하는 하이에나들을 위한 블로그](https://joong-sunny.github.io/react/react1/)

react 18 이상을 기준으로 작성되었으며 ReactFiberHooks.js분만 아니라 다른 파일도 함께 살펴보며 전체적인 렌더링 로직을 이해할 수 있는 글. 추천합니다
[Medium](https://s4mprk.medium.com/%EB%A6%AC%EC%95%A1%ED%8A%B8-%EB%82%B4%EB%B6%80-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-5-usestate-0c7d779997a9)
