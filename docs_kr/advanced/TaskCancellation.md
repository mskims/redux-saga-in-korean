# 태스크 취소

우리는 이미 [비봉쇄(non-blocking) 호출](NonBlockingCalls.md) 장에서 태스크 취소의 예를 본 적이 있습니다. 이 장에서 우리는 더 자세히 태스크 취소에 관해서 알아볼겁니다.

태스크가 fork되었다면, `yield cancel(task)`을 사용해서 무산시킬 수 있습니다.

이 과정이 어떻게 작동하는지 보기 위해서 간단한 예제를 봅시다: UI 명령으로 백그라운드 싱크를 시작/중지할 수 있는 예제입니다. `START_BACKGROUND_SYNC` 액션을 받으면, 우리는 원격 서버에서 데이터를 싱크하는 백그라운드 태스크를 fork합니다.

태스크는 `STOP_BACKGROUND_SYNC` 액션이 발동할 때까지 실행될 겁니다. 그러면 우리는 백그라운드 태스크를 취소하고 다음 `START_BACKGROUND_SYNC` 액션을 다시 기다립니다.

```javascript
import { take, put, call, fork, cancel, cancelled } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { someApi, actions } from 'somewhere'

function* bgSync() {
  try {
    while (true) {
      yield put(actions.requestStart())
      const result = yield call(someApi)
      yield put(actions.requestSuccess(result))
      yield call(delay, 5000)
    }
  } finally {
    if (yield cancelled())
      yield put(actions.requestFailure('Sync cancelled!'))
  }
}

function* main() {
  while ( yield take(START_BACKGROUND_SYNC) ) {
    // starts the task in the background
    const bgSyncTask = yield fork(bgSync)

    // wait for the user stop action
    yield take(STOP_BACKGROUND_SYNC)
    // user clicked stop. cancel the background task
    // this will cause the forked bgSync task to jump into its finally block
    yield cancel(bgSyncTask)
  }
}
```

위의 예제에서, `bgSyncTask`의 취소는 제너레이터를 finally 구간으로 점프시킬 것입니다. 여기서 제너레이터가 취소되었는지 아닌지를 확인하기 위해서 `yield cancelled()`를 사용할 수 있습니다.

실행중인 태스크를 취소하면, 취소의 순간에 봉쇄(blocked)된 태스크가 있는 현재 이펙트 또한 취소합니다.

예를 들어, 어플리케이션의 작동되는 동안 어떤 지점에서 이러한 호출 사슬(chain)이 있다고 가정해 봅시다:

```javascript
function* main() {
  const task = yield fork(subtask)
  ...
  // later
  yield cancel(task)
}

function* subtask() {
  ...
  yield call(subtask2) // currently blocked on this call
  ...
}

function* subtask2() {
  ...
  yield call(someApi) // currently blocked on this call
  ...
}
```

`yield cancel(task)`는 `subtask`를 취소합니다. 그리고 이것은 차례로 `subtask2`를 취소합니다.

이렇게 우리는 취소가 아래로 전해지는 것을 보았습니다 (값 반환 또는 예상하지 못한 에러가 위로 전해지는 것과는 반대입니다). 당신은 호출자(비동기 작업을 실행함) 와 피호출자(실행된 작업) 사이의 *계약*을 보았습니다. 피호출자는 작업을 실행할 책임이 있습니다. 만약 그것이 완료된다면 (성공 혹은 에러), 결과는 계속해서 호출자에서 호출자로 전달될 것입니다. 즉, 피호출자는 *플로우를 완료하는 것*에 책임이 있습니다.

만약 피호출자가 여전히 대기 중이고 호출자가 작업을 취소하기로 결정했다면, 그것은 일종의 신호를 피호출자에게 밑으로 전달합니다 (또한 피호출자에게 호출당한 얼마나 깊은 작업일지라도 전달하겠죠). 깊게 대기중인 작업들은 모두 취소될 것입니다.

취소를 다른 방향으로 전달할 수도 있습니다: 태스크의 병합자(joiner) (`yield join(task)`에 의해 봉쇄된 태스크)는 병합된(joined) 태스크가 취소되면 같이 취소될 것입니다. 비슷하게, 병합자들의 잠재적인 호출자 또한 취소될 것입니다 (왜냐하면, 호출자들은 바깥에서 취소된 작업에 봉쇄되어 있기 때문입니다).

## fork 이펙트로 제너레이터 테스트하기

`fork`가 호출되면 태스크는 백그라운드에서 시작하고, 전에 배웠던 것처럼 태스크를 반환합니다. 이를 테스트할 때, 우리는 유틸리티 함수 `createMockTask`를 사용해야 합니다. 이 함수에서 반환되는 객체는 fork 테스트 후에 `next`의 인자로 주어져야 합니다. 그렇게 해야 가짜(mock) 태스크가 `cancel`로 이어질 수 있습니다. 여기 이 페이지의 맨 위에 있는 `main` 제너레이터의 테스트 코드입니다.

```javascript
import { createMockTask } from 'redux-saga/utils';

describe('main', () => {
  const generator = main();

  it('waits for start action', () => {
    const expectedYield = take(START_BACKGROUND_SYNC);
    expect(generator.next().value).to.deep.equal(expectedYield);
  });

  it('forks the service', () => {
    const expectedYield = fork(bgSync);
    expect(generator.next().value).to.deep.equal(expectedYield);
  });

  it('waits for stop action and then cancels the service', () => {
    const mockTask = createMockTask();

    const expectedTakeYield = take(STOP_BACKGROUND_SYNC);
    expect(generator.next(mockTask).value).to.deep.equal(expectedTakeYield);

    const expectedCancelYield = cancel(mockTask);
    expect(generator.next().value).to.deep.equal(expectedCancelYield);
  });
});
```

또한 가짜 태스크의 상태를 설정하기 위해 `setRunning`, `setResult`, `setError` 같은 가짜 태스크의 함수도 사용할 수 있습니다. 예를 들어 `mockTask.setRunning(false)`.

### 주의

`yield cancel(task)`는 태스크가 끝날 때까지 기다리지 않는다는 중요한 사실을 기억하세요 (즉, finally 블록을 기다리지 않습니다). `cancel` 이펙트는 `fork` 이펙트처럼 행동합니다. 이것은 취소가 시작된 직후에 반환합니다. 취소가 되면, 태스크는 보통 청소 로직(finally 로직)을 끝내자 마자 반환해야 합니다.

## 자동 취소

직접 작성하는 취소 작업 말고도, 취소가 자동으로 작동되는 경우가 있습니다.

1. `race` 이펙트에서, 우승자를 제외한 모든 경주 경쟁자들은 자동으로 취소됩니다.

2. 병렬 이펙트(`yield [...]`)에서 하나의 서브 이펙트들이 reject된다면, 그 병렬 이펙트는 reject됩니다 (`Promise.all` 처럼요). 이 경우, 모든 서브 이펙트들은 자동으로 취소됩니다.
