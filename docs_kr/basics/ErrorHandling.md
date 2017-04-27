# 에러 핸들링

이 섹션에선 이전 예제의 실패 케이스들을 어떻게 다룰지 볼겁니다. `Api.fetch` 함수가 어떤 이유때문에 fetch 가 실패했을때, reject 된 Promise 를 리턴한다고 가정합시다.
<!--In this section we'll see how to handle the failure case from the previous example. Let's suppose that our API function `Api.fetch` returns a Promise which gets rejected when the remote fetch fails for some reason.-->

우리의 Saga 안에서 `PRODUCTS_REQUEST_FAILED` 액션을 스토어에 dispatch 함으로써 이런 에러들을 다룰겁니다.
<!--We want to handle those errors inside our Saga by dispatching a `PRODUCTS_REQUEST_FAILED` action to the Store.-->

Saga 내에서 친숙한 `try/catch` 문법을 써서 에러들을 잡아낼 수 있습니다.
<!--We can catch errors inside the Saga using the familiar `try/catch` syntax.-->

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

// ...

function* fetchProducts() {
  try {
    const products = yield call(Api.fetch, '/products')
    yield put({ type: 'PRODUCTS_RECEIVED', products })
  }
  catch(error) {
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
  }
}
```

실패하는 경우도 테스트하기 위해서, 제너레이터의 `throw` 메소드를 사용할겁니다.
<!--In order to test the failure case, we'll use the `throw` method of the Generator-->

```javascript
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products'),
  "fetchProducts should yield an Effect call(Api.fetch, './products')"
)

// create a fake error
const error = {}

// expects a dispatch instruction
assert.deepEqual(
  iterator.throw(error).value,
  put({ type: 'PRODUCTS_REQUEST_FAILED', error }),
  "fetchProducts should yield an Effect put({ type: 'PRODUCTS_REQUEST_FAILED', error })"
)
```


이 경우에선, `throw` 메소드로 가짜 에러를 전달하고 있습니다. 이렇게 된다면 제너레이터는 현재 흐름을 중단하고 catch 블록을 실행할겁니다.
<!--In this case, we're passing the `throw` method a fake error. This will cause the Generator to break the current flow and execute the catch block.-->

무조건 API 에러들을 `try`/`catch` 블록 안에서 다뤄야 한다는건 아닙니다. 에러 상태를 리턴하는 평범한 값을 돌려주는 API 서비스를 만들수도 있습니다. 예를들면, Promise rejection 들을 catch 하여 에러필드를 가진 객체를 만들수도 있습니다.
<!--Of course, you're not forced to handle your API errors inside `try`/`catch` blocks. You can also make your API service return a normal value with some error flag on it. For example, you can catch Promise rejections and map them to an object with an error field.-->

```javascript
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

function fetchProductsApi() {
  return Api.fetch('/products')
    .then(response => ({ response }))
    .catch(error => ({ error }))
}

function* fetchProducts() {
  const { response, error } = yield call(fetchProductsApi)
  if (response)
    yield put({ type: 'PRODUCTS_RECEIVED', products: response })
  else
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
}
```
