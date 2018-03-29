# 서술적 이펙트

`redux-saga` 에서, Saga들은 제너레이터 함수들을 사용해서 구현되었습니다. Saga 로직을 표현하기 위해서 우리는 제너레이터로부터 온 순수 자바스크립트 객체를 yield 합니다. 이런 오브젝트들을 *이펙트* 라고 부릅니다. 이펙트는 미들웨어에 의해 해석되는 몇몇 정보들을 담고있는 간단한 객체입니다. 어떤 기능을 수행하기 위해 미들웨어에 전해지는 명령(스토어에 액션을 dispatch 하는 행위나 비동기 함수를 호출하는 등)이라고 볼수 있죠.

<!-- In `redux-saga`, Sagas are implemented using Generator functions. To express the Saga logic we yield plain JavaScript Objects from the Generator. We call those Objects *Effects*. An Effect is simply an object which contains some information to be interpreted by the middleware. You can view Effects like instructions to the middleware to perform some operation (invoke some asynchronous function, dispatch an action to the store). -->

이펙트들을 만들기 위해서, `redux-saga/effects` 패키지에 있는 라이브러리들이 제공하는 함수들을 사용합니다.
<!-- To create Effects, you use the functions provided by the library in the `redux-saga/effects` package. -->

이 섹션에서, 몇몇 기본 이펙트들을 소개하겠습니다. 어떻게 Saga 가 테스트되기 쉬운지 관찰해보세요.
<!-- In this section and the following, we will introduce some basic Effects. And see how the concept allows the Sagas to be easily tested. -->

Sagas 는 여러 형식으로 이펙트들을 yield 할 수 있습니다. 가장 쉬운 방법은 Promise 를 yield 하는것입니다.
<!-- Sagas can yield Effects in multiple forms. The simplest way is to yield a Promise. -->

예를 들어, `PRODUCTS_REQUESTED` 라는 액션을 보고있는 Saga 가 있다고 가정해봅시다, 액션이 매칭될때 마다, 서버로부터 온 상품 리스트를 가지고 오는 태스크를 실행합니다.
<!-- For example suppose we have a Saga that watches a `PRODUCTS_REQUESTED` action. On each matching action, it starts a task to fetch a list of products from a server. -->

```javascript
import { takeEvery } from 'redux-saga/effects'
import Api from './path/to/api'

function* watchFetchProducts() {
  yield takeEvery('PRODUCTS_REQUESTED', fetchProducts)
}

function* fetchProducts() {
  const products = yield Api.fetch('/products')
  console.log(products)
}
```

예제는 제너레이터 내부에서 `Api.fetch` 를 직접적으로 호출하고 있습니다. (제너레이터 함수에선, `yield` 의 오른편에 있는 모든 구문이 실행 되고, 그 결과는 호출자로 yield 됩니다.)
<!-- In the example above, we are invoking `Api.fetch` directly from inside the Generator (In Generator functions, any expression at the right of `yield` is evaluated then the result is yielded to the caller). -->

`Api.fetch('/products')` 는 AJAX 요청을 트리거 하고, resolve 된 응답을 resolve 할 Promise 를 리턴합니다. 이 AJAX 요청은 즉시, 간단하게 그리고 일반적으로 실행될겁니다. 그러나...
<!-- `Api.fetch('/products')` triggers an AJAX request and returns a Promise that will resolve with the resolved response, the AJAX request will be executed immediately. Simple and idiomatic, but... -->

위의 제너레이터를 테스트하고싶다고 가정해봅시다:
<!-- Suppose we want to test generator above: -->

```javascript
const iterator = fetchProducts()
assert.deepEqual(iterator.next().value, ??) // what do we expect ?
```

