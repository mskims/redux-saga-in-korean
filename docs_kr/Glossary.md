# 용어사전

<!-- # Glossary -->

이 문서는 리덕스 사가에 핵심 용어집입니다.

<!-- This is a glossary of the core terms in Redux Saga. -->

### 이펙트

<!-- ### Effect -->

이펙트는 사가의 미들웨어가 실행할 명령을 포함하고 있는 평범한 자바스크립트 객체입니다.

<!-- An effect is a plain JavaScript Object containing some instructions to be executed by the saga middleware. -->

리덕스 사가 라이브러리를 통해 제공되는 팩토리 함수를 통해 이펙트를 만들 수 있습니다. 예를 들어 `call(myfunc, 'arg1', 'arg2')`를 사용하여 미들웨어가 `myfunc('arg1', 'arg2')`를 호출하도록 할 수 있으며, yield된 이펙트에 대한 결과는 제너레이터로 반환됩니다.

<!-- You create effects using factory functions provided by the redux-saga library. For example you use `call(myfunc, 'arg1', 'arg2')` to instruct the middleware to invoke `myfunc('arg1', 'arg2')` and return the result back to the Generator that yielded the effect -->

### 테스크

<!-- ### Task -->

테스크는 백그라운드에서 실행되는 프로세스와 같습니다. 리덕스 사가 기반의 애플리케이션은 여러 테스크들을 병렬로 실행시킬 수 있습니다. `fork` 함수를 통해 이러한 테스크들을 생성할 수 있습니다.

<!-- A task is like a process running in background. In a redux-saga based application there can be multiple tasks running in parallel. You create tasks by using the `fork` function -->

```javascript
function* saga() {
  ...
  const task = yield fork(otherSaga, ...args)
  ...
}
```

### 블로킹/논블로킹 호출

<!-- ### Blocking/Non-blocking call -->

블로킹 호출은 Saga가 이펙트를 yield 하면 실행에 대한 결과를 기다렸다가 제네레이너 내부에서 다음 명령어의 실행을 재개합니다.

<!-- A Blocking call means that the Saga yielded an Effect and will wait for the outcome of its execution before resuming to the next instruction inside the yielding Generator. -->

논블로킹 호출은 Saga가 이펙트를 yield한 이후 바로 실행을 재개한다는 것을 의미합니다.

<!-- A Non-blocking call means that the Saga will resume immediately after yielding the Effect. -->

예를 들어,

<!-- For example -->

```javascript
function* saga() {
  yield take(ACTION)              // 블로킹: 액션을 기다립니다.
  yield call(ApiFn, ...args)      // 블로킹: ApiFn 함수를 기다립니다.
  yield call(otherSaga, ...args)  // 블로킹: otherSaga 가 종료될때까지 기다립니다.

  yield put(...)                   // 논블로킹: 내부 스케줄러에서 디스패치됩니다.

  const task = yield fork(otherSaga, ...args)  // 논블로킹: otherSaga 를 기다리지 않습니다.
  yield cancel(task)                           // 논블로킹: 실행을 즉시 재개합니다.
  // or
  yield join(task)                              // 블로킹: task가 종료될때까지 기다립니다.
}
```

### 감시자/워커

<!-- ### Watcher/Worker -->

각각 두 개의 Saga를 이용하여 제어 흐름을 구성하는 방법을 나타냅니다.

<!-- refers to a way of organizing the control flow using two separate Sagas -->

- 감시자(The watcher): 디스패치된(dispatched) 액션을 관찰하고 모든 액션에 대해 워커(worker)를 포크합니다.

<!-- - The watcher: will watch for dispatched actions and fork a worker on every action -->

- 워커(The worker): 액션을 처리하고 종료합니다.

<!-- - The worker: will handle the action and terminate -->

예시

<!-- example -->

```javascript
function* watcher() {
  while (true) {
    const action = yield take(ACTION)
    yield fork(worker, action.payload)
  }
}

function* worker(payload) {
  // ... do some stuff
}
```
