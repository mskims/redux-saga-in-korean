<img src='https://raw.githubusercontent.com/mskims/redux-saga-in-korean/master/logo/0800/Redux-Saga-Logo-Landscape.png' alt='Redux Logo Landscape' width='800px'>

# redux-saga

[![Join the chat at https://gitter.im/yelouafi/redux-saga](https://badges.gitter.im/yelouafi/redux-saga.svg)](https://gitter.im/yelouafi/redux-saga?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![npm version](https://img.shields.io/npm/v/redux-saga.svg?style=flat-square)](https://www.npmjs.com/package/redux-saga) [![CDNJS](https://img.shields.io/cdnjs/v/redux-saga.svg?style=flat-square)](https://cdnjs.com/libraries/redux-saga)
[![OpenCollective](https://opencollective.com/redux-saga/backers/badge.svg)](#backers)
[![OpenCollective](https://opencollective.com/redux-saga/sponsors/badge.svg)](#sponsors)

### **NOTE**
이 문서는 현재 `redux-saga` [d0aa83d](https://github.com/redux-saga/redux-saga/tree/d0aa83d782b241799264879e139db4d03438ae28) 기준으로 작성되었습니다.

--

`redux-saga` 는 리액트/리덕스 애플리케이션의 사이드 이펙트, 예를 들면 데이터 fetching이나 브라우저 캐시에 접근하는 순수하지 않은 비동기 동작들을, 더 쉽고 좋게 만드는 것을 목적으로하는 라이브러리입니다.

saga는 애플리케이션에서 사이드 이펙트만을 담당하는 별도의 쓰레드와 같은 것으로 보면 됩니다. `redux-saga`는 리덕스 미들웨어입니다. 따라서 앞서 말한 쓰레드가 메인 애플리케이션에서 일반적인 리덕스 액션을 통해 실행되고, 멈추고, 취소될 수 있게 합니다. 또한 모든 리덕스 애플리케이션의 상태에 접근할 수 있고 리덕스 액션 또한 dispatch 할 수 있습니다.

이 라이브러리는 비동기 흐름을 쉽게 읽고, 쓰고, 테스트 할 수 있게 도와주는 ES6의 피쳐인 Generator를 사용합니다.  *(만약 Generator와 익숙하지 않다면 [여기 몇가지 소개 링크가 있습니다](https://mskims.github.io/redux-saga-in-korean/ExternalResources.html).)* Generator를 사용함으로써, 비동기 흐름은 표준 동기식 자바스크립트 코드처럼 보이게 됩니다. (`async`/`await`와 비슷한데, generator는 우리가 필요한 몇가지 훌륭한 피쳐들을 더 가지고 있습니다.)

당신은 데이터 fetching을 관리하기 위해 `redux-thunk`를 써본 적이 있을 수 있습니다. `redux-thunk`와는 대조적으로, 콜백 지옥에 빠지지 않으면서 비동기 흐름들을 쉽게 테스트할 수 있고 액션들을 순수하게 유지합니다.

# 시작하기

## 설치

```sh
// repo 생성
$ git clone https://github.com/redux-saga/redux-saga-beginner-tutorial.git

$ cd redux-saga-beginner-tutorial
```

```sh
$ npm install --save redux-saga
```
혹은

```sh
$ yarn add redux-saga
```

다른 방법으로, HTML 페이지의 `<script>` 태그에 직접 UMD 빌드를 사용할 수 있습니다. 이 [섹션](#using-umd-build-in-the-browser)을 확인하세요.

## 사용 예제

우리가 버튼을 눌렀을 때 원격 서버로부터 한 유저 데이터를 fetch하는 UI를 다룬다고 가정해봅시다. (간결함을 위해, 액션 실행 코드를 바로 보여드리겠습니다.)

```javascript
class UserComponent extends React.Component {
  ...
  onSomeButtonClicked() {
    const { userId, dispatch } = this.props
    dispatch({type: 'USER_FETCH_REQUESTED', payload: {userId}})
  }
  ...
}
```

컴포넌트는 플레인 오브젝트 액션을 스토어에 dispatch 합니다. 우리는 모든 `USER_FETCH_REQUESTED` 액션을 지켜보면서 유저 데이터를 fetch하는 API를 호출하도록 하는 Saga를 만들 것입니다.

#### `sagas.js`

```javascript
import { call, put, takeEvery, takeLatest } from 'redux-saga/effects'
import Api from '...'

// worker Saga: USER_FETCH_REQUESTED 액션에 대해 호출될 것입니다
function* fetchUser(action) {
   try {
      const user = yield call(Api.fetchUser, action.payload.userId);
      yield put({type: "USER_FETCH_SUCCEEDED", user: user});
   } catch (e) {
      yield put({type: "USER_FETCH_FAILED", message: e.message});
   }
}

/*
  각각의 dispatch 된 `USER_FETCH_REQUESTED` 액션에 대해 fetchUser를 실행합니다.
  동시에 user를 fetch하는 것을 허용합니다.
*/
function* mySaga() {
  yield takeEvery("USER_FETCH_REQUESTED", fetchUser);
}

/*
  또는 takeLatest를 사용할 수 있습니다.
  
  동시에 user를 fetch하는 것을 허용하지 않습니다. 만약 fetch가 이미 대기 상태일 때  "USER_FETCH_REQUESTED"가 dispatch가 되었다면 대기 상태의 fetch는 취소되고 항상 최근 것만이 실행됩니다.
*/
function* mySaga() {
  yield takeLatest("USER_FETCH_REQUESTED", fetchUser);
}

export default mySaga;
```

Saga를 실행하기 위해서 `redux-saga` 미들웨어를 리덕스 스토어에 연결해야 합니다.

#### `main.js`

```javascript
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

import reducer from './reducers'
import mySaga from './sagas'

// saga 미들웨어를 생성합니다.
const sagaMiddleware = createSagaMiddleware()
// 스토어에 mount 합니다.
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)

// 그리고 saga를 실행합니다.
sagaMiddleware.run(mySaga)

// 애플리케이션을 render합니다.
```

# Documentation

- [시작하면서](https://mskims.github.io/redux-saga-in-korean/introduction/BeginnerTutorial.html)
- [기본 개념](https://mskims.github.io/redux-saga-in-korean/basics/index.html)
- [고급 개념](https://mskims.github.io/redux-saga-in-korean/advanced/index.html)
- [레시피](https://mskims.github.io/redux-saga-in-korean/recipes/index.html)
- [외부 자료](https://mskims.github.io/redux-saga-in-korean/ExternalResources.html)
- [문제해결](https://mskims.github.io/redux-saga-in-korean/Troubleshooting.html)
- [용어 사전](https://mskims.github.io/redux-saga-in-korean/Glossary.html)
- [API 레퍼런스](https://mskims.github.io/redux-saga-in-korean/api/index.html)

# 번역

- [중국어 간체](https://github.com/superRaytin/redux-saga-in-chinese)
- [중국어 번체](https://github.com/neighborhood999/redux-saga)
- [일본어](https://github.com/redux-saga/redux-saga/blob/master/README_ja.md)
- [한국어](https://github.com/redux-saga/redux-saga/blob/master/README_ko.md)
- [러시아어](https://github.com/redux-saga/redux-saga/blob/master/README_ru.md)

# 브라우저에서 umd 빌드 사용하기

`dist/` 폴더 하위에 `redux-saga`의 **umd** 빌드 버전을 제공합니다. umd 빌드 버전의 `redux-saga`를 사용하면 window 오브젝트의 `ReduxSaga`를 사용할 수 있습니다.

이러한 umd 버전은 Webpack 혹은 Browserify를 사용하지 않을 때 매우 편리합니다. [unpkg](https://unpkg.com/)에서 직접 사용할 수도 있습니다.

다음과 같은 빌드 버전을 사용할 수 있습니다:

- [https://unpkg.com/redux-saga/dist/redux-saga.js](https://unpkg.com/redux-saga/dist/redux-saga.js)
- [https://unpkg.com/redux-saga/dist/redux-saga.min.js](https://unpkg.com/redux-saga/dist/redux-saga.min.js)

**중요!** 만약 당신의 타겟 브라우져가 **ES2015 genertor**를 지원하지 않는다면 [`babel`에서 제공하는 것](https://cdnjs.cloudflare.com/ajax/libs/babel-core/5.8.25/browser-polyfill.min.js) 같은 polyfill을 이용해야 합니다. 이 때 **redux-saga** 보다 먼저 import 되야합니다.

```javascript
import 'babel-polyfill'
// 그 다음에
import sagaMiddleware from 'redux-saga'
```

# 소스를 빌드하는 예제

```sh
$ git clone https://github.com/redux-saga/redux-saga.git
$ cd redux-saga
$ npm install
$ npm test
```

아래는 리덕스 저장소로부터 포팅한 (포팅중인) 예제들입니다.

### 카운터 예제

다음의 세가지 카운터 예제가 존재합니다.

#### 기본 카운터

UMD 빌드 버전과 기본 자바스크립트를 사용한 예제입니디. 모든 소스는 `index.html`에 작성되어 있습니다.

예제를 실행하기 위해선 그냥 `index.html`를 브라우저에 실행시키면 됩니다.

> 중요: 브라우저가 Generator를 지원해야합니다. 최신 Chrome/Firefox/Edge 브라우저 버전은 지원됩니다.

#### 카운터

`webpack`과 high-level API인 `takeEvery`를 사용하는 예제입니다.

```sh
$ npm run counter

# generator를 위한 테스트 예제
$ npm run test-counter
```

#### 취소할 수 있는 카운터

low-level API를 이용해 동작을 취소하는 예제입니다.

```sh
$ npm run cancellable-counter
```

### 쇼핑 카트 예제

```sh
$ npm run shop

# generator를 위한 테스트 예제
$ npm run test-shop
```

### 비동기 예제

```sh
$ npm run async

# generator를 위한 테스트 예제
$ npm run test-async
```

### real-world 예제 (webpack hot reloading을 이용함)

```sh
$ npm run real-world

# 아직 작업 중 입니다
```

### 로고

다양한 종류의 공식 Redux-Saga 로고를 [로고 디렉토리](logo)에서 찾을 수 있습니다.


### Backers
매달 기부함으로써 저희의 오픈 소스 활동을 서포트 해주세요. \[[후원자 되기](https://opencollective.com/redux-saga#backer)\]

<a href="https://opencollective.com/redux-saga/backer/0/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/0/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/1/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/1/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/2/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/2/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/3/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/3/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/4/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/4/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/5/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/5/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/6/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/6/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/7/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/7/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/8/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/8/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/9/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/9/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/10/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/10/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/11/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/11/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/12/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/12/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/13/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/13/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/14/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/14/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/15/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/15/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/16/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/16/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/17/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/17/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/18/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/18/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/19/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/19/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/20/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/20/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/21/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/21/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/22/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/22/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/23/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/23/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/24/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/24/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/25/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/25/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/26/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/26/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/27/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/27/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/28/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/28/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/backer/29/website" target="_blank"><img src="https://opencollective.com/redux-saga/backer/29/avatar.svg"></a>


### 스폰서
스폰서가 되어 당신의 로고와 링크를 Github README에 노출하세요. \[[스폰서 되기](https://opencollective.com/redux-saga#sponsor)\]

<a href="https://opencollective.com/redux-saga/sponsor/0/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/0/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/1/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/1/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/2/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/2/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/3/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/3/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/4/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/4/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/5/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/5/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/6/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/6/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/7/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/7/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/8/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/8/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/9/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/9/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/10/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/10/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/11/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/11/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/12/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/12/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/13/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/13/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/14/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/14/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/15/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/15/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/16/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/16/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/17/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/17/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/18/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/18/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/19/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/19/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/20/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/20/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/21/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/21/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/22/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/22/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/23/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/23/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/24/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/24/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/25/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/25/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/26/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/26/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/27/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/27/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/28/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/28/avatar.svg"></a>
<a href="https://opencollective.com/redux-saga/sponsor/29/website" target="_blank"><img src="https://opencollective.com/redux-saga/sponsor/29/avatar.svg"></a>
