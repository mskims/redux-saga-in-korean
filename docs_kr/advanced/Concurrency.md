# 동시성 패턴

기본 장에서 우리는 이펙트 간의 동시성을 관리하기 위해 `takeEvery`와 `takeLatest`를  어떻게 사용하는지 배웠습니다.

이 장에서 우리는 이러한 헬퍼들이 어떻게 로우 레벨 이펙트를 사용하여 작동하는지 볼 것입니다.

## `takeEvery`

```javascript
function* takeEvery(pattern, saga, ...args) {
  const task = yield fork(function* () {
    while (true) {
      const action = yield take(pattern)
      yield fork(saga, ...args.concat(action))
    }
  })
  return task
}
```

`takeEvery`는 여러 개의 `saga`가 동시에 포크되게 합니다.

## `takeLatest`

```javascript
function* takeLatest(pattern, saga, ...args) {
  const task = yield fork(function* () {
    let lastTask
    while (true) {
      const action = yield take(pattern)
      if (lastTask)
        yield cancel(lastTask) // cancel is no-op if the task has already terminated

      lastTask = yield fork(saga, ...args.concat(action))
    }
  })
  return task
}
```

`takeLatest`는 여러 개의 사가 태스크들이 동시에 실행되게 하지 않습니다. 새로운 액션이 dispatch되자 마자, 그것은 자신을 제외한 이전의 모든 포크된 태스크를 취소합니다 (이미 작동 중이더라도).

`takeLatest`는 가장 나중의 응답만 받고 싶은 AJAX 요청을 다룰 때에 유용합니다.
