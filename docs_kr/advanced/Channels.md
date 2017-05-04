# 채널

지금까지 우리는 `take` 과 `put` 이펙트를 사용해서 리덕스 스토어와 통신했습니다. 채널은 외부의 이벤트 소스 또는 사가 간 통신을 위해 해당 이펙트를 일반화합니다. 또한 스토어에서 특정 작업을 대기열(queue)에 넣을 때에도 사용할 수 있습니다.

이 장에서, 우리는 다음을 살펴볼 것입니다:

- `yield actionChannel` 이펙트를 이용해 스토어의 특정 액션을 버퍼링하는 방법

- `eventChannel` 팩토리 함수를 사용하여 `take` 이펙트를 외부 이벤트 소스에 연결하는 방법

- 일반 `channel` 팩토리 함수를 이용하여 채널을 만드는 방법과 사가 간의 통신을 위해 `take`/`put` 이펙트에 이를 사용하는 방법

## `actionChannel` 이펙트 사용

기본 예제를 다시 볼까요?

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

위의 예제는 전형적인 *watch 와 fork* 패턴입니다. `watchRequests` 사가는 봉쇄를 피하고 스토어에서 어떤 액션도 놓치지 않기 위해서 `fork`를 사용하고 있습니다. `handleRequest` 태스크는 각 `REQUEST` 액션에서 생성됩니다. 따라서 짧은 시간에 많은 액션이 들어온다면 동시에 많은 `handleRequest` 태스크가 실행될 수 있겠죠.

이제 우리의 요구 사항은 다음과 같습니다: 우리는 `REQUEST`을 순차적으로 처리하려고 합니다. 우리가 어떤 순간에 네 개의 액션을 가지고 있다면, 우리는 첫 번째 액션을 먼저 처리하고, 두 번째 액션을 처리하는 등 액션을 순차적으로 처리하려고 합니다.

그래서 우리는 아직 처리되지 않은 액션을 대기열에 집어넣을 겁니다. 그리고 현재 요청을 마쳤다면, 대기열에서 다음 것을 가져올 겁니다.

Redux-Saga는 작은 헬퍼 이펙트 `actionChannel`를 제공합니다. 이는 위에서 말한 것들을 다룰 수 있습니다. 이제 위의 예제를 어떻게 바꿀 수 있는지 봅시다.

```javascript
import { take, actionChannel, call, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 1- Create a channel for request actions
  const requestChan = yield actionChannel('REQUEST')
  while (true) {
    // 2- take from the channel
    const {payload} = yield take(requestChan)
    // 3- Note that we're using a blocking call
    yield call(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

첫 번째는 액션 채널을 만드는 것입니다. 우리는 이전에 말했던 `take(pattern)`처럼, 같은 규칙을 사용하여 해석되는 패턴이 있는 곳에 `yield actionChannel(pattern)`를 사용합니다. 둘의 차이점은, 사가가 아직 그들을 처리할 준비가 되지 않았다면 (예를 들어, API 호출에 봉쇄됨) `actionChannel`은 들어오는 메시지를 버퍼링할 수 있다는 것입니다.

다음은 `yield take(requestChan)`을 봅시다. 스토어에서 특정 액션을 받기 위해 패턴을 사용한 것처럼, `take`는 채널과 같이 쓰일 수도 있습니다 (위에서 우리는 특정 액션으로부터 채널 객체를 만들었습니다). `take`는 메시지를 받을 수 있을 때에만 사가를 봉쇄할 것입니다. `take`는 버퍼에 메시지가 저장되어 있을 경우에만 봉쇄되지 않고 진행할 것입니다.

우리가 봉쇄하는 `call`을 어떻게 사용하고 있는지 주목하세요. 사가는 `call(handleRequest)`가 반환할 때까지 봉쇄를 유지할 것입니다. 하지만 봉쇄되어 있는 중에 다른 `REQUEST` 액션이 dispatch된다면, 그것은 `requestChan`에 의해 내부적으로 대기열에 저장될 것입니다. 사가가 `call(handleRequest)`에 의해 재개되고 다음 `yield take(requestChan)`이 실행될 때, `take`는 대기열에 저장된 메시지를 resolve할 것입니다.

기본적으로 `actionChannel`은 제한 없이 버퍼링이 가능합니다. 버퍼링에 대해서 더 섬세히 제어하고 싶다면, 이펙트 생성자에 버퍼 인자를 줄 수 있습니다. Redux-Saga는 일반 버퍼(none, dropping, sliding)를 제공하지만 직접 추가해서 사용해도 됩니다. [API 문서](../api#buffers)를 참조하세요.

예를 들어 최근 다섯 개의 아이템만 관리하고 싶다면 다음과 같이 쓸 수 있습니다:

```javascript
import { buffers } from 'redux-saga'
import { actionChannel } from 'redux-saga/effects'