제너레이터에 의해 yield 된 첫번째 결과를 체크하고 싶습니다. 이 경우에선 `Api.fetch('/products')` 의 실행 결과인 Promise 가 됩니다. 하지만 테스트에 실제 서비스를 호출하는건 현실적이지도, 실용적이지도 못합니다. 그래서 실제 AJAX 요청을 실행하지 않고 진짜 함수대신 함수와 인자들을 올바르게 호출했다는 사실만 확인하는 가짜함수를 사용해서 `Api.fetch` 함수를 *흉내* 내봅시다. 
<!-- We want to check the result of the first value yielded by the generator. In our case it's the result of running `Api.fetch('/products')` which is a Promise . Executing the real service during tests is neither a viable nor practical approach, so we have to *mock* the `Api.fetch` function, i.e. we'll have to replace the real function with a fake one which doesn't actually run the AJAX request but only checks that we've called `Api.fetch` with the right arguments (`'/products'` in our case). -->

이런 가짜함수들은 테스팅을 더 어렵게 만들수도 있습니다, 하지만 다른 관점에서 보면, 결과를 체크하기 위해 `equal()` 을 사용할 수 있기 때문에 값들을 간단히 리턴하는 함수들은 테스트하기 더 쉽습니다. 이것이 가장 믿을만한 테스트를 작성하는 방법입니다.
<!-- Mocks make testing more difficult and less reliable. On the other hand, functions that simply return values are easier to test, since we can use a simple `equal()` to check the result. This is the way to write the most reliable tests. -->

