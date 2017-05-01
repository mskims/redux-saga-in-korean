# 사가와 외부 입/출력 연결

`take` 이펙트는 스토어에 dispatch될 액션이 들어오면 resolve 되었습니다. 그리고 `put` 이펙트는 액션을 인자로 dispatch함으로써 resolve되었습니다.

사가가 시작될 때 미들웨어는 자동으로 `take`/`put`을 스토어와 연결합니다. 이 두 이펙트는 사가의 입력/출력처럼 보일 수 있겠죠.

`redux-saga`는 리덕스 미들웨어 환경 바깥에서 사가를 실행하고 커스텀 입/출력에 연결할 수 있는 방법을 제공합니다.

```javascript
import { runSaga } from 'redux-saga'

function* saga() { ... }

const myIO = {
  subscribe: ..., // this will be used to resolve take Effects
  dispatch: ...,  // this will be used to resolve put Effects
  getState: ...,  // this will be used to resolve select Effects
}

runSaga(
  saga(),
  myIO
)
```

자세한 정보는 [API 문서](https://redux-saga.js.org/docs/api/index.html#runsagaiterator-options)를 참조하세요.
