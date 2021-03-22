# 스토어에 액션 dispatch 하기

Save 후 스토어에게 fetch 가 성공했다고 알려주는 액션을 dispatch 하려 한다고 가정해봅시다. (실패하는 케이스는 나중에 살펴보겠습니다.)

<!-- Taking the previous example further, let's say that after each save, we want to dispatch some action
to notify the Store that the fetch has succeeded (we'll omit the failure case for the moment). -->

스토어의 `dispatch` 함수를 제너레이터에게 넘기면, 제너레이터는 이 함수를 fetch 응답을 받은 후에 실행할 것입니다.

<!-- We could pass the Store's `dispatch` function to the Generator. Then the
Generator could invoke it after receiving the fetch response: -->

```javascript
// ...

function* fetchProducts(dispatch) {
  const products = yield call(Api.fetch, "/products")
  dispatch({ type: "PRODUCTS_RECEIVED", products })
}
```

그러나, 이 방법은 제너레이터 내부에서 함수를 직접적으로 호출하는 것과 비슷한 단점이 있습니다. (이전 섹션에서 말했던 것과 같습니다.) 만약 `fetchProducts` 가 AJAX 응답을 받은 후에 dispatch 를 수행한다는 것을 테스트하고 싶다면, 또다시 `dispatch` 함수를 흉내 내야 할것입니다.

<!-- However, this solution has the same drawbacks as invoking functions directly from inside the Generator (as discussed in the previous section). If we want to test that `fetchProducts` performs the dispatch after receiving the AJAX response, we'll need again to mock the `dispatch`
function. -->

흉내 내는 것 대신, 서술적 해결책이 필요합니다. 그저 미들웨어에게 어떤 액션을 dispatch 해야 하는지 지시하는 객체를 만들고, 실제 dispatch 는 미들웨어가 하도록 놔두세요. 이렇게만 한다면 yield 된 이펙트를 검사하고 정확한 명령이 포함되어있는지 확인하는 것만으로 제너레이터의 dispatch 를 테스트 할 수 있습니다.

<!-- Instead, we need the same declarative solution. Just create an Object to instruct the
middleware that we need to dispatch some action, and let the middleware perform the real
dispatch. This way we can test the Generator's dispatch in the same way: by just inspecting the yielded Effect and making sure it contains the correct instructions. -->

이 라이브러리는 이런 목적들 때문에 dispatch 이펙트를 생성하는 `put` 함수를 제공합니다.

<!-- The library provides, for this purpose, another function `put` which creates the dispatch
Effect. -->

```javascript
import { call, put } from "redux-saga/effects"
// ...

function* fetchProducts() {
  const products = yield call(Api.fetch, "/products")
  // dispatch 이펙트를 생성하고 yield 합니다.
  yield put({ type: "PRODUCTS_RECEIVED", products })
}
```

자 이제 이전 섹션에서 했던 것과 같이 제너레이터를 쉽게 테스트 할 수 있게 되었습니다.

<!-- Now, we can test the Generator easily as in the previous section -->

```javascript
import { call, put } from "redux-saga/effects"
import Api from "..."

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, "/products"),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// create a fake response
const products = {}

// expects a dispatch instruction
assert.deepEqual(
  iterator.next(products).value,
  put({ type: "PRODUCTS_RECEIVED", products }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_RECEIVED', products })"
)
```

`next` 메소드를 통해 제너레이터에게 어떻게 가짜 응답을 줬는지 기억하세요. 미들웨어 내부가 아니기 때문에 제너레이터를 완벽하게 제어할 수 있고, 이 때문에 결과를 흉내 내고 제너레이터를 재시작하는 것 만으로 실제 환경을 시뮬레이트 할 수 있습니다. 데이터(결과)를 흉내 내는 것은 함수나 call 들을 흉내내는것보다 훨씬 간단합니다.

<!-- Note how we pass the fake response to the Generator via its `next` method. Outside the
middleware environment, we have total control over the Generator, we can simulate a
real environment by simply mocking results and resuming the Generator with them. Mocking
data is a lot simpler than mocking functions and spying calls. -->
