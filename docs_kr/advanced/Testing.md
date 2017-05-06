# 사가 테스트

**이펙트는 일반 자바스크립트 객체를 반환합니다.**

이 객체들은 이펙트를 표현 또는 설명하고, redux-saga는 이를 실행합니다.

이렇게 하면 테스트가 아주 쉬워집니다. 왜냐하면 사가에 의해 yield된 객체가 우리가 원하는 이펙트를 설명하는지 비교만 하면 되니까요.
 
## 기본 예제

```javascript
console.log(put({ type: MY_CRAZY_ACTION }));

/*
{
  @@redux-saga/IO': true,
  PUT: {
    channel: null,
    action: {
      type: 'MY_CRAZY_ACTION'
    }
  }
}
 */
```

사용자의 액션을 기다리고 dispatch하는 사가를 테스트해봅시다.

```javascript
const CHOOSE_COLOR = 'CHOOSE_COLOR';
const CHANGE_UI = 'CHANGE_UI';

const chooseColor = (color) => ({
  type: CHOOSE_COLOR,
  payload: {
    color,
  },
});

const changeUI = (color) => ({
  type: CHANGE_UI,
  payload: {
    color,
  },
});


function* changeColorSaga() {
  const action = yield take(CHOOSE_COLOR);
  yield put(changeUI(action.payload.color));
}

test('change color saga', assert => {
  const gen = changeColorSaga();

  assert.deepEqual(
    gen.next().value,
    take(CHOOSE_COLOR),
    'it should wait for a user to choose a color'
  );

  const color = 'red';
  assert.deepEqual(
    gen.next(chooseColor(color)).value,
    put(changeUI(color)),
    'it should dispatch an action to change the ui'
  );

  assert.deepEqual(
    gen.next().done,
    true,
    'it should be done'
  );

  assert.end();
});
```

테스트는 또한 문서처럼 사용되므로 더 좋습니다! 일어날만한 모든 일을 설명하니까요.

## 사가 분기

가끔씩 사가는 다른 결과를 가질 때가 있습니다. 사가의 모든 단계를 다시 반복하지 않고 분기하려면 **cloneableGenerator** 유틸리티 함수를 사용하세요.

```javascript
const CHOOSE_NUMBER = 'CHOOSE_NUMBER';
const CHANGE_UI = 'CHANGE_UI';
const DO_STUFF = 'DO_STUFF';

const chooseNumber = (number) => ({
  type: CHOOSE_NUMBER,
  payload: {
    number,
  },
});

const changeUI = (color) => ({
  type: CHANGE_UI,
  payload: {
    color,
  },
});

const doStuff = () => ({
  type: DO_STUFF, 
});


function* doStuffThenChangeColor() {
  yield put(doStuff());
  yield put(doStuff());
  const action = yield take(CHOOSE_NUMBER);
  if (action.payload.number % 2 === 0) {
    yield put(changeUI('red'));
  } else {
    yield put(changeUI('blue'));
  }
}

import { put, take } from 'redux-saga/effects';
import { cloneableGenerator } from 'redux-saga/utils';

test('doStuffThenChangeColor', assert => {
  const data = {};
  data.gen = cloneableGenerator(doStuffThenChangeColor)();

  assert.deepEqual(
    data.gen.next().value,
    put(doStuff()),
    'it should do stuff'
  );

  assert.deepEqual(
    data.gen.next().value,
    put(doStuff()),
    'it should do stuff'
  );

  assert.deepEqual(
    data.gen.next().value,
    take(CHOOSE_NUMBER),
    'should wait for the user to give a number'
  );

  assert.test('user choose an even number', a => {
    // cloning the generator before sending data
    data.clone = data.gen.clone();
    a.deepEqual(
      data.gen.next(chooseNumber(2)).value,
      put(changeUI('red')),
      'should change the color to red'
    );

    a.equal(
      data.gen.next().done,
      true,
      'it should be done'
    );

    a.end();
  });

  assert.test('user choose an odd number', a => {
    a.deepEqual(
      data.clone.next(chooseNumber(3)).value,
      put(changeUI('blue')),
      'should change the color to blue'
    );

    a.equal(
      data.clone.next().done,
      true,
      'it should be done'
    );

    a.end();
  });
});
```

fork 이펙트 테스트에 관해서는 [태스크 취소](TaskCancellation.md)를 참조하세요.

저장소 예제들입니다.

<https://github.com/redux-saga/redux-saga/blob/master/examples/counter/test/sagas.js>

<https://github.com/redux-saga/redux-saga/blob/master/examples/shopping-cart/test/sagas.js>
