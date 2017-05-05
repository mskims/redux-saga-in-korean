# 비봉쇄(non-blocking) 호출

이전 장에서, 우리는 `take` 이펙트로 특별한 플로우를 표현하는 방법을 알아보았습니다.

다시 로그인 플로우 예제를 봅시다:

```javascript
function* loginFlow() {
  while (true) {
    yield take('LOGIN')
    // ... perform the login logic
    yield take('LOGOUT')
    // ... perform the logout logic
  }
}
```

이 예제를 실제 로그인/로그아웃 로직이 수행되도록 완료해봅시다. 우리가 원격 서버에 유저의 권한을 주는 API를 가지고 있다고 가정해봅시다. 권한 부여가 성공적이라면, 서버는 권한에 대한 토큰을 리턴할 것입니다. 토큰은 DOM 저장소를 이용하는 어플리케이션에 의해 저장됩니다 (우리의 API가 DOM 저장소를 위한 다른 기능도 제공한다고 가정해봅시다).

유저가 로그아웃 했을 때, 우리는 간단히 이전에 저장했던 권한 토큰을 지울 것입니다.

### 첫 시도

이제 우리는 위에서 설명했던 플로우를 수행하기 위해서 필요한 모든 이펙트를 알고 있습니다. 우리는 `take` 이펙트를 이용해 스토어의 특정한 액션들을 기다릴 수 있습니다. 그리고 우리는 `call` 이펙트를 이용해 비동기 호출을 할 수도 있습니다. 마지막으로, 우리는 `put` 이펙트를 이용해서 액션들을 스토어로 dispatch할 수 있습니다.

자, 그렇다면 시도해봅시다:

> 주의: 아래의 코드는 작은 문제가 있습니다. 장을 마지막까지 읽어주시길 바랍니다.

```javascript
import { take, call, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    const token = yield call(authorize, user, password)
    if (token) {
      yield call(Api.storeItem, {token})
      yield take('LOGOUT')
      yield call(Api.clearItem, 'token')
    }
  }
}
```

첫 번째로 우리는 분리된 제너레이터 `authorize`를 생성했습니다. 이는 실제 API 호출을 한 뒤, 성공 여부를 스토어에 알릴 것입니다.

`loginFlow`는 `while (true)` 루프 안의 전체 플로우를 수행합니다. 이는 플로우의 마지막 단계(`LOGOUT`)에 도달했을 때, 새로운 `LOGIN_REQUEST` 액션을 기다리며 새로운 반복을 시작한다는 것을 의미합니다.

`loginFlow`는 처음 `LOGIN_REQUEST` 액션을 기다립니다. 그런 다음 액션의 payload에서 유저의 증명(`user`와 `password`)을 가져오고 `authorize` 태스크를 `call`함수로 호출합니다.

알아채셨듯이, `call`은 프로미스를 반환하는 함수들만을 위한 것이 아닙니다. `call`은 제너레이터 함수들을 실행하는 데에도 사용할 수 있습니다. 위의 예제에서, **`loginFlow`는 `authorize`가 종료되고 반환할 때까지 기다릴 것입니다** (즉 API 호출 이후에, 액션을 dispatch하고, `loginFlow`에 토큰을 반환할 때까지).

만약 API 호출이 성공했다면, `authorize`는 `LOGIN_SUCCESS` 액션을 dispatch할 것이고, 그 다음에는 가져온 토큰을 반환할 것입니다. 만약 에러가 발생했다면, `LOGIN_ERROR` 액션을 dispatch할 것입니다.

만약 `authorize` 호출이 성공적이라면, `loginFlow`는 반환된 토큰을 DOM 저장소에 저장하고 `LOGOUT` 액션을 기다릴 것입니다. 유저가 로그아웃할 때, 우리는 저장된 토큰을 지우고, 새로운 유저의 로그인을 기다릴 것입니다.

`authorize`가 실패했을 경우에는 undefined 값을 반환할 것입니다. 반환된 값은 `loginFlow`에게 전의 프로세스를 스킵하고 새로운 `LOGIN_REQUEST` 액션을 기다리게 할 것입니다.

어떻게 전체 로직이 한 곳에 저장되는지 관찰해보세요. 우리의 코드를 읽는 새로운 개발자는 컨트롤 플로우를 이해하기 위해서 다양한 장소를 오가며 여행할 필요가 없습니다. 이것은 마치 동기(synchronous) 알고리즘을 읽는 것 같습니다: 각 단계가 자연스러운 순서에 놓여있습니다. 그리고 우리는 다른 함수들을 호출하고 그 결과를 기다리는 함수들을 가지고 있습니다.

