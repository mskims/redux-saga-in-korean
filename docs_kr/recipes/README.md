# 레시피

## 쓰로들링

`throttle` 헬퍼 함수를 사용해서 dispatch 된 액션들에 쓰로들링을 할 수 있습니다.

<!--편리한 내장 도우미(helper) 함수인 throttle을 사용하여 디스패치된 액션 순서를 조정할 수 있습니다. 예를 들어, 사용자가 텍스트 필드에 타이핑하는 동안 UI가`INPUT_CHANGED` 액션을 발생 시킨다고 가정해보겠습니다.-->

```javascript
import { throttle } from 'redux-saga/effects'

function* handleInput(input) {
  // ...
}

function* watchInput() {
  yield throttle(500, 'INPUT_CHANGED', handleInput)
}
```

throttle 헬퍼(helper) 함수를 사용하면 `watchInput`는 500ms동안 `handleInput` 작업을 새로 수행하지 않습니다. 동시에 가장 최신의 `INPUT_CHANGED` 액션을 `buffer`에 넣습니다. 그래서 `handleInput` 작업을 수행하지 않는 그 500ms 사이에 발생하는 `INPUT_CHANGED` 액션들을 모두 놓칠 것입니다. Saga는 각 500ms의 지연시간 동안 최대 하나의 `INPUT_CHANGED` 액션을 수행하고 후행 액션을 처리 할 수 있도록 보장합니다.


## 디바운싱(Debouncing)

내장 헬퍼(helper) 함수인 `delay`를 fork된 작업(아래 예제에서는 `handleInput`)에 넣음으로써 지연(debounce)을 줄 수 있습니다.

```javascript

import { delay } from 'redux-saga'

function* handleInput(input) {
  // 500ms마다 지연
  yield call(delay, 500)
  ...
}

function* watchInput() {
  let task
  while (true) {
    const { input } = yield take('INPUT_CHANGED')
    if (task) {
      yield cancel(task)
    }
    task = yield fork(handleInput, input)
  }
}
```

`delay` 함수는 promise를 사용하여 간단한 지연(debounce)을 구현합니다.
```
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms))
```

위 예제에서 `handleInput`은 로직을 수행하기 전에 500ms를 기다립니다. 만약 유저가 그 시간(500ms)동안 무언가를 타이핑한다면 `INPUT_CHANGED` 액션을 더 많이 얻게 됩니다. `handleInput`은 호출된 `delay` 함수에 의해 차단 될 것이기 때문에, 로직을 수행하기 전에`watchInput`에 의해 취소 될 것입니다.

위 예제는 또다른 헬퍼(helper) 함수인 `takeLatest`를 적용하여 다시 작성할 수 있습니다.

```javascript

import { delay } from 'redux-saga'

function* handleInput({ input }) {
  // 500ms마다 
  yield call(delay, 500)
  ...
}

function* watchInput() {
  // 현재 실행중인 handleInput 작업을 취소합니다.
  yield takeLatest('INPUT_CHANGED', handleInput);
}
```

## XHR호출 재시도(Retrying XHR calls)

특정 시간 동안 XHR 호출을 재시도하려면 지연(delay)이 있는 for 루프를 사용해야 합니다.

```javascript

import { delay } from 'redux-saga'

function* updateApi(data) {
  for(let i = 0; i < 5; i++) {
    try {
      const apiResponse = yield call(apiRequest, { data });
      return apiResponse;
    } catch(err) {
      if(i < 5) {
        yield call(delay, 2000);
      }
    }
  }
  // 시도가 5x2초 후에 실패했습니다.
  throw new Error('API request failed');
}

export default function* updateResource() {
  while (true) {
    const { data } = yield take('UPDATE_START');
    try {
      const apiResponse = yield call(updateApi, data);
      yield put({
        type: 'UPDATE_SUCCESS',
        payload: apiResponse.body,
      });
    } catch (error) {
      yield put({
        type: 'UPDATE_ERROR',
        error
      });
    }
  }
}

```

위 예제에서 `apiRequest`는 각각 2초의 지연시간을 가지고 5번 다시 시도됩니다. 5번째 실패 후에 던져진(thrown) 예외는 부모 사가(parent saga)에 의해 catch되고, 부모 사가는 'UPDATE_ERROR` 액션을 전달(디스패치, dispatch)합니다.