아직 헷갈리십니까? [Eric Elliott의 글](https://medium.com/javascript-scene/what-every-unit-test-needs-f6cd34d9836d#.4ttnnzpgc)를 읽어보세요!:

> (...)`equal()`, 은 본질적으로 모든 단위테스트가 대답해야할 중요한 두가지 질문에 대답합니다.
하지만 대부분은 그러지 않는것이죠:
- 실제 출력값은 무엇입니까?
- 예상된 출력값은 무엇입니까?
>
> 만약 이 두가지 질문에 대답하지 않고 테스트를 마쳤다면, 진짜 단위 테스트가 아닌 반쪽짜리 날림 테스트가 될것입니다.

<!-- 
 (...)`equal()`, by nature answers the two most important questions every unit test must answer,
but most don’t:
- What is the actual output?
- What is the expected output?

 If you finish a test without answering those two questions, you don’t have a real unit test. You have a sloppy, half-baked test. -->

사실 우리는 그저 `fetchProducts` 태스크가 정상적인 함수와 인자를 가진 call 을 yield 하는지 확실하게 만들고 싶을 뿐입니다.
<!-- What we actually need is just to make sure the `fetchProducts` task yields a call with the right function and the right arguments. -->

제너레이터 안에서 직접적으로 비동기 함수를 호출하는 대신, **함수 호출에 관한 설명만 yield 할 수 있습니다.**. 이제 다음과 같이 생긴 오브젝트를 간단히 yield 할겁니다.
<!-- Instead of invoking the asynchronous function directly from inside the Generator, **we can yield only a description of the function invocation**. i.e. We'll simply yield an object which looks like -->

```javascript
// 이펙트 -> Api.fetch 함수를 './products' 인자와 함께 호출
{
  CALL: {
    fn: Api.fetch,
    args: ['./products']
  }
}
```

다른식으로 보자면, 제너레이터는 *명령* 을 담고 있는 순수한 객체를 yield 할 것이고, `redux-saga` 미들웨어는 이런 명령들의 실행을 처리하고, 결과를 제너레이터에 돌려줄 것입니다. 제너레이터를 테스트 할때 이 방법을 사용하면, yield 된 객체에 간단한 `deepEqual` 을 사용해 비교해서, 올바른 명령을 yield 하는지 확인하기만 하면 됩니다.
<!-- Put another way, the Generator will yield plain Objects containing *instructions*, and the `redux-saga` middleware will take care of executing those instructions and giving back the result of their execution to the Generator. This way, when testing the Generator, all we need to do is to check that it yields the expected instruction by doing a simple `deepEqual` on the yielded Object. -->

이러한 이유 때문에, 이 라이브러리는 비동기 요청을 수행할 다른 방법을 제공합니다.
<!-- For this reason, the library provides a different way to perform asynchronous calls. -->

```javascript
import { call } from 'redux-saga/effects'

function* fetchProducts() {
  const products = yield call(Api.fetch, '/products')
  // ...
}
```

이제 우린 `call(fn, ...args)` 함수를 쓸겁니다. **앞의 예제와 다른점은 이제 우린 더이상 fetch 요청을 즉시하지 않는다는것 입니다. 대신, `call` 은 이펙트에 대한 설명을 생성합니다.** Redux 에서와 마찬가지로, 스토어에 의해 실행될 액션을 설명하는 순수 객체를 만들기 위해 액션 생성자(action creator)들을 사용하고, `call` 은 함수 호출을 설명하는 순수 객체를 생성합니다. redux-saga 미들웨어는 함수 호출과 제너레이터를 resolve 된 응답과 함께 재가동 시킵니다.

<!-- We're using now the `call(fn, ...args)` function. **The difference from the preceding example is that now we're not executing the fetch call immediately, instead, `call` creates a description of the effect**. Just as in Redux you use action creators to create a plain object describing the action that will get executed by the Store, `call` creates a plain object describing the function call. The redux-saga middleware takes care of executing the function call and resuming the generator with the resolved response. -->

`call` 은 그저 순수 객체만 리턴하는 함수기 때문에 제너레이터를 Redux 환경 바깥에서 쉽게 테스트하게 만듭니다.
<!-- This allows us to easily test the Generator outside the Redux environment. Because `call` is just a function which returns a plain Object. -->

```javascript
import { call } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)
```

이제 아무것도 흉내낼 필요가 없어졌습니다. 간단한 비교 테스트로 충분할것입니다.
<!-- Now we don't need to mock anything, and a simple equality test will suffice. -->

이런 *서술적 호출을* 을 함으로써 Saga 내부에서 간단히 제너레이터를 반복하고, 연속적으로 yield 된 값들에 `deepEqual` 테스트를 하는것 만으로 모든 로직을 테스트 할 수 있습니다. 
This is a real benefit, as your complex asynchronous operations are no longer black boxes, and you can test in detail their operational logic no matter how complex it is. <!-- The advantage of those *declarative calls* is that we can test all the logic inside a Saga by simply iterating over the Generator and doing a `deepEqual` test on the values yielded successively. -->

`call` 은 또한 오브젝트 메소드 호출을 지원합니다. 다음과 같은 방식을 사용하여 호출된 함수에 `this` 컨텍스트를 사용할 수 있습니다.
<!-- `call` also supports invoking object methods, you can provide a `this` context to the invoked functions using the following form: -->

```javascript
yield call([obj, obj.method], arg1, arg2, ...) // as if we did obj.method(arg1, arg2 ...)
```

똑같은 기능을 하는 `apply` alias 함수도 있습니다.
<!-- `apply` is an alias for the method invocation form -->

```javascript
yield apply(obj, obj.method, [arg1, arg2, ...])
```

`call` 과 `apply` 는 Promise 들을 리턴하는 함수들에 적당합니다. 또다른 함수 `cps`(Continuation Passing Style) 는 노드 스타일의 함수들을 다루기 위해 쓰일수도 있습니다 (예: `fn(...args, callback)`, `callback` => `(error, result) => ()`). 
<!-- `call` and `apply` are well suited for functions that return Promise results. Another function `cps` can be used to handle Node style functions (e.g. `fn(...args, callback)` where `callback` is of the form `(error, result) => ()`). `cps` stands for Continuation Passing Style. -->

예:
<!-- For example: -->

```javascript
import { cps } from 'redux-saga/effects'

const content = yield cps(readFile, '/path/to/file')
```

그리고 당연히 `call` 로 테스트 했던것과 비슷하게 테스트 할 수 있습니다:
<!-- And of course you can test it just like you test `call`: -->

```javascript
import { cps } from 'redux-saga/effects'

const iterator = fetchSaga()
assert.deepEqual(iterator.next().value, cps(readFile, '/path/to/file') )
```

또, `cps` 는 `call` 과 똑같은 메소드 호출 형식을 지원합니다.
`cps` also supports the same method invocation form as `call`.
