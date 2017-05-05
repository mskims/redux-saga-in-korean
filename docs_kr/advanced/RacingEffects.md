## 여러 이펙트의 경주

가끔씩 우리는 여러 태스크를 병렬로 시작하지만, 그 태스크들을 전부 기다리고 싶지는 않을 때가 있습니다. 우리는 그저 *우승자*가 필요할 뿐입니다: 첫 번째로 resolve(혹은 reject)된 태스크 말입니다. `race` 이펙트는 여러 이펙트를 경주할 수 있게 합니다.

다음 예시는 원격 요청을 하는 태스크입니다. 그리고 응답 시간을 1초로 제한하고 있습니다.

```javascript
import { race, take, put } from 'redux-saga/effects'
import { delay } from 'redux-saga'

function* fetchPostsWithTimeout() {
  const {posts, timeout} = yield race({
    posts: call(fetchApi, '/posts'),
    timeout: call(delay, 1000)
  })

  if (posts)
    put({type: 'POSTS_RECEIVED', posts})
  else
    put({type: 'TIMEOUT_ERROR'})
}
```

`race`의 다른 유용한 기능 중 하나는 **경주에서 진 이펙트들을 자동으로 취소시키는 것**입니다. 예를 들어, 우리가 두 개의 UI 버튼이 있다고 가정해봅시다:

- 첫 번째 버튼은 백그라운드에서 무한루프 `while (true)` 안에서 태스크를 시작합니다 (예를 들어 매 x초마다 데이터를 서버와 싱크하는 태스크가 있겠죠).

- 백그라운드 태스크가 시작하면, 우리는 두 번째 버튼을 눌러 이 태스크를 취소할 수 있습니다.

```javascript
import { race, take, put } from 'redux-saga/effects'

function* backgroundTask() {
  while (true) { ... }
}

function* watchStartBackgroundTask() {
  while (true) {
    yield take('START_BACKGROUND_TASK')
    yield race({
      task: call(backgroundTask),
      cancel: take('CANCEL_TASK')
    })
  }
}
```

이 경우에서 `CANCEL_TASK` 액션이 dispatch되면, `race` 이펙트는 자동으로 `backgroundTask`를 취소할 것입니다. 그리고 그 태스크 내부에 취소 에러를 throw할 것입니다.