### 하지만 위의 접근에는 아직도 작은 문제가 있습니다

`loginFlow`가 밑의 예제처럼 주어지는 호출의 resolve를 기다리고 있을 때를 가정해봅시다:

```javascript
function* loginFlow() {
  while (true) {
    // ...
    try {
      const token = yield call(authorize, user, password)
      // ...
    }
    // ...
  }
}
```

유저는 `LOGOUT` 액션이 dispatch되는 `Logout` 버튼을 클릭할 겁니다.

다음은 일어날 수 있는 이벤트의 순서를 표현한 것입니다.

```
UI                              loginFlow
--------------------------------------------------------
LOGIN_REQUEST...................call authorize.......... waiting to resolve
........................................................
........................................................
LOGOUT.................................................. missed!
........................................................
................................authorize returned...... dispatch a `LOGIN_SUCCESS`!!
........................................................
```

`loginFlow`가 `authorize` 호출에 의해 봉쇄되었을 때(blocked), 호출과 응답 사이에서 발생된 `LOGOUT`은 무시될 것입니다. 왜냐하면, `loginFlow`는 아직 `yield take('LOGOUT')`를 만나지 않았기 때문입니다.

위 코드의 문제는 `call`이 봉쇄(blocking) 이펙트라는 것입니다. 즉, 제너레이터는 호출이 종료되기 전까지는 아무것도 수행할 수 없습니다. 하지만 우리의 경우, 우리는 `loginFlow`가 `authorize` 호출 뿐만 아니라, 호출의 중간에서 일어날 수 있는 우발적인 `LOGOUT` 액션 또한 watch하기를 원합니다. `LOGOUT`은 `authorize` 호출과 *동시 발생적*이기 때문입니다.

그래서 우리가 필요한 것은 `authorize`를 봉쇄하지 않고 시작해서, `loginFlow`가 동시 발생적이고 우발적인 `LOGOUT`을 계속해서 watch할 수 있도록 하는 방법입니다.

