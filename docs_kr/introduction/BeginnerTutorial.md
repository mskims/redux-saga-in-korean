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

<!--> In case you encountered an issue with running the application. Feel free to create an issue on the [tutorial repo](https://github.com/redux-saga/redux-saga-beginner-tutorial/issues).-->

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

이제, 오리지널 카운터 데모에 가까운 무언가를 추가해봅시다. 비동기 호출을 설명하기 위해 클릭 1초 후 증가되는 또다른 버튼을 추가할겁니다.
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

Now we will introduce another Saga to perform the asynchronous call. Our use case is as follows:

> On each `INCREMENT_ASYNC` action, we want to start a task that will do the following

> - Wait 1 second then increment the counter

Add the following code to the `sagas.js` module:

```javascript
import { delay } from 'redux-saga'
import { put, takeEvery } from 'redux-saga/effects'

// Our worker Saga: will perform the async increment task
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// Our watcher Saga: spawn a new incrementAsync task on each INCREMENT_ASYNC
export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

Time for some explanations.

We import `delay`, a utility function that returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will resolve after a specified number of milliseconds. We'll use this function to *block* the Generator.

Sagas are implemented as [Generator functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) that *yield* objects to the redux-saga middleware. The yielded objects are a kind of instruction to be interpreted by the middleware. When a Promise is yielded to the middleware, the middleware will suspend the Saga until the Promise completes. In the above example, the `incrementAsync` Saga is suspended until the Promise returned by `delay` resolves, which will happen after 1 second.

Once the Promise is resolved, the middleware will resume the Saga, executing code until the next yield. In this example, the next statement is another yielded object: the result of calling `put({type: 'INCREMENT'})`, which instructs the middleware to dispatch an `INCREMENT` action.

`put` is one example of what we call an *Effect*. Effects are simple JavaScript objects which contain instructions to be fulfilled by the middleware. When a middleware retrieves an Effect yielded by a Saga, the Saga is paused until the Effect is fulfilled.

So to summarize, the `incrementAsync` Saga sleeps for 1 second via the call to `delay(1000)`, then dispatches an `INCREMENT` action.

Next, we created another Saga `watchIncrementAsync`. We use `takeEvery`, a helper function provided by `redux-saga`, to listen for dispatched `INCREMENT_ASYNC` actions and run `incrementAsync` each time.

Now we have 2 Sagas, and we need to start them both at once. To do that, we'll add a `rootSaga` that is responsible for starting our other Sagas. In the same file `sagas.js`, add the following code:

```javascript
// single entry point to start all Sagas at once
export default function* rootSaga() {
  yield [
    incrementAsync(),
    watchIncrementAsync()
  ]
}
```

This Saga yields an array with the results of calling our two sagas, `helloSaga` and `watchIncrementAsync`. This means the two resulting Generators will be started in parallel. Now we only have to invoke `sagaMiddleware.run` on the root Saga in `main.js`.

```javascript
// ...
import rootSaga from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = ...
sagaMiddleware.run(rootSaga)

// ...
```

## Making our code testable

We want to test our `incrementAsync` Saga to make sure it performs the desired task.

Create another file `sagas.spec.js`:

```javascript
import test from 'tape';

import { incrementAsync } from './sagas'

test('incrementAsync Saga test', (assert) => {
  const gen = incrementAsync()

  // now what ?
});
```

`incrementAsync` is a generator function. When run, it returns an iterator object, and the iterator's `next` method returns an object with the following shape

```javascript
gen.next() // => { done: boolean, value: any }
```

The `value` field contains the yielded expression, i.e. the result of the expression after
the `yield`. The `done` field indicates if the generator has terminated or if there are still
more 'yield' expressions.

In the case of `incrementAsync`, the generator yields 2 values consecutively:

1. `yield delay(1000)`
2. `yield put({type: 'INCREMENT'})`

So if we invoke the next method of the generator 3 times consecutively we get the following
results:

```javascript
gen.next() // => { done: false, value: <result of calling delay(1000)> }
gen.next() // => { done: false, value: <result of calling put({type: 'INCREMENT'})> }
gen.next() // => { done: true, value: undefined }
```

The first 2 invocations return the results of the yield expressions. On the 3rd invocation
since there is no more yield the `done` field is set to true. And since the `incrementAsync`
Generator doesn't return anything (no `return` statement), the `value` field is set to
`undefined`.

So now, in order to test the logic inside `incrementAsync`, we'll simply have to iterate
over the returned Generator and check the values yielded by the generator.

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

The issue is how do we test the return value of `delay`? We can't do a simple equality test
on Promises. If `delay` returned a *normal* value, things would've been easier to test.

Well, `redux-saga` provides a way to make the above statement possible. Instead of calling
`delay(1000)` directly inside `incrementAsync`, we'll call it *indirectly*:

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

Instead of doing `yield delay(1000)`, we're now doing `yield call(delay, 1000)`. What's the difference?

In the first case, the yield expression `delay(1000)` is evaluated before it gets passed to the caller of `next` (the caller could be the middleware when running our code. It could also be our test code which runs the Generator function and iterates over the returned Generator). So what the caller gets is a Promise, like in the test code above.

In the second case, the yield expression `call(delay, 1000)` is what gets passed to the caller of `next`. `call` just like `put`, returns an Effect which instructs the middleware to call a given function with the given arguments. In fact, neither `put` nor `call` performs any dispatch or asynchronous call by themselves, they simply return plain JavaScript objects.

```javascript
put({type: 'INCREMENT'}) // => { PUT: {type: 'INCREMENT'} }
call(delay, 1000)        // => { CALL: {fn: delay, args: [1000]}}
```

What happens is that the middleware examines the type of each yielded Effect then decides how to fulfill that Effect. If the Effect type is a `PUT` then it will dispatch an action to the Store. If the Effect is a `CALL` then it'll call the given function.

This separation between Effect creation and Effect execution makes it possible to test our Generator in a surprisingly easy way:

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

Since `put` and `call` return plain objects, we can reuse the same functions in our test code. And to test the logic of `incrementAsync`, we simply iterate over the generator and do `deepEqual` tests on its values.

In order to run the above test, run:

```sh
$ npm test
```

which should report the results on the console.