function* watchRequests() {
  const requestChan = yield actionChannel('REQUEST', buffers.sliding(5))
  ...
}
```

## `eventChannel` 팩토리를 사용해 외부 이벤트에 연결하기

`actionChannel` 이펙트처럼, `eventChannel` (이펙트가 아닌 팩토리 함수)는 리덕스 스토어가 아닌 외부 이벤트를 위한 채널을 생성합니다.

이 예제는 일정한 간격마다 채널을 생성합니다:

```javascript
import { eventChannel, END } from 'redux-saga'

function countdown(secs) {
  return eventChannel(emitter => {
      const iv = setInterval(() => {
        secs -= 1
        if (secs > 0) {
          emitter(secs)
        } else {
          // this causes the channel to close
          emitter(END)
        }
      }, 1000);
      // The subscriber must return an unsubscribe function
      return () => {
        clearInterval(iv)
      }
    }
  )
}
```

`eventChannel`의 첫 번째 인자는 *구독자(subscriber)* 함수입니다. 구독자의 역할은 외부의 이벤트 소스를 초기화하고 (위의 `setInterval` 사용), 제공된 `emitter`를 실행하여 소스에서 채널로 들어오는 모든 이벤트를 라우팅합니다. 위의 예제에서 우리는 매 초마다 `emitter`를 호출합니다.

> 주의: 이벤트 채널을 통해 null 또는 undefined를 전달하지 않도록 해야합니다. 숫자를 전달하는 것이 좋지만, 이벤트 채널 데이터를 리덕스 액션처럼 구조화하는 것을 추천합니다. `number`를 `{ number }`로 바꾸는 것처럼요.

`emitter(END)` 호출에도 주의하세요. 우리는 채널 소비자에게 채널이 폐쇄되었다는 것을 알리기 위해 사용합니다. 이는 더 이상 다른 메시지가 이 채널을 통해 올 수 없다는 것을 의미합니다.

우리의 사가에서 이 채널을 어떻게 쓰는지 봅시다. (이 예제는 저장소(repository)의 cancellable-counter 예제에서 가져왔습니다.)

```javascript
import { take, put, call } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// creates an event Channel from an interval of seconds
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {    
    while (true) {
      // take(END) will cause the saga to terminate by jumping to the finally block
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    console.log('countdown terminated')
  }
}
```

사가는 `take(chan)`를 yield하고 있습니다. 메시지가 채널에 들어가기 전까지 사가는 봉쇄됩니다. 위의 예제에서, 이는 `emitter(secs)`를 호출할 때와 일치합니다. 우리가 `try/finally` 구역 내에서 전체 `while (true) {...}`를 실행하고 있는 것에 주목하세요. countdown의 interval이 종료되면, countdown 함수는 `emitter(END)`를 호출함으로써 이벤트 채널을 폐쇄합니다. 채널을 닫으면 그 채널에서 `take`에 봉쇄된 모든 사가들을 종료시키는 효과가 있습니다. 예제에서, 사가를 종료하면 `finally` 구간으로 점프하게 됩니다 (`finally` 구간이 없으면 그냥 종료됩니다).

구독자는 `unsubscribe` 함수를 반환합니다. 이것은 이벤트 소스가 완료되기 전에 채널 구독을 취소하는 데에 사용됩니다. 이벤트 채널의 메시지를 소비하는 사가 내에서 이벤트 소스가 완료되기 전에 *일찍 나가기*를 원한다면 (예로, 사가가 취소됨) `chan.close()`를 호출해 채널을 폐쇄하고 구독을 취소할 수 있습니다.

예를 들어 우리는 사가가 취소를 지원하도록 만들 수 있습니다.

```javascript
import { take, put, call, cancelled } from 'redux-saga/effects'
import { eventChannel, END } from 'redux-saga'

// creates an event Channel from an interval of seconds
function countdown(seconds) { ... }

export function* saga() {
  const chan = yield call(countdown, value)
  try {    
    while (true) {
      let seconds = yield take(chan)
      console.log(`countdown: ${seconds}`)
    }
  } finally {
    if (yield cancelled()) {
      chan.close()
      console.log('countdown cancelled')
    }    
  }
}
```

여기 웹 소켓 이벤트를 사가에 전달하여 이벤트 채널을 사용하는 방법을 다룬 또 다른 예제가 있습니다 (예: socket.io 라이브러리 사용). `ping`이라는 서버 메시지를 기다리고 있고, 조금 뒤에 `pong`이라는 메시지로 답한다고 가정해봅시다.


```javascript
import { take, put, call, apply } from 'redux-saga/effects'
import { eventChannel, delay } from 'redux-saga'
import { createWebSocketConnection } from './socketConnection'