비봉쇄(non-blocking) 호출을 위해서, 라이브러리는 [`fork`](https://redux-saga.js.org/docs/api/index.html#forkfn-args)라는 다른 이펙트를 제공합니다. 우리가 태스크를 *fork*한다면, 그 태스크는 백그라운드에서 시작되고, 호출자는 fork된 태스크가 종료될 때까지 기다리지 않고 플로우를 계속해서 진행합니다.

그래서 `loginFlow`가 동시 발생적인 `LOGOUT`를 놓치지 않게 하려면, 우리는 `authorize`를 `call`하면 안되고, `fork`를 사용해야만 합니다.

```javascript
import { fork, call, take, put } from 'redux-saga/effects'

function* loginFlow() {
  while (true) {
    ...
    try {
      // non-blocking call, what's the returned value here ?
      const ?? = yield fork(authorize, user, password)
      ...
    }
    ...
  }
}
```

이제 문제는, `authorize`의 액션이 백그라운드에서 실행되기 때문에, 우리가 `token` 결과를 얻을 수 없다는 것입니다 (왜냐하면, 우리는 결과를 기다려야 하기 때문입니다). 그래서 우리는 토큰 저장소 관리 로직을 `authorize` 태스크로 옮겨야만 합니다.

```javascript
import { fork, call, take, put } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  }
}

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    yield fork(authorize, user, password)
    yield take(['LOGOUT', 'LOGIN_ERROR'])
    yield call(Api.clearItem, 'token')
  }
}
```

`yield take(['LOGOUT', 'LOGIN_ERROR'])`을 보세요. 이는 우리가 두 개의 동시 발생적인 액션들을 watch하고 있다는 것을 의미합니다:

- 만약 `authorize` 태스크가 유저의 로그아웃 전에 성공한다면, `LOGIN_SUCCESS` 액션을 dispatch하고 종료될 것입니다. 그러면 `loginFlow` 사가는 `LOGOUT` 액션만을 기다릴 것입니다 (이제 `LOGIN_ERROR`은 절대 일어나지 않을 것이기 때문이죠).

- 만약 `authorize`가 종료되기 전에 유저가 로그아웃한다면, `loginFlow`는 `LOGOUT` 액션을 받고 다음에 올 `LOGIN_REQUEST` 액션을 기다릴 것입니다.

`Api.clearItem` 호출이 *멱등적*인 것에 주의하세요. `authorize` 호출에 의해 저장된 토큰이 없다면 이는 아무런 효과가 없습니다. `loginFlow`는 다음 로그인을 기다리기 전에 저장된 토큰을 지워, 저장소에 아무런 토큰도 없다는 것을 보장해 줍니다.

아직 끝나지 않았습니다. 만약 우리가 API 호출 도중에 `LOGOUT`을 받는다면, 우리는 `authorize` 프로세스를 취소해야만 합니다. 아니면 두 개의 동시 발생적인 태스크들이 진행되게 될 것입니다: `authorize` 태스크는 계속해서 성공 혹은 실패하는 결과를 기다릴 것이고, `LOGIN_SUCCESS` 혹은 `LOGIN_ERROR`를 dispatch해서 엇갈린 상태를 만들게 될 것입니다.

우리는 전용 이펙트 [`cancel`](https://redux-saga.js.org/docs/api/index.html#canceltask)를 사용해서 fork된 태스크를 취소합니다.

```javascript
import { take, put, call, fork, cancel } from 'redux-saga/effects'

// ...

function* loginFlow() {
  while (true) {
    const {user, password} = yield take('LOGIN_REQUEST')
    // fork return a Task object
    const task = yield fork(authorize, user, password)
    const action = yield take(['LOGOUT', 'LOGIN_ERROR'])
    if (action.type === 'LOGOUT')
      yield cancel(task)
    yield call(Api.clearItem, 'token')
  }
}
```

`yield fork`는 [태스크 객체](https://redux-saga.js.org/docs/api/index.html#task)를 반환합니다. 우리는 지역 상수 `task`에 반환된 객체를 할당합니다. 나중에 만약 우리가 `LOGOUT` 액션을 받는다면, 우리는 그 태스크를 취소합니다. 만약 태스크가 실행 중이라면, 저지될 것입니다. 만약 태스크가 이미 완료되었다면, 아무런 일도 일어나지 않을 것이고, 취소 작업은 아무런 실행을 하지 않게 될 것입니다. 그리고 마지막으로, 만약 태스크가 에러로 종료되었다면, 우리는 아무것도 하지 않습니다. 왜냐하면 우리는 태스크가 이미 끝난 것을 알고 있기 때문입니다.

*거의* 다 끝났습니다 (동시 실행은 그렇게 만만하지 않습니다. 조금만 더 힘냅시다 :D)

우리가 `LOGIN_REQUEST` 액션을 받았을 때를 가정해봅시다. 우리의 리듀서는 `isLoginPending` 같은 플래그를 참으로 설정해서, 메시지나 스피너를 UI에서 보여줄 수 있습니다. 만약 우리가 API 호출 도중에 `LOGOUT`를 받고 태스크가 바로 정지된다면, 우리는 엇갈린 상태를 남긴 채로 끝날 수도 있습니다. 여전히 `isLoginPending` 가 참으로 설정되어 있고, 리듀서는 외부에서 오는 액션(`LOGIN_SUCCESS` 혹은 `LOGIN_ERROR`)을 기다리고 있을 것입니다.

다행히도, `cancel` 이펙트는 우리의 `authorize` 태스크를 잔인하게 없애버리지 않을 겁니다. 대신에 그들의 청소 로직을 실행할 기회를 줄 것입니다. 취소된 태스크는 `finally` 구간 안에서, 어떤 취소 로직이든지 다룰 수 있습니다. 왜냐하면, finally 구간은 모든 종류의 완료에서 실행되기 때문입니다 (일반 리턴, 에러, 강제로 취소됨). 만약 특별한 방법으로 취소 로직을 다루고 싶다면, `cancelled` 라는 이펙트를 사용하실 수 있습니다.

```javascript
import { take, call, put, cancelled } from 'redux-saga/effects'
import Api from '...'

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password)
    yield put({type: 'LOGIN_SUCCESS', token})
    yield call(Api.storeItem, {token})
    return token
  } catch(error) {
    yield put({type: 'LOGIN_ERROR', error})
  } finally {
    if (yield cancelled()) {
      // ... put special cancellation handling code here
    }
  }
}
```

아직 우리는 `isLoginPending` 상태를 처리하지 않았습니다. 그것에 대해서는 최소한 두가지 해결법이 있습니다:

- `RESET_LOGIN_PENDING` 라는 전용 액션을 dispatch하기
- 더욱 간단하게는, `LOGOUT` 액션에서 리듀서에게 `isLoginPending`를 처리하게 하기
