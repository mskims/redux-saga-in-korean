# 병렬 태스크 실행

`yield`문은 비동기 컨트롤 플로우를 간단하고 선형적으로 나타내기에 좋습니다. 하지만 우리는 병렬 처리가 필요한 경우도 있습니다. 우리는 다음을 단순화하기 어렵습니다:

```javascript
// wrong, effects will be executed in sequence
const users  = yield call(fetch, '/users'),
      repos = yield call(fetch, '/repos')
```

왜냐하면, 두 번째 이펙트는 첫번째 call이 resolve되기 전까지는 실행되지 않을 것이기 때문입니다. 대신에, 우리는 이렇게 써야 합니다:

```javascript
import { call } from 'redux-saga/effects'

// correct, effects will get executed in parallel
const [users, repos]  = yield [
  call(fetch, '/users'),
  call(fetch, '/repos')
]
```

위와 같이 이펙트의 배열을 yield하면, 제너레이터는 모든 이펙트들이 resolve되거나, 어느 하나라도 reject될 때까지 봉쇄(blocked)됩니다 (`Promise.all`의 방식처럼요).
