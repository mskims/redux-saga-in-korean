# yield*로 사가 배열하기

`yield*` 연산자를 이용해서 여러 개의 사가들을 순서에 맞게 배열할 수 있습니다. 이는 당신의 *마이크로 태스크*들을 간단하고 절차적인 스타일로 배열할 수 있게 합니다.

```javascript
function* playLevelOne() { ... }

function* playLevelTwo() { ... }

function* playLevelThree() { ... }

function* game() {
  const score1 = yield* playLevelOne()
  yield put(showScore(score1))

  const score2 = yield* playLevelTwo()
  yield put(showScore(score2))

  const score3 = yield* playLevelThree()
  yield put(showScore(score3))
}
```

`yield*`가 자바스크립트 런타임을 골고루 *퍼지게* 한다는 것에 주목하세요. `game()`은 각 반복(LevelOne, LevelTwo, LevelThree)으로부터 모든 값들을 yield할 것입니다. 더 강력한 대안은 더 일반적인 미들웨어 구성 메커니즘을 사용하는 것입니다.
