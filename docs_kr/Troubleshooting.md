# 문제 해결

### 사가를 추가한 후 앱이 멈춘다.

제너레이터 함수의 `yield` 이펙트를 확인하세요.

아래의 예제를 살펴보도록 합시다:

```js
import { take } from 'redux-saga/effects'

function* logActions() {
  while (true) {
    const action = take() // wrong
    console.log(action)
  }
}
```

이 예제코드는 애플리케이션을 무한 루프에 빠지게 만듭니다. 왜냐하면 `take()` 함수는 이펙트에 대한 설명만 생성합니다. 이 코드를 미들웨어를 실행하기 위해 `yield` 하지 않는 한, `while` 루프는 일반적인 `while` 루프와 같이 동작할 것이며, 애플리케이션은 멈추게 될 것이다.

<!-- It will put the application into an infinite loop because `take()` only creates a description of the effect. Unless you `yield` it for the middleware to execute, the `while` loop will behave like a regular `while` loop, and freeze your application. -->

`yield` 를 추가하는 것은 제너레이터가 일시정지되며, 이펙트를 실행시킬 리덕스 사가 미들웨어에 대한 제어권이 반환된다. `take()` 함수의 경우, 리덕스 사가 다음 액션의 패턴에 일치할 때까지 기다리며, 일치한 다음에서야 제네레이터를 다시 시작합니다.
 
<!-- Adding `yield` will pause the generator and return control to the Redux Saga middleware which will execute the effect. In case of `take()`, Redux Saga will wait for the next action matching the pattern, and only then will resume the generator. -->

위의 예제를 수정하기 위해 `take()`으로부터 단순히 반환된 이펙트를 `yield` 합니다.

<!-- To fix the example above, simply `yield` the effect returned by `take()`: -->

```js
import { take } from 'redux-saga/effects'

function* logActions() {
  while (true) {
    const action = yield take() // correct
    console.log(action)
  }
}
```

### 사가 내에서 디스패치된 액션이 누락됩니다.

<!-- ### My Saga is missing dispatched actions -->

사가가 어떤 이펙트에 의해 봉쇄되지 않았는지 확인하세요. 사가가 이펙트가 해결되기를 기다리는 동안은 액션을 디스패치할 수 없습니다.

<!-- Make sure the Saga is not blocked on some effect. When a Saga is waiting for an Effect to resolve, it will not be able to take dispatched actions until the Effect is resolved. -->

예를 들어, 다음의 예제를 살펴보도록 하겠습니다.

<!-- For example, consider this example -->

```javascript
function* watchRequestActions() {
  while (true) {
    const {url, params} = yield take('REQUEST')
    yield call(handleRequestAction, url, params) // The Saga will block here
  }
}

function* handleRequestAction(url, params) {
  const response = yield call(someRemoteApi, url, params)
  yield put(someAction(response))
}
```

`watchRequestActions`이 `yield call(handleRequestAction, url, params)`를 실행시킬 때, 다음의 `yield take`가 실행되기 전에 `handleRequestAction`가 종료 될때까지 기다릴 것이다. 예제를 통해 다음의 일련의 이벤트가 발생한다는 것을 추측할수 있다.

<!-- When `watchRequestActions` performs `yield call(handleRequestAction, url, params),` it'll wait for `handleRequestAction` until it terminates an returns before continuing on the next `yield take`. For example suppose we have this sequence of events -->

```
UI                     watchRequestActions             handleRequestAction
-----------------------------------------------------------------------------
.......................take('REQUEST').......................................
dispatch(REQUEST)......call(handleRequestAction).......call(someRemoteApi)... Wait server resp.
.............................................................................
.............................................................................
dispatch(REQUEST)............................................................ Action missed!!
.............................................................................
.............................................................................
.......................................................put(someAction).......
.......................take('REQUEST')....................................... saga is resumed
```

위에서 언급했듯, **봉쇄 호출(blocking call)** 에 의해 사가가 봉쇄되면 중간에 디스패치된 액션들은 누락됩니다. 

<!-- As illustrated above, when a Saga is blocked on a **blocking call** then it will miss
all the actions dispatched in-between. -->

사가의 봉쇄를 피하기 위해, `call` 대신에 `fork`를 사용함으로 **논 블록킹 호출**을 사용할 수 있다.  

<!-- To avoid blocking the Saga, you can use a **non-blocking call** using `fork` instead of `call` -->

```javascript
function* watchRequestActions() {
  while (true) {
    const {url, params} = yield take('REQUEST')
    yield fork(handleRequestAction, url, params) // The Saga will resume immediately
  }
}
```
