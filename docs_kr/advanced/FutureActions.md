# Pulling future actions

지금까지 우리는 들어오는 각각의 액션에 새로운 테스크를 만들기 위해서 `takeEvery` 헬퍼 이펙트를 사용했습니다. 이것은 약간 redux-thunk를 흉내낸 것 같습니다: 예를 들어, 매번 `fetchProducts` 액션 크리에이터를 호출하면, 그 액션 크리에이터는 데이터 컨트롤을 위해 썽크를 dispatch하는 것과 같네요.

사실, `takeEvery`는 로우 레벨의 끝에 있는 내부적인 헬퍼 함수와 강력한 API가 감싸진 이펙트에 불과합니다. 이 장에서 우리는 새로운 이펙트 `take`를 보게 될 것입니다. 이 이펙트는 액션 감시 프로세스의 전체적인 제어를 가능하게 함으로써, 복잡한 데이터 컨트롤 플로우를 설계할 수 있게 합니다.

## 간단한 로거

이제 사가의 간단한 예제를 봅시다. 이 사가는 스토어로 dispatch되는 모든 액션들을 watch하고, 콘솔으로 로그를 찍어줍니다.

`takeEvery('*')`(와일드카드 `*` 패턴)를 씀으로써, 우리는 액션의 타입과는 무관하게 모든 액션을 잡아낼 수 있습니다.

```javascript
import { select, takeEvery } from 'redux-saga/effects'

function* watchAndLog() {
  yield takeEvery('*', function* logger(action) {
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  })
}
```

이제, 위와 같은 플로우를 이행하기 위해 어떻게 `take` 이펙트를 사용하는지 봅시다.


```javascript
import { select, take } from 'redux-saga/effects'

function* watchAndLog() {
  while (true) {
    const action = yield take('*')
    const state = yield select()

    console.log('action', action)
    console.log('state after', state)
  }
}
```

`take`는 우리가 전에 봤던 `call`, `put`와 비슷합니다. 이는 특정한 액션을 기다리기 위해서 미들웨어에 알려주는 명령 오브젝트를 생성합니다. `call` 이펙트는 미들웨어가 프로미스의 resolve를 기다리게 합니다. `take`의 경우에는 미들웨어가 매칭되는 액션이 dispatch될 때까지 기다립니다. 위의 예에서, `watchAndLog`는 어떠한 한 액션이 dispatch될 때까지 기다릴 겁니다.

우리가 어떻게 무한 루프 `while (true)`를 실행시키는지 주목해 주세요. 이건 제너레이터 함수라는 것을 기억하세요. 제너레이터는 완료를 향해 달려가는(run-to-completion) 함수가 아닙니다. 우리가 만든 제너레이터는 한 번 반복될 때마다 액션이 일어나기를 기다릴 것입니다.

`take`를 사용하는 것은 우리의 코드 작성법에 대해 작은 충격을 줍니다. `takeEvery`의 경우에, 실행된 태스크는 그들이 다시 실행될 때에 대한 관리 방법이 없습니다. 그저 각각의 매칭되는 액션에 실행되고, 다시 실행되겠죠. 또한 그들은 언제 감시(옵저빙)를 멈춰야 하는지에 대한 관리 방법도 없습니다.

`take`의 경우에는 컨트롤의 방향이 정반대입니다. 핸들러 태스크에 *푸시*되고 있는 액션들 대신, 사가는 스스로 액션들을 *풀링*합니다. 이는 사가가 일반 함수 콜을 하는 것처럼 보입니다. 액션이 dispatch되었을 때 resolve하는 `action = getNextAction()`처럼요.

이 컨트롤의 전환은 특별한 컨트롤 플로우를 수행할 수 있게 합니다. 전통적인 액션의 *푸시* 접근법을 해결해주죠.

간단한 예로, 우리가 Todo 어플리케이션에서 유저의 액션들을 watch하고 있다가, 유저가 세 개의 todo를 만들면 축하 메세지를 띄우게 한다고 가정해봅시다.

```javascript
import { take, put } from 'redux-saga/effects'

function* watchFirstThreeTodosCreation() {
  for (let i = 0; i < 3; i++) {
    const action = yield take('TODO_CREATED')
  }
  yield put({type: 'SHOW_CONGRATULATION'})
}
```

`while (true)` 대신에, 우리는 딱 3번만 반복하는 `for` 루프를 만들었습니다. 처음 세 번의 `TODO_CREATED` 액션 후에, `watchFirstThreeTodosCreation`는 어플리케이션에게 축하 메시지를 띄우도록 요청하고 종료될 것입니다. 이는 제너레이터가 *가비지 콜렉션*이 되고, 더 이상의 쓸데 없는 감시는 없다는 것을 의미합니다.

*풀* 접근법의 다른 이점은 우리가 친숙한 동기적(synchronous) 스타일로 컨트롤 플로우를 표현할 수 있다는 것입니다. 예를 들어, 우리가 `LOGIN` 액션과 `LOGOUT` 액션을 이용하여 로그인 플로우를 실행시키고 싶다고 가정해봅시다. `takeEvery`(혹은 `redux-thunk`)을 이용했다면 `LOGIN`과 `LOGOUT`으로 나뉘어진 두 개의 태스크(혹은 썽크)를 작성해야 했을 것입니다.

우리의 결과물은 두 개로 나뉘어졌습니다. 누군가 진행 상황을 이해하기 위해서 우리들의 코드를 읽는다면, 그는 두 개의 핸들러의 소스를 읽고, 이 두 소스를 연결시켜 생각해야 합니다. 이는 그가 다양한 위치에 있는 코드들을 머리 속으로 올바른 순서로 재 정렬한 뒤에 플로우 모델을 재설계해야 한다는 것을 의미합니다.

풀 모델을 사용하면 우리는 같은 액션을 반복해서 핸들링하지 않고 같은 곳에 우리의 플로우를 작성할 수 있습니다.

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

`loginFlow` 사가는 더 깔끔하게 예상되는 액션 순서를 전달합니다. 이 사가는 `LOGIN` 액션이 언제나 `LOGOUT` 액션 전에 오고, 반대로 `LOGOUT` 액션이 언제나 `LOGIN` 전에 와야 한다는 것을 알고 있습니다 (좋은 UI는 예상되지 않은 액션을 숨기거나 비활성화하여, 언제나 안정된 순서의 액션들을 시행해야 합니다).
