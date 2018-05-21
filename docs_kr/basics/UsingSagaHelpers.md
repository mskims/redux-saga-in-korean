# 헬퍼 함수

`redux-saga` 는 스토어에 몇몇 지정된 액션들이 dispatch 되었을때 태스크를 만들기 위해 내부 함수들을 감싸는 몇몇 헬퍼 이펙트들을 제공합니다.
<!--`redux-saga` provides some helper effects wrapping internal functions to spawn tasks when some specific actions are dispatched to the Store.-->

헬퍼 함수들은 저레벨 API의 상단에 내장되어있습니다. 우리는 심화 섹션에서 어떻게 이 함수들이 구현될 수 있는지 볼겁니다.
<!--The helper functions are built on top of the lower level API. In the advanced section, we'll see how those functions can be implemented.-->

첫번째로 살펴볼 헬퍼 함수는 `redux-thunk` 와 비슷한 기능을 제공하는 아주 유명한 `takeEvery` 입니다
<!--The first function, `takeEvery` is the most familiar and provides a behavior similar to `redux-thunk`.-->

흔한 AJAX 예제와 설명해보겠습니다. 클릭할때 마다 `FETCH_REQUESTED` 액션을 dispatch 하는 버튼이 있고,
서버로부터 받은 데이터를 fetch 시키게 하는 어떤 태스크를 실행해서 이 액션을 핸들링 해봅시다.
<!-- Let's illustrate with the common AJAX example. On each click on a Fetch button we dispatch a `FETCH_REQUESTED` action. -->
<!-- We want to handle this action by launching a task that will fetch some data from the server. -->

먼저 비동기 액션을 수행하는 태스크를 만듭시다.
<!-- First we create the task that will perform the asynchronous action: -->

```javascript
import { call, put } from 'redux-saga/effects'

export function* fetchData(action) {
   try {
      const data = yield call(Api.fetchUser, action.payload.url)
      yield put({type: "FETCH_SUCCEEDED", data})
   } catch (error) {
      yield put({type: "FETCH_FAILED", error})
   }
}
```

위의 태스크를 각각의 `FETCH_REQUESTED` 액션마다 실행하려면..:
<!-- To launch the above task on each `FETCH_REQUESTED` action: -->

```javascript
import { takeEvery } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeEvery('FETCH_REQUESTED', fetchData)
}
```

위 예제에서 `takeEvery` 는 여러개의 `fetchData` 인스턴스를 동시에 시작되게 합니다.
한개 혹은 여러개의 아직 종료되지 않은 `fetchData` 태스크들이 있더라도 새로운 `fetchData` 태스크를 시작할 수 있습니다.
<!-- In the above example, `takeEvery` allows multiple `fetchData` instances to be started concurrently. At a given moment, -->
<!-- we can start a new `fetchData` task while there are still one or more previous `fetchData` tasks which have not yet terminated. -->

만약 마지막으로 발생된 리퀘스트의 응답만 얻고싶다면, `takeLatest` 헬퍼를 사용할 수 있습니다. (예: 항상 마지막 버전의 데이터만 보여주어야 할 때)
<!-- If we want to only get the response of the latest request fired -->
<!-- (e.g. to always display the latest version of data) we can use the `takeLatest` helper: -->

```javascript
import { takeLatest } from 'redux-saga/effects'

function* watchFetchData() {
  yield takeLatest('FETCH_REQUESTED', fetchData)
}
```

`takeEvery` 와 달리, `takeLatest` 는 어느 순간에서도 단 하나의 `fetchData` 태스크만 실행되게 합니다. 마지막으로 시작된 태스크가 되겠죠.
만약 `fetchData` 태스크가 시작되었을때 이전 태스크가 실행중이라면, 이전 태스크는 자동적으로 취소될겁니다.
<!-- Unlike `takeEvery`, `takeLatest` allows only one `fetchData` task to run at any moment. -->
<!-- And it will be the latest started task. If a previous task is still running when another `fetchData` task is started, the previous task will be automatically cancelled. -->

다른 액션들을 보고있는 여러개의 Saga 들을 가지고계시다면, Saga들을 생성하기위해 사용된 `fork` 와 비슷한 동작을 하는 내장 함수 들과 함께 여러개의 워쳐들을 만들 수 있습니다. 
(`fork` 에 대해선 나중에 말해보죠, 지금은 여러개의 Saga들을 백그라운드에서 시작할 수 있게 해주는 이펙트라고 여기세요.)
<!-- If you have multiple Sagas watching for different actions, you can create multiple watchers with those built-in helpers which will behave like there was `fork` used to spawn them (we'll talk about `fork` later. For now consider it to be an Effect that allows us to start multiple sagas in the background) -->

예:
<!-- For example: -->

```javascript
import { takeEvery } from 'redux-saga'

// FETCH_USERS
function* fetchUsers(action) { ... }

// CREATE_USER
function* createUser(action) { ... }

// use them in parallel
export default function* rootSaga() {
  yield takeEvery('FETCH_USERS', fetchUsers)
  yield takeEvery('CREATE_USER', createUser)
}
```
