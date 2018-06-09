# 사가 조합

`yield*`은 관용적으로 사가 조합을 가능하게 해 주지만, 이 접근법에는 한계가 있습니다:

- 뭉쳐진 제너레이터들을 분리하여 테스트하고 싶으실 겁니다. 이는 테스트 코드 안에서의 중복뿐만 아니라, 중복된 실행의 오버헤드 또한 발생시킵니다. 우리는 같은 제너레이터를 중첩하여 실행시키고 싶지 않고, 그것이 올바른 인수로 호출되었는지만 확인하고 싶습니다.

- 더 중요한 것은, `yield*`는 오직 순차적인 태스크들의 조합만 허용하기 때문에, 한 번에 하나의 제너레이터만 `yield*`할 수 있습니다.

`yield`를 사용하여 하나 혹은 그 이상의 서브 태스크들을 병렬로 시작할 수 있습니다. 제너레이터를 `yield`할 시에, 사가는 더 진행하기 전에 제너레이터가 종료되기를 기다릴 것입니다. 그런 다음에는 제너레이터로부터 반환된 값(혹은 서브 태스크로부터 전해진 에러)으로 다시 진행할 것입니다.

```javascript
function* fetchPosts() {
  yield put(actions.requestPosts())
  const products = yield call(fetchApi, '/products')
  yield put(actions.receivePosts(products))
}

function* watchFetch() {
  while (yield take(FETCH_POSTS)) {
    yield call(fetchPosts) // waits for the fetchPosts task to terminate
  }
}
```

배열으로 이루어진 여러 제너레이터를 `yield`하면 그 안의 서브 제너레이터들은 병렬로 시작하게 될 것이고, 그들이 끝나기를 기다린 다음에, 반환된 모든 결과를 가지고 진행될 것입니다.

```javascript
function* mainSaga(getState) {
  const results = yield all[call(task1), call(task2), ...]
  yield put(showResults(results))
}
```

사실, 사가를 `yield`하는 것은 다른 이펙트(액션, `timeout` 등)를 `yield`하는 것과 다르지 않습니다. 사가들을 이펙트 조합자를 이용하여 다른 모든 타입과 같이 조합할 수 있습니다.

예를 들어, 제한 시간 내에 어떤 게임을 끝내고 싶을 수 있습니다.

```javascript
function* game(getState) {
  let finished
  while (!finished) {
    // has to finish in 60 seconds
    const {score, timeout} = yield race({
      score: call(play, getState),
      timeout: call(delay, 60000)
    })

    if (!timeout) {
      finished = true
      yield put(showScore(score))
    }
  }
}
```
