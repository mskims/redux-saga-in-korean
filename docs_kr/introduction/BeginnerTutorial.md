# 튜토리얼

## 목적

이 튜토리얼은 redux-saga 를 가능한 쉬운 방법으로 소개할것입니다.
<!--This tutorial attempts to introduce redux-saga in a (hopefully) accessible way.-->

튜토리얼을 위해서, 우리는 간단한 Redux 저장소에 있는 간단한 카운터 예시를 사용할겁니다.
이 카운터 애플리케이션은 아주 간단하면서, 과도하게 빠지지 않고 redux-sage 의 기본 컨셉들을 설명 하기에 딱입니다.
<!--For our getting started tutorial, we are going to use the trivial Counter demo from the Redux repo. The application is quite simple but is a good fit to illustrate the basic concepts of redux-saga without being lost in excessive details.-->

### 초기 설정

시작하기 전에, [튜토리얼 저장소](https://github.com/redux-saga/redux-saga-beginner-tutorial) 를 클론 하세요.
<!--Before we start, clone the [tutorial repository](https://github.com/redux-saga/redux-saga-beginner-tutorial).-->

> 이 튜토리얼의 코드들은 `sagas` 브랜치에 있습니다.

<!--The final code of this tutorial is located in the `sagas` branch.-->

커맨드 라인에서 다음 명령어를 실행하세요:
<!--Then in the command line, run:-->

```sh
$ cd redux-saga-beginner-tutorial
$ npm install
```

애플리케이션을 시작하기 위해서는 다음 명령어를 실행하시면 됩니다:
<!--To start the application, run:-->

```sh
$ npm start
```

우리는 `증가` 와 `감소` 버튼이 있는 카운터로 아주 간단하게 시작하고, 그후 비동기 호출에 대해서 설명하겠습니다
<!--We are starting with the simplest use case: 2 buttons to `Increment` and `Decrement` a counter. Later, we will introduce asynchronous calls.-->

이상이 없다면, 당신은 `증가` 와 `감소` 버튼과 `Counter: 0` 이라는 메세지를 볼 수 있을것 입니다.
<!--If things go well, you should see 2 buttons `Increment` and `Decrement` along with a message below showing `Counter: 0`.-->

> 만약 이 단계에서 어려움을 겪고계시다면, 고민하지 마시고 [튜토리얼 저장소](https://github.com/redux-saga/redux-saga-beginner-tutorial/issues) 에 에슈를 만들어주세요.

<!-- > In case you encountered an issue with running the application. Feel free to create an issue on the [tutorial repo](https://github.com/redux-saga/redux-saga-beginner-tutorial/issues).-->

## Hello Sagas!

첫번째 Saga 를 만들어봅시다! 전통을 따라, Saga 버전 'Hello, world' 를 작성해 봅시다.
<!--We are going to create our first Saga. Following the tradition, we will write our 'Hello, world' version for Sagas.-->

`sagas.js` 파일을 만드신 후 다음 내용을 적으세요.
<!--Create a file `sagas.js` then add the following snippet:-->

```javascript
export function* helloSaga() {
  console.log('Hello Sagas!')
}
```

무서운것이 없습니다, 이건 그냥 평범한 함수일 뿐이에요. (`*`를 제외하면요). 이것이 하는일은 콘솔에 환영 메세지를 적는것밖에 없습니다.
<!--So nothing scary, just a normal function (except for the `*`). All it does is print a greeting message into the console.-->

우리의 Saga 를 실행하기 위해서, 몇가지 할 일이 있습니다.
<!--In order to run our Saga, we need to:-->

- Sagas 리스트와 함께 Saga 미들웨어를 만드세요. (지금까진 `helloSaga` 오직 하나입니다)
- Saga 미들웨어를 Redux 스토어에 연결하세요.

<!--- create a Saga middleware with a list of Sagas to run (so far we have only one `helloSaga`)-->
<!--- connect the Saga middleware to the Redux store-->

이제 `main.js` 를 작성해봅시다:
<!--We will make the changes to `main.js`:-->

```javascript
// ...
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

// ...
import { helloSaga } from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(helloSaga)

const action = type => store.dispatch({type})

// rest unchanged
```

처음에, `./sagas` 모듈에서 가져온 우리의 Saga 를 임포트 합니다. 그리고 나서 `redux-saga` 라이브러리에서 가져온 `createSagaMiddleware` 팩토리 함수를 사용해서 미들웨어를 만들었죠.
<!--First we import our Saga from the `./sagas` module. Then we create a middleware using the factory function `createSagaMiddleware` exported by the `redux-saga` library.-->

`helloSaga` 를 실행하기 전에, 반드시 `applyMiddleware` 를 사용해서 미들웨어를 연결해야 `sagaMiddleware.run(helloSaga)` 를 통해 Saga 를 시작할 수 있습니다.. 
<!--Before running our `helloSaga`, we must connect our middleware to the Store using `applyMiddleware`. Then we can use the `sagaMiddleware.run(helloSaga)` to start our Saga.-->

지금까지 우리의 Saga 는 특별하지 않습니다. 이건 단지 로그 메세지만을 남기고 종료될 뿐입니다.
<!--So far, our Saga does nothing special. It just logs a message then exits.-->

## 비동기 호출

이제, 오리지널 카운터 데모 가까이 무언가를 추가해봅시다. 비동기 호출을 설명하기 위해 클릭 1초 후 증가되는 또다른 버튼을 추가할겁니다.
<!--Now let's add something closer to the original Counter demo. To illustrate asynchronous calls, we will add another button to increment the counter 1 second after the click.-->

먼저, UI 컴포넌트에 `onIncrementAsync` 라는 콜백을 넣을겁니다.
<!--First thing's first, we'll provide an additional callback `onIncrementAsync` to the UI component.-->

```javascript
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
  <div>
    {' '}
    <button onClick={onIncrementAsync}>
      Increment after 1 second
    </button>
    <hr />
    <div>
      Clicked: {value} times
    </div>
  </div>
```

다음으로, `onIncrementAsync` 를 스토어 액션에 연결해야 합니다.
<!--Next we should connect the `onIncrementAsync` of the Component to a Store action.-->

`main.js` 모듈을 다음과 같이 수정하겠습니다.
<!--We will modify the `main.js` module as follows-->

```javascript
function render() {
  ReactDOM.render(
    <Counter
      value={store.getState()}
      onIncrement={() => action('INCREMENT')}
      onDecrement={() => action('DECREMENT')} 
      onIncrementAsync={() => action('INCREMENT_ASYNC')} />,
    document.getElementById('root')
  )
}
```
명심하세요, redux-thunk 와는 달리 우리의 컴포넌트는 순수 액션 오브젝트만 dispatch 할겁니다.
<!--Note that unlike in redux-thunk, our component dispatches a plain object action.-->

이제 비동기 호출을 구현하기 위해 또다른 Saga 를 소개해볼까 합니다. 
<!--Now we will introduce another Saga to perform the asynchronous call. Our use case is as follows:-->

> 각각의 `INCREMENT_ASYNC` 액션에서, 우리는 다음과 같은 작업을 수행할 태스크를 시작하고자 합니다. 

<!--> On each `INCREMENT_ASYNC` action, we want to start a task that will do the following-->

> - 1 초를 기다린 후 카운터 증가

<!-- > - Wait 1 second then increment the counter-->

다음 코드를 `sagas.js` 모듈에 추가하세요.
<!--Add the following code to the `sagas.js` module:-->

```javascript
import { delay } from 'redux-saga'
import { put, takeEvery } from 'redux-saga/effects'

// worker Saga: 비동기 증가 태스크를 수행할겁니다.
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// watcher Saga: 각각의 INCREMENT_ASYNC 에 incrementAsync 태스크를 생성할겁니다.
export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

설명할 때가 왔군요.
<!--Time for some explanations.-->

`delay` 라는 유틸리티 함수를 임포트 했는데요, 이 함수는 설정된 시간 이후에 resolve 를 하는 [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) 객체를 리턴합니다. 우리는 이 함수를 제너레이터를 *정지* 하는데 사용할겁니다.

<!--We import `delay`, a utility function that returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will resolve after a specified number of milliseconds. We'll use this function to *block* the Generator.-->

Sagas 는 오브젝트들을 redux-saga 미들웨어에 *yield* 하는 [제너레이터 함수](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) 로 구현되었습니다. *yield된* 오브젝트들은 미들웨어에 의해 해석되는 명령의 한 종류입니다. Promise 가 미들웨어에 yield 될 때, 미들웨어는 Promise 가 끝날때 까지 Saga 를 일시정지 시킬것 입니다. 위의 예시에서, `incrementAsync` Saga 는 1초 후에 일어날 `delay`의 resolve 에 의해 Promise 가 리턴될때 까지 정지되어있을겁니다.

<!--Sagas are implemented as [Generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) that *yield* objects to the redux-saga middleware. The yielded objects are a kind of instruction to be interpreted by the middleware. When a Promise is yielded to the middleware, the middleware will suspend the Saga until the Promise completes. In the above example, the `incrementAsync` Saga is suspended until the Promise returned by `delay` resolves, which will happen after 1 second.-->

Promise 가 한번 resolve 되고 나면, 미들웨어는 Saga 를 다시 작동시키면서, 다음 yield 까지 코드를 실행합니다. 이 예제에서 다음 상태는 미들웨어에게 `INCREMENT` 액션을 dispach 하게 알려주는  `put({type: 'INCREMENT'})` 의 결과 객체가 됩니다.

<!--Once the Promise is resolved, the middleware will resume the Saga, executing code until the next yield. In this example, the next statement is another yielded object: the result of calling `put({type: 'INCREMENT'})`, which instructs the middleware to dispatch an `INCREMENT` action.-->

`put` 은 우리가 *이펙트* 라고 부르는 예중 하나입니다. 이펙트는 미들웨어에 의해 수행되는 지시를 담고있는 간단한 자바스크립트 객체입니다. 미들웨어가 Saga 에 의해 yield 된 이펙트를 받을때, Saga 는 이펙트가 수행될때까지 정지되어 있을겁니다.

<!--`put` is one example of what we call an *Effect*. Effects are simple JavaScript objects which contain instructions to be fulfilled by the middleware. When a middleware retrieves an Effect yielded by a Saga, the Saga is paused until the Effect is fulfilled.-->

정리하자면, `incrementAsync` Saga 는 `delay(1000)` 호출에 의해 1초동안 자고있다가, `INCREMENT` 액션을 dispatch 하게 되는것이죠.
<!--So to summarize, the `incrementAsync` Saga sleeps for 1 second via the call to `delay(1000)`, then dispatches an `INCREMENT` action.-->

다음으로, 우리는 `watchIncrementAsync` 라는 또다른 Saga를 만들었습니다. dispatch된 `INCREMENT_ASYNC` 액션을 바라보고, 매번 `incrementAsync` 를 실행하기 위해 `redux-saga` 패키지가 제공하는 `takeEvery` 헬퍼 함수를 사용했습니다.
<!--Next, we created another Saga `watchIncrementAsync`. We use `takeEvery`, a helper function provided by `redux-saga`, to listen for dispatched `INCREMENT_ASYNC` actions and run `incrementAsync` each time.-->

이제 두개의 Saga가 있네요, 이제 두 Saga 모두 한번에 실행해야할 필요가 생겼습니다, 이 작업을 하려면, 다른 Saga들을 시작해야할 책임이 있는 `rootSaga` 를 추가해봅시다.

자 이제 여기 코드들을 `sagas.js` 에 추가해보세요.

<!--Now we have 2 Sagas, and we need to start them both at once. To do that, we'll add a `rootSaga` that is responsible for starting our other Sagas. In the same file `sagas.js`, add the following code:-->

```javascript
// 모든 Saga들을 한번에 시작하기 위한 하나의 지점입니다.
export default function* rootSaga() {
  yield [
    incrementAsync(),
    watchIncrementAsync()
  ]
}
```

이 Saga는 `helloSaga` Saga 와 `watchIncrementAsync` Saga 가 호출된 결과의 배열을 yield 합니다. 이것은 생선된 두 제너레이터가 병렬로 시작된다는것을 의미하죠. 이제 `sagaMiddleware.run` 를 `main.js` 의 root Saga에 주입할 일만 남았습니다.

<!--This Saga yields an array with the results of calling our two sagas, `helloSaga` and `watchIncrementAsync`. This means the two resulting Generators will be started in parallel. Now we only have to invoke `sagaMiddleware.run` on the root Saga in `main.js`.-->

```javascript
// ...
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = ...
sagaMiddleware.run(rootSaga)

// ...
```

## 테스트

이제 우리의 `incrementAsync` Saga 가 바람직한 태스크를 수행하는지 확실하게 해야겠죠? 테스트를 만들어 봅시다.
<!--We want to test our `incrementAsync` Saga to make sure it performs the desired task.-->

`sagas.spec.js` 파일을 만듭시다.
<!--Create another file `sagas.spec.js`:-->

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

`incrementAsync` 는 제너레이터 함수입니다. 이것을 실행하면, 이터레이터 오브젝트를 반환하고, 이터레이터의 `next` 메소드는 다음과 같은 모양을 가진 객체를 돌려줍니다. 
<!--`incrementAsync` is a generator function. When run, it returns an iterator object, and the iterator's `next` method returns an object with the following shape-->

```javascript
gen.next() // => { done: boolean, value: any }
```

`value` 필드는 yield 된 표현식을 포함합니다. `yield` 다음 표현식의 결과 같은 것 말이죠.
`done` 필드는 아직 `yield` 표현이 남아있는지, 아니면 제너레이터가 종료되었는지 가리킵니다.
<!--The `value` field contains the yielded expression, i.e. the result of the expression after
the `yield`. The `done` field indicates if the generator has terminated or if there are still
more 'yield' expressions.-->


`incrementAsync` 로 예를 들자면, 제너레이터는 두개의 값을 연속으로 yield 합니다.
<!--In the case of `incrementAsync`, the generator yields 2 values consecutively:-->

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`


그래서 우리가 제너레이터의 next 메소드를 세번 연속하여 부른다면, 다음과 같은 결과값을 얻게 됩니다.
<!--So if we invoke the next method of the generator 3 times consecutively we get the following
results:-->

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

처음 두개 호출은 yield 표현의 결과를 돌려줍니다. 3번째 호출은 더이상 yield 가 없기 때문에 `done` 필드는 true 로 설정되고, `incrementAsync` 제너레이터가 아무것도 리턴하지 않기 때문에 `value` 필드는 `undefined` 로 설정됩니다.
<!--The first 2 invocations return the results of the yield expressions. On the 3rd invocation
since there is no more yield the `done` field is set to true. And since the `incrementAsync`
Generator doesn't return anything (no `return` statement), the `value` field is set to
`undefined`.-->


자 이제, `incrementAsync` 안에서 로직을 테스트하기 위해, 돌려받은 제너레이터를 반복하고, 제너레이터에 의해 yield 된 값들을 간단히 체크할겁니다.
<!--So now, in order to test the logic inside `incrementAsync`, we'll simply have to iterate
over the returned Generator and check the values yielded by the generator.-->

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next(),
    { done: false, value: ??? },
    'incrementAsync should return a Promise that will resolve after 1 second'
  )
});
```

하지만 Promise 에선 비교연산을 할 수 없는데 어떻게 `delay` 의 리턴값을 테스트 하죠?  만약 `delay` 가 *평범한* 값을 돌려준다면 테스트하기 쉬울텐데요..
<!--The issue is how do we test the return value of `delay`? We can't do a simple equality test
on Promises. If `delay` returned a *normal* value, things would've been easier to test.-->

`redux-saga` 는 위의 고민을 해결할 방법을 제시하고 있습니다. `incrementAsync` 에서 `delay(1000)` 을 직접적으로 호출하는것 대신, 우린 *간접적으로* 호출할겁니다.
<!--Well, `redux-saga` provides a way to make the above statement possible. Instead of calling
`delay(1000)` directly inside `incrementAsync`, we'll call it *indirectly*:-->

```javascript
// ...
import { delay } from 'redux-saga'
import { put, call, takeEvery } from 'redux-saga/effects'

export function* incrementAsync() {
  // use the call Effect
  yield call(delay, 1000)
  yield put({ type: 'INCREMENT' })
}
```

`yield delay(1000)` 대신 `yield call(delay, 1000)` 를 하고있습니다, 무엇이 달라졌는지 보이시나요?
<!--Instead of doing `yield delay(1000)`, we're now doing `yield call(delay, 1000)`. What's the difference?-->

첫번째 경우에서, `delay(1000)` yield 구문은 `next` 의 호출자로 넘겨지기 전에 실행되고, (여기서 호출자는 미들웨어가 되거나, 제너레이터 함수를 실행하고 리턴된 제너레이터를 넘어 반복하는 테스트코드가 되어야 합니다.)  호출자가 얻게 되는것은 Promise 입니다. 아래 코드를 참고하세요.

<!--In the first case, the yield expression `delay(1000)` is evaluated before it gets passed to the caller of `next` (the caller could be the middleware when running our code. It could also be our test code which runs the Generator function and iterates over the returned Generator). So what the caller gets is a Promise, like in the test code above.-->

두번째 경우에선, `call(delay, 1000)` yield 구문은 `next` 의 호출자에게 넘겨지게 됩니다. `put` 과 유사한 `call` 은 미들웨어에게 주어진 함수와 인자들을 실행하라는 지시를 하는 이펙트를 리턴합니다.
사실, `put` 과 `call` 은 스스로 어떤 dispatch 나 비동기적인 호출을 하지 않습니다. 그들은 단지 순수한 자바스크립트 객체를 돌려줄 뿐입니다.

<!--In the second case, the yield expression `call(delay, 1000)` is what gets passed to the caller of `next`. `call` just like `put`, returns an Effect which instructs the middleware to call a given function with the given arguments. In fact, neither `put` nor `call` performs any dispatch or asynchronous call by themselves, they simply return plain JavaScript objects.-->

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

무슨일이 일어날까요? 미들웨어는 각각의 yield 된 이펙트들을 검사한뒤, 어떻게 이펙트를 수행할지 결정합니다. 만약 이펙트의 타입이 `PUT` 이라면, 미들웨어는 스토어에 액션을 dispatch 할것입니다. `CALL` 인 경우엔 주어진 함수를 실행하게 되는것 이고요.

<!--What happens is that the middleware examines the type of each yielded Effect then decides how to fulfill that Effect. If the Effect type is a `PUT` then it will dispatch an action to the Store. If the Effect is a `CALL` then it'll call the given function.-->

이 이펙트생성과 이펙트 실행의 분리는 제너레이터를 놀랍게도 쉽게 테스트가 가능하도록 만듭니다.
<!--This separation between Effect creation and Effect execution makes it possible to test our Generator in a surprisingly easy way:-->

```javascript
import test from 'tape';

import { put, call } from 'redux-saga/effects'
import { delay } from 'redux-saga'
import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  assert.deepEqual(
    gen.next().value,
    call(delay, 1000),
    'incrementAsync Saga must call delay(1000)'
  )

  assert.deepEqual(
    gen.next().value,
    put({type: 'INCREMENT'}),
    'incrementAsync Saga must dispatch an INCREMENT action'
  )

  assert.deepEqual(
    gen.next(),
    { done: true, value: undefined },
    'incrementAsync Saga must be done'
  )

  assert.end()
});
```

`put` 과 `call` 이 순수 객체를 반환하기 때문에, 테스트 코드에서 같은 함수들을 재사용할수 있게 되었고, `incrementAsync` 의 로직을 테스트 하기 위해 단순히 제너레이터를 반복하고 값에 대해 `deepEqual` 테스트를 할수 있게 되었습니다.

<!--Since `put` and `call` return plain objects, we can reuse the same functions in our test code. And to test the logic of `incrementAsync`, we simply iterate over the generator and do `deepEqual` tests on its values.-->

위의 테스트를 진행하기 위한 코드입니다:
<!--In order to run the above test, run:-->

```sh
$ npm test
```

이 테스트는 콘솔에 결과를 보고해야 합니다.
<!--which should report the results on the console.-->
