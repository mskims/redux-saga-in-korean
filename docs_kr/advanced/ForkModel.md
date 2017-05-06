# redux-saga의 포크(fork) 모델

`redux-saga`에서는 두 개의 이펙트를 사용하여 태스크를 백그라운드에서 동적으로 실행시킬 수 있습니다.


- `fork`는 *결합된(attached)* 포크를 만들 때에 사용됩니다.
- `spawn`은 *분리된(detached)* 포크를 만들 때에 사용됩니다.

## 결합된 포크 (`fork` 사용)

결합된 포크들은 다음의 규칙에 따라 그들의 부모와 결합되어 있습니다.

### 완료

- 사가는 오직 다음 사건에만 종료됩니다.
  - 자신의 명령을 모두 이행한 뒤
  - 모든 결합된 포크들이 종료된 뒤

다음과 같은 예를 봅시다.

```js
import { delay } from 'redux-saga'
import { fork, call, put } from 'redux-saga/effects'
import api from './somewhere/api' // app specific
import { receiveData } from './somewhere/actions' // app specific

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield call(delay, 1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  yield call(fetchAll)
}
```

`call(fetchAll)`은 다음에 종료될 것입니다:

- `fetchAll` 의 내용(body)이 종료된 뒤 입니다. 이는 세 개의 이펙트들이 실행되었다는 것을 의미합니다. `fork` 이펙트는 비봉쇄이기 때문에, 태스크는 `call(delay, 1000)`에 봉쇄될 것입니다.

- 두 개의 포크된 태스크들이 종료된 뒤 입니다. 즉, 요구된 리소스들을 가져오고`receiveData` 액션을 전달한 뒤 입니다.

따라서 전체 태스크는 1000 밀리초가 지나고[1], `task1`과 `task2`가 그들의 일을 마친 뒤[2]에야 종료됩니다 ([1]과 [2]가 모두 만족해야 합니다).

예를 들어 1000 밀리초가 경과했지만 두 태스크가 아직 끝나지 않았을 때에는, `fetchAll`는 전체 태스크를 끝내기 전에 포크된 태스크가 끝날 때까지 기다릴 것입니다.

`fetchAll` 사가가 병렬 이펙트로 다음과 같이 작성될 수 있다는 것을 깨달은 분도 계실 것입니다:

```js
function* fetchAll() {
  yield [
    call(fetchResource, 'users'),     // task1
    call(fetchResource, 'comments'),  // task2,
    call(delay, 1000)
  ]
}
```

사실, 결합된 포크들은 병렬 이펙트와 같은 의미를 공유합니다:

- 병렬으로 태스크를 실행합니다.
- 부모는 그 안에서 실행된 모든 태스크가 종료된 뒤 종료될 것입니다. 

그리고 이것은 다른 모든 의미에도 적용됩니다 (에러와 취소 전달). 간단하게 *동적 병렬* 이펙트라고 생각하시면 결합된 포크의 동작을 이해하기 쉬울 것입니다.

## 에러 전달

같은 방법으로 에러가 병렬 이펙트에서 어떻게 다뤄지는지 자세히 검사해봅시다.

예를 들어, 이러한 이펙트가 있다고 가정해봅시다:

```js
yield [
  call(fetchResource, 'users'),
  call(fetchResource, 'comments'),
  call(delay, 1000)
]
```

위의 이펙트는 세 개의 자식 이펙트들 중 하나가 실패하자 마자 바로 실패할 것입니다. 더욱이, 예상치 못한 에러가 발생하면 병렬 이펙트는 모든 대기 중인 이펙트들을 취소할 것입니다. 따라서, 예를 들어 만약 `call(fetchResource, 'users')` 에서 예상치 못한 에러가 발생했다면, 병렬 이펙트는 두 개의 다른 태스크들을 취소할 것입니다 (만약 두 개의 태스크가 대기 중이라면). 그리고 자신 또한 실패한 태스크와 같은 에러로 취소될 것입니다.

결합된 포크와 비슷하게, 사가들은 다음에 즉시 취소됩니다:

- 자신의 내용이 에러를 throw할 때

- 결합된 포크에서 예상치 못한 에러가 발생했을 때

따라서 이전의 예제에서

```js
//... imports

function* fetchAll() {
  const task1 = yield fork(fetchResource, 'users')
  const task2 = yield fork(fetchResource, 'comments')
  yield call(delay, 1000)
}

function* fetchResource(resource) {
  const {data} = yield call(api.fetch, resource)
  yield put(receiveData(data))
}

function* main() {
  try {
    yield call(fetchAll)
  } catch (e) {
    // handle fetchAll errors
  }
}
```

이 때, `fetchAll`은 `call(delay, 1000)` 이펙트에서 봉쇄될 것이고, 만약 `task1`가 실패한다면, `fetchAll` 태스크는 다음을 수행하며 실패할 것입니다:

- 다른 대기 중인 모든 태스크들을 취소합니다. 이는 다음을 포함합니다:  
  - *메인 태스크* (`fetchAll`의 내용): 이를 취소하는 것은 현재 이펙트 `call(delay, 1000)`를 취소하는 것을 의미합니다.
  - 대기 중인 다른 태스크들. 즉, 우리 예제에서는 `task2`를 의미합니다.

- `call(fetchAll)`는 스스로 에러를 발생시켜 `main` 안에 있는 catch 구간에 잡힐 것입니다.

오직 `main` 안에서만 `call(fetchAll)`에서 발생하는 에러를 잡을 수 있다는 것에 주의하세요. 왜냐하면 우리는 봉쇄된 호출을 사용하고 있기 때문입니다. 우리는 `fetchAll` 안에서 바로 에러를 잡을 수 없습니다. 이는 경험에서 나온 법칙입니다. **포크된 태스크에서는 에러를 잡을 수 없습니다.** 결합된 포크가 내부에서 실패한다면, 포크한 부모가 이를 취소시킬 것입니다 (발생하는 에러를 병렬 이펙트 *안에서* 잡을 수 없는 것처럼 말입니다).


## 취소

사가를 취소하는 것은 다음을 야기합니다:

- *메인 태스크*의 취소 (이는 봉쇄된 현재 이펙트를 취소하는 것을 의미합니다).

- 실행 중인 모든 결합된 포크들의 취소


**개발 중**

## 분리된 포크 (`spawn` 사용)

분리된 포크들은 그들 스스로 살아갑니다. 부모는 분리된 포크가 종료되는 것을 기다리지 않습니다. 예상치 못한 에러는 부모로 전달되지 않습니다. 그리고 부모를 취소한다고 해도 분리된 포크는 자동으로 취소되지 않습니다 (분리된 포크는 명시적으로 취소해야 합니다).

즉, 분리된 포크들은 `middleware.run` API를 이용한 최상위 사가(root saga)처럼 행동합니다.


**개발 중**