만약 무제한으로 제시도하기를 원한다면, `for` 반복문을 `while (true)`로 대체하면 가능합니다. 또한 `take`대신에 `takeLatest`를 사용하면 마지막 요청만 재시도할 수 있습니다. 에러 핸들링에서 `UPDATE_RETRY`액션을 추가하면, 업데이트가 성공적이지 않았지만 다시 시도 할 것임을 유저에게 알릴 수 있습니다.

```javascript
import { delay } from 'redux-saga'

function* updateApi(data) {
  while (true) {
    try {
      const apiResponse = yield call(apiRequest, { data });
      return apiResponse;
    } catch(error) {
      yield put({
        type: 'UPDATE_RETRY',
        error
      })
      yield call(delay, 2000);
    }
  }
}

function* updateResource({ data }) {
  const apiResponse = yield call(updateApi, data);
  yield put({
    type: 'UPDATE_SUCCESS',
    payload: apiResponse.body,
  });
}

export function* watchUpdateResource() {
  yield takeLatest('UPDATE_START', updateResource);
}

```

## 실행 취소(Undo)

실행 취소 기능은 '사용자가 자신이 무엇을 하고 있는지를 모르는 상황'을 가정합니다. 그 가정 아래 자연스럽게 이후 액션을 발생시켜 사용자의 선택을 존중합니다.(참고: [GoodUI](https://goodui.org/#8))

[redux documentation](http://redux.js.org/docs/recipes/ImplementingUndoHistory.html)은 `past`, `present`, `future` 상태(state)를 담고 있는 리듀서(reducer)를 수정하는 것을 기본으로 실행 취소를 구현하는 강력한 방법을 기술합니다. 심지어 [redux-undo](https://github.com/omnidan/redux-undo)라는 라이브러리도 제공합니다. 이 라이브러리는 개발자에게서 무거운 짐을 덜어주기 위해 고차원의 리듀서를 만들어줍니다. 하지만, 이 방법은 응용 프로그램의 이전 상태에 대한 '저장된' 레퍼런스(`past`)를 전달(overheard)받는 것입니다.

리덕스 사가의 `delay`와 `race`를 사용하면, 리듀서를 고도화하거나 이전 상태(state)를 저장하지 않고도 한번의 실행 취소를 간단하게 구현할 수 있습니다.

```javascript
import { take, put, call, spawn, race } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { updateThreadApi, actions } from 'somewhere'

function* onArchive(action) {

  const { threadId } = action
  const undoId = `UNDO_ARCHIVE_${threadId}`

  const thread = { id: threadId, archived: true }

  // 실행취소 UI 요소를 보여줍니다. 그리고 커뮤니케이션을 위한 키(key)를 제공합니다.
  yield put(actions.showUndo(undoId))

  // 낙관적으로, 쓰레드를 `archived`로 표시해둡니다.
  yield put(actions.updateThread(thread))

  // 사용자가 5초 동안 실행 취소를 수행할 수 있게 합니다.
  // 5초가 지나면, 'archive'가 race의 최종 승자가 됩니다.
  const { undo, archive } = yield race({
    undo: take(action => action.type === 'UNDO' && action.undoId === undoId),
    archive: call(delay, 5000)
  })

  // 실행 취소 UI 요소를 감춥니다. race의 최종 답안(answer)이 있을 것입니다. 
  yield put(actions.hideUndo(undoId))

  if (undo) {
    // 답안이 undo이면 쓰레드를 이전 상태로 되돌립니다.
    yield put(actions.updateThread({ id: threadId, archived: false }))
  } else if (archive) {
    // 답안이 archive이면, API를 호출하여 변경 사항을 원격으로 적용합니다.
    yield call(updateThreadApi, thread)
  }
}

function* main() {
  while (true) {
    // ARCHIVE_THREAD가 발생할때까지 기다립니다.
    const action = yield take('ARCHIVE_THREAD')
    // onArchive를 실행하기 위해 비차단(non-blocking) 방식으로 `spawn`을 사용합니다.
    // 이는 메인 사가가 취소되었을 때, onArchive도 함께 취소되는 것을 방지합니다.
    // 이는 서버와 클라이언트 간 상태(state)가 동일하게 유지되도록(동기화하도록) 돕습니다.
    yield spawn(onArchive, action)
  }
}
```