// this function creates an event channel from a given socket
// Setup subscription to incoming `ping` events
function createSocketChannel(socket) {
  // `eventChannel` takes a subscriber function
  // the subscriber function takes an `emit` argument to put messages onto the channel
  return eventChannel(emit => {

    const pingHandler = (event) => {
      // puts event payload into the channel
      // this allows a Saga to take this payload from the returned channel
      emit(event.payload)
    }

    // setup the subscription
    socket.on('ping', pingHandler)

    // the subscriber must return an unsubscribe function
    // this will be invoked when the saga calls `channel.close` method
    const unsubscribe = () => {
      socket.off('ping', pingHandler)
    }

    return unsubscribe
  })
}

// reply with a `pong` message by invoking `socket.emit('pong')`
function* pong(socket) {
  yield call(delay, 5000)
  yield apply(socket, socket.emit, ['pong']) // call `emit` as a method with `socket` as context
}

export function* watchOnPings() {
  const socket = yield call(createWebSocketConnection)
  const socketChannel = yield call(createSocketChannel, socket)

  while (true) {
    const payload = yield take(socketChannel)
    yield put({ type: INCOMING_PONG_PAYLOAD, payload })
    yield fork(pong, socket)
  }
}
```

> 주의: eventChannel의 메시지는 기본적으로 버퍼링되지 않습니다. 채널의 버퍼링 전략을 지정하려면 eventChannel 팩토리에 버퍼를 인수로 넣어줘야 합니다 (예: `eventChannel(subscriber, buffer)`). 자세한 사항은 [API 문서](../api#buffers)를 참조하세요.

### 사가 간 통신에 채널 사용하기

액션 채널과 이벤트 채널 외에도 기본적으로 어떤 소스에도 연결되지 않은 채널을 직접 생성할 수 있습니다. 그런 다음 채널에 수동으로 `put` 할 수 있습니다. 이는 사가 간에 통신을 하기 위해 채널을 사용할 때 유용합니다.

설명을 위해서 요청 처리 예를 다시 봅시다.

```javascript
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  while (true) {
    const {payload} = yield take('REQUEST')
    yield fork(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

우리는 'watch 와 fork' 패턴이 동시적으로 실행되는 태스크의 갯수에 제한 없이 동시에 다수의 요청을 처리할 수 있다는 것을 보았죠. 그리고 우리는 `actionChannel` 이펙트를 이용해서 한 번에 하나의 태스크만 실행되도록 제한해보았습니다.

우리는 최대 세 개의 태스크를 한번에 실행해보려고 합니다. 우리가 세 개보다 적은 태스크를 실행하고 있다면, 요청을 즉시 실행하겠죠. 그게 아니라면 세 개의 *슬롯* 중에 하나의 태스크가 끝나기 전까지 대기열에 요청을 집어넣을 것입니다.

아래는 채널을 이용한 해법의 예제입니다:

```javascript
import { channel } from 'redux-saga'
import { take, fork, ... } from 'redux-saga/effects'

function* watchRequests() {
  // create a channel to queue incoming requests
  const chan = yield call(channel)

  // create 3 worker 'threads'
  for (var i = 0; i < 3; i++) {
    yield fork(handleRequest, chan)
  }

  while (true) {
    const {payload} = yield take('REQUEST')
    yield put(chan, payload)
  }
}

function* handleRequest(chan) {
  while (true) {
    const payload = yield take(chan)
    // process the request
  }
}
```

위의 예제에서 우리는 `channel` 팩토리로 채널을 만들었습니다. 우리는 기본적으로 채널에 있는 모든 메시지를 버퍼링합니다 (보류 중인 take가 없는 한 메시지와 함께 즉시 재개됩니다).

그런 다음 `watchRequests` 사가는 세 개의 워커 사가를 `fork`합니다. 생선된 채널이 fork된 모든 사가에 인수로 들어간 것을 보시기 바랍니다. `watchRequests`는 세 개의 워커 사가들에게 일을 dispatch하기 위해서 채널을 사용할 것입니다. 각 `REQUEST` 액션 마다 사가는 채널에 `payload`를 `put`합니다. 그러면 이는 *할 일이 없는* 워커 사가에게 전달될 것입니다. 그게 아니라면 이는 워커 사가가 가져갈 준비가 될 때까지 채널에 의해 대기열에 넣어질 것입니다.

모든 세 개의 워커가 전형적인 while 반복으로 실행됩니다. 각 반복 마다, 워커는 다음 요청(`REQUEST` 액션)을 가져가거나, 요청으로 만들어진 메시지가 있을 때까지 봉쇄될 것입니다. 이 메커니즘이 세 개의 워커 간에 자동 로드밸런싱을 해주게 됩니다. 빠르게 일을 처리한 워커는 느리게 처리중인 워커 때문에 같이 느려지는 일이 없습니다.